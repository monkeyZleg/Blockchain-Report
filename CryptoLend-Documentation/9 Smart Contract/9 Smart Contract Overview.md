
## 9. Smart Contract Overview

This section walks `CryptoLoan.sol` end to end: its architecture, risk parameters, and every function, each core action paired with the real **frontend bridge** code that calls it from `web/src/lib/WalletContext.tsx`, the same pairing Section 7 in the full document already walked from the UI's point of view. It closes with the supporting token contracts.

### Table of contents, Section 9

- [9.1 Contract Architecture](#91-contract-architecture)
- [9.2 Risk Parameters & Constants](#92-risk-parameters--constants)
- [9.3 State Variables](#93-state-variables)
- [9.4 Events](#94-events)
- [9.5 Core Function Walkthroughs](#95-core-function-walkthroughs)
- [9.6 Admin & Oracle Functions](#96-admin--oracle-functions)
- [9.7 View Functions, Health Factor & Interest Model](#97-view-functions-health-factor--interest-model)
- [9.8 Liquidation](#98-liquidation)
- [9.9 Supporting Contracts](#99-supporting-contracts)

---

### 9.1 Contract Architecture

`CryptoLoan` is a single monolithic protocol contract. There is no separate vault/oracle/token split. It inherits three OpenZeppelin building blocks:

```solidity
contract CryptoLoan is ReentrancyGuard, Pausable, Ownable2Step {
    MockMYR public myr;
    ...
}
```

| Inherited from    | Gives the contract                                                                                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ReentrancyGuard` | `nonReentrant` on every state-changing function that moves ETH or MYR                                                                                                       |
| `Pausable`        | `whenNotPaused` guards + admin-only `pause()`/`unpause()`, an emergency stop                                                                                                |
| `Ownable2Step`    | The admin role, plus a two-step transfer of it (safer than one-shot `Ownable`, since a typo'd `transferOwnership` address can't permanently brick control) |

**The central design decision is that each borrow is its own loan.** The contract's own header states it directly:

> Each borrow is its OWN loan: its principal, its APR (locked at the moment of borrowing), its start date and its due date. Collateral stays one shared pot per borrower — every loan is backed by the same ETH, so LTV and the health factor are computed against the SUM of active principals, while interest and repayment are strictly per-loan.

That split runs through the whole contract, and it's what makes "repay plan #2" mean exactly plan #2's principal plus plan #2's interest and nothing else:

| Concern                                    | Scope                                                            |
| ------------------------------------------ | ---------------------------------------------------------------- |
| Collateral (`collateralOf`)                | **One shared pot** per borrower                                  |
| LTV, health factor, borrow limit           | Against the sum of active principals                         |
| Principal, APR, start/due date, interest   | **Per loan** (`_userLoans[borrower][loanId]`)                    |
| Repayment                                  | **Per loan**, addressed by `loanId`                              |

Loans are also **fixed-term** (30/90/180/365 days). Past the due date a 7-day grace period runs; after that the loan is liquidatable even if the collateral itself is perfectly healthy (§9.8).

Its constructor deploys its own `MockMYR` token instance (`myr = new MockMYR()`), so the loan contract is the token's `minter` by construction, with no separate wiring step and no way to end up with a `CryptoLoan` pointed at the wrong MYR token (see [6.5](#65-what-deployts-actually-does) for how this shapes the deploy script's fresh-node requirement).

On the frontend, every call into the contract goes through one factory in `WalletContext.tsx`, which picks a read-only provider or a MetaMask-signed runner depending on whether the call changes state:

```ts
const getContracts = useCallback(async (signed: boolean) => {
  if (LOAN_ADDR === ZERO_ADDR) return null;
  const provider = getProvider();
  if (!provider) return null;
  const runner = signed ? await provider.getSigner() : provider;
  return {
    loan: new ethers.Contract(CONTRACT_ADDRESSES.CryptoLoan, CRYPTO_LOAN_ABI, runner),
    myr:  new ethers.Contract(CONTRACT_ADDRESSES.MockMYR,    MOCK_MYR_ABI,    runner),
  };
}, [getProvider]);
```

`CONTRACT_ADDRESSES` and both ABIs come straight from the generated `contractConfig.ts` ([6.5](#65-what-deployts-actually-does)). Nothing about the contract's address or interface is hand-maintained on the frontend.

### 9.2 Risk Parameters & Constants

```solidity
uint256 public constant MAX_LTV          = 70;   // 70% max loan-to-value
uint256 public constant LIQ_THRESHOLD    = 80;   // 80% liquidation trigger
uint256 public constant LIQ_BONUS        = 5;    // 5% bonus for liquidators
uint256 public constant MIN_HEALTH       = 1e18; // HF below this = liquidatable
uint256 public constant PRECISION        = 1e18;
uint256 public constant MYR_DECIMALS     = 1e6;
uint256 public constant MAX_PRICE_CHANGE = 20;   // max 20% price move per update
uint256 public constant GRACE_PERIOD     = 7 days;
uint256 public constant ACCRUAL_STEP     = 1 days;
uint256 public constant MYR_TZ_OFFSET    = 8 hours;
```

| Constant           | Value                  | Meaning                                                                                                                      |
| ------------------ | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `MAX_LTV`          | 70%                    | The ceiling `borrow()` enforces against collateral value (matches the "70% Max LTV" reference card in Section 7.5 Portfolio) |
| `LIQ_THRESHOLD`    | 80%                    | The collateral-value multiplier used in the health-factor formula (§9.7)                                                     |
| `LIQ_BONUS`        | 5%                     | Extra collateral a liquidator seizes on top of the debt they cover (§9.8)                                                    |
| `MIN_HEALTH`       | `1e18` (i.e. HF = 1.0) | Below this, the account's collateral is unsafe and any of its loans is liquidatable                                          |
| `PRECISION`        | `1e18`                 | Fixed-point scale for the health factor                                                                                      |
| `MYR_DECIMALS`     | `1e6`                  | MYR token precision (matches `MockMYR.decimals()`)                                                                           |
| `MAX_PRICE_CHANGE` | 20%                    | Max allowed move per `setEthPrice()` call (a circuit breaker against a fat-fingered or manipulated oracle update)            |
| `GRACE_PERIOD`     | 7 days                 | Extra repayment time after a loan's due date before *overdue* liquidation becomes possible. Not a free extension: interest keeps accruing through it, it is only a shield against being liquidated the second the term ends |
| `ACCRUAL_STEP`     | 1 day                  | Interest ticks once per calendar day, so the figure the UI quotes matches the figure the contract charges (§9.7)          |
| `MYR_TZ_OFFSET`    | 8 hours                | Day boundaries land on **Malaysia midnight (UTC+8)**, not UTC midnight. This is an MYR product, so "today" means local time  |

**The rate model.** The borrow rate is a base rate plus a premium that rises with pool utilization:

```solidity
uint256 public baseRateBps               = 300;   // owner-adjustable market rate (initial 3.0%)
uint256 public constant UTIL_SLOPE_BPS    = 400;  // up to +4.0% as lending capacity fills
uint256 public constant VOL_SLOPE_BPS     = 300;  // defined but NOT applied — see below
uint256 public constant MAX_BASE_RATE_BPS = 1500; // hard cap: 15%

uint256 public supplyCap = 100_000_000 * MYR_DECIMALS;  // RM 100,000,000
```

`supplyCap` gives "the pool" a real size. MYR is minted on demand, so without a ceiling there would be nothing for utilisation to be a fraction *of*. It is the denominator of `utilizationBps()`, which drives the rate premium, and the limit `borrow()` refuses to exceed. It is admin-settable (§9.6), which is why it can't be `constant` or `immutable`.

`VOL_SLOPE_BPS` is defined but never applied. The contract's comment gives the reason: it would move the rate on a price update, which a borrower can neither see coming nor reproduce from a single view call. It is an unused constant kept for reference, not dead code left by accident.

### 9.3 State Variables

| Variable          | Type                         | Purpose                                                                                          |
| ----------------- | ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `ethPrice`        | `uint256`                    | MYR per ETH, whole number (e.g. `18000`), the on-chain price oracle                              |
| `prevEthPrice`    | `uint256`                    | Price before the last update (volatility input, currently unused)                                |
| `lastPriceTime`   | `uint256`                    | Timestamp of the last `setEthPrice()` call                                                       |
| `protocolFees`    | `uint256`                    | Accumulated interest revenue (MYR units), payable out via `withdrawProtocolFees()`               |
| `totalBorrowed`   | `uint256`                    | Protocol-wide outstanding debt (MYR units); numerator of `utilizationBps()`                      |
| `totalCollateral` | `uint256`                    | Protocol-wide collateral (wei)                                                                   |
| `supplyCap`       | `uint256`                    | Lending-pool ceiling (MYR units), admin-settable                                                 |
| `baseRateBps`     | `uint256`                    | Admin-adjustable market base rate, before the utilisation premium                                |
| `kycApproved`     | `mapping(address ⇒ bool)`    | Per-wallet gate for `borrow()`, set only by the admin via `setKYC()` (§9.6)                      |
| `liquidators`     | `mapping(address ⇒ bool)`    | Whitelisted addresses allowed to call `liquidate()`                                              |
| `supplyStart`     | `mapping(address ⇒ uint256)` | Timestamp a depositor's supply-interest accrual clock started; `0` means "not currently earning" |
| `collateralOf`    | `mapping(address ⇒ uint256)` | **The shared collateral pot** (wei) backing all of that borrower's loans together                |
| `_userLoans`      | `mapping(address ⇒ Loan[])`  | **The loan book.** Private; read through `getUserLoans()`. Array index = `loanId`                |

```solidity
struct Loan {
    uint256 principal;     // MYR units (6 decimals) still owed
    uint256 startTime;     // unix timestamp of this borrow
    uint256 dueDate;       // startTime + term
    uint256 lastRepayTime; // interest clock for THIS loan (resets on its repays)
    uint256 termDays;      // 30 / 90 / 180 / 365
    uint256 aprBps;        // APR locked at borrow time — fixed for the loan's life
    uint256 baseBps;       // baseRateBps at the same moment — display split only
    bool    active;        // false once fully repaid or fully liquidated
}
```

Two details of this struct are easy to misread:

- **`loanId` is the array index, and it is stable for the life of the account.** Repaid and liquidated loans are flagged `active = false` and left in place (never spliced out), so an id can never silently come to mean a different loan. This is what the `BorrowPosition.loanId` join key in [8.4](#84-table-by-table-walkthrough) depends on.
- **`baseBps` is presentation-only.** It records what `baseRateBps` was at the same moment `aprBps` was locked, purely so the UI can show `aprBps − baseBps` as the utilisation premium. Interest math never reads it; `aprBps` is the rate that is actually charged.

### 9.4 Events

| Event                                                                                    | Emitted by                                                     | Carries                                                        |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------- |
| `CollateralDeposited(user, amount)`                                                      | `depositCollateral()`                                          | wei deposited                                                  |
| `Borrowed(user, myrAmount, newTotal)`                                                    | `borrow()`                                                     | MYR minted, borrower's new total active principal              |
| `LoanCreated(borrower, loanId, principal, dueDate, termDays, aprBps, baseBps)`           | `borrow()`                                                     | The full birth certificate of the new loan, including the locked rates |
| `Repaid(user, loanId, principal, interest)`                                              | `repay()` / `repayMany()`                                      | Which loan was paid, and the principal/interest split          |
| `LoanClosed(user, loanId)`                                                               | `_repayOne()`, `liquidate()`                                   | Loan reached zero principal and went inactive                  |
| `CollateralWithdrawn(user, amount)`                                                      | `withdrawCollateral()`                                         | wei returned                                                   |
| `Liquidated(user, liquidator, debtCovered, collateralSeized)`                            | `liquidate()`                                                  | Liquidation outcome in aggregate                               |
| `LoanLiquidated(borrower, loanId, collateralAmount, reason)`                             | `liquidate()`                                                  | Which loan, and why (`"collateral unsafe"` / `"loan overdue"`) |
| `PriceUpdated(oldPrice, newPrice, updatedBy)`                                            | `setEthPrice()`                                                | oracle update, with who pushed it                              |
| `BaseRateUpdated(oldRateBps, newRateBps)`                                                | `setBaseRate()`                                                | market-rate change                                             |
| `SupplyCapUpdated(oldCap, newCap)`                                                       | `setSupplyCap()`                                               | pool resize                                                    |
| `KYCSet(user, approved)`                                                                 | `setKYC()`                                                     | entitlement flip                                               |
| `LiquidatorSet(liquidator, approved)`                                                    | `setLiquidator()`                                              | whitelist change                                               |
| `ProtocolFeesWithdrawn(to, amount)`                                                      | `withdrawProtocolFees()`                                       | treasury payout (Section 7.12)                                 |
| `MYRPurchased(buyer, ethSpent, myrReceived)`                                             | `buyMYR()`                                                     | ETH→MYR swap                                                   |
| `SupplyInterestClaimed(user, amount)`                                                    | `claimSupplyInterest()`, and implicitly `withdrawCollateral()` | depositor interest payout                                      |

The frontend depends on `LoanCreated` and the per-loan `Repaid` rather than just logging them: they carry the `loanId` and locked rates the off-chain ledger keys off (§9.5.2, §9.5.3). The rest are what `LoanTransaction` rows mirror ([8.4](#84-table-by-table-walkthrough)) and what the public Explorer (Section 7.6) filters by type.

### 9.5 Core Function Walkthroughs

Every state-changing user action pairs one `CryptoLoan` function with one callback in `WalletContext.tsx`. They share the same shape: build the transaction, `await tx.wait()`, mirror the receipt into Postgres via `saveTxToDB`, then call `refresh()` to re-read the wallet's on-chain position.

#### 9.5.1 Deposit Collateral

```solidity
function depositCollateral() external payable whenNotPaused nonReentrant {
    require(msg.value > 0, "Send ETH");
    if (supplyStart[msg.sender] == 0) {
        supplyStart[msg.sender] = block.timestamp;
    }
    collateralOf[msg.sender] += msg.value;
    totalCollateral          += msg.value;
    emit CollateralDeposited(msg.sender, msg.value);
}
```

Any ETH sent with the call joins the borrower's shared pot and, if this is their first deposit, starts the supply-interest accrual clock (§9.7) in the same transaction. Note there is **no `onlyKYC`** here: depositing is not gated, only borrowing is.

```ts
const depositCollateral = useCallback(async (ethAmt: string) => {
  if (!guardTx()) return;
  const c = await getContracts(true);
  if (!c || !s.address) return;
  setTx('pending', `Depositing ${ethAmt} ETH…`);
  try {
    const tx = await c.loan.depositCollateral({ value: ethers.parseEther(ethAmt) });
    const receipt = await tx.wait();
    if (receipt && s.address) saveTxToDB(s.address, 'CollateralDeposited', ethers.parseEther(ethAmt).toString(), receipt);
    setTx('success', `Deposited ${ethAmt} ETH as collateral`);
    await refresh(s.address);
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Deposit failed: ${reason}` : 'Deposit failed');
  }
}, [getContracts, s.address, s.ethPriceMYR, refresh, guardTx]);
```

`guardTx()` runs first on every action (not just this one). It refuses the call unless the connected MetaMask account *is* the account's own linked wallet, surfacing a precise message ("wrong account selected", "no wallet linked") instead of letting a mismatched signer silently fail on-chain. **UI**: the Deposit tab (Section 7.7), with quick-fill buttons and a live "max you could borrow at 70% LTV" preview computed from the same `_maxBorrow` math the contract uses (§9.7).

#### 9.5.2 Borrow MYR (opens a new fixed-term loan)

```solidity
function borrow(uint256 myrAmount, uint256 termDays) external whenNotPaused nonReentrant onlyKYC {
    require(myrAmount > 0, "Amount must be > 0");
    require(
        termDays == 30 || termDays == 90 || termDays == 180 || termDays == 365,
        "Invalid term"
    );
    require(collateralOf[msg.sender] > 0, "Deposit collateral first");

    uint256 principalNow = _activePrincipal(msg.sender);
    uint256 maxBorrow    = _maxBorrow(collateralOf[msg.sender]);
    require(principalNow + myrAmount <= maxBorrow, "Exceeds max LTV");
    require(totalBorrowed + myrAmount <= supplyCap, "Pool cap reached");

    uint256 loanId = _userLoans[msg.sender].length;
    uint256 due    = block.timestamp + termDays * 1 days;
    _userLoans[msg.sender].push(Loan({
        principal:     myrAmount,
        startTime:     block.timestamp,
        dueDate:       due,
        lastRepayTime: block.timestamp,
        termDays:      termDays,
        aprBps:        currentAprBps(),   // locked for this loan's whole life
        baseBps:       baseRateBps,
        active:        true
    }));
    totalBorrowed += myrAmount;

    myr.mint(msg.sender, myrAmount);
    emit Borrowed(msg.sender, myrAmount, principalNow + myrAmount);
    emit LoanCreated(msg.sender, loanId, myrAmount, due, termDays, _userLoans[msg.sender][loanId].aprBps, baseRateBps);
}
```

The order of the three guards matters. The per-user LTV check runs before the pool-cap check, so a borrower normally sees `"Exceeds max LTV"`, which they can act on, instead of a protocol-wide condition they cannot.

The rate is captured before `totalBorrowed` grows on the next line. That ordering is why `LoanCreated` carries the rates: the loan locks the utilisation the pool had when the borrower committed, not the utilisation their own borrow creates. A `currentAprBps()` read taken *after* the transaction therefore comes back **higher** than what the loan is actually charged, which is precisely the bug the bridge below avoids:

```ts
const TERM_DAYS: Record<number, number> = { 1: 30, 3: 90, 6: 180, 12: 365 };
const termDays = TERM_DAYS[opts?.termMonths ?? 1] ?? 30;
const tx = await c.loan.borrow(units, BigInt(termDays));
const receipt = await tx.wait();

// LoanCreated carries the on-chain loanId, dueDate AND the rates the contract
// actually locked. Quoting a fresh currentAprBps() read instead is what once
// showed 3.40% on the receipt for a loan locked at 3.36%.
const createdEvt = ((receipt?.logs ?? []) as ethers.Log[])
  .map(l => { try { return c.loan.interface.parseLog(l); } catch { return null; } })
  .find(p => p?.name === 'LoanCreated');
if (createdEvt) {
  loanId        = Number(createdEvt.args[1] as bigint);
  dueDateIso    = new Date(Number(createdEvt.args[3] as bigint) * 1000).toISOString();
  lockedAprBps  = Number(createdEvt.args[5] as bigint);
  lockedBaseBps = Number(createdEvt.args[6] as bigint);
}
```

Those decoded values are then written into the `BorrowPosition` ledger row ([8.4](#84-table-by-table-walkthrough)), which is what ties the DB row to its on-chain loan:

```ts
await fetch('/api/borrows', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    wallet: addr, principal: units.toString(), aprBps, baseAprBps,
    termMonths: opts?.termMonths ?? 1, txHash: receipt.hash,
    loanId, dueDate: dueDateIso,
  }),
});
```

The `onlyKYC` modifier reverts before touching any state if `kycApproved[msg.sender]` is `false`, and the bridge builds in a **self-healing retry** for exactly that revert reason, typically after a local redeploy wiped the on-chain flag while the database still says approved:

```ts
try {
  await attemptBorrow();
  return true;
} catch (e) {
  const reason = revertReason(e);
  if (reason === 'KYC required') {
    setTx('pending', 'Re-syncing KYC…');
    await resyncKyc(addr);
    try { await attemptBorrow(); return true; }
    catch (e2) { /* surface the second failure */ return false; }
  }
  setTx('error', reason ? `Borrow failed: ${reason}` : 'Borrow failed — check LTV or collateral');
  return false;
}
```

**UI**: the Borrow tab (Section 7.9), with the 1/3/6/12-month term picker that maps to `termDays`, the base-plus-premium rate breakdown, and a receipt quoting the locked APR and due date (plus its 7-day grace period).

#### 9.5.3 Repay Loan (per loan, or several at once)

Repayment is addressed to a **specific loan**, and both entry points funnel into one internal routine:

```solidity
function repay(uint256 loanId, uint256 myrAmount) external whenNotPaused nonReentrant {
    require(myrAmount > 0, "Amount must be > 0");
    uint256 paying = _repayOne(msg.sender, loanId, myrAmount);
    require(paying > 0, "Nothing to repay");
    myr.transferFrom(msg.sender, address(this), paying);
}

/// @notice Repay several loans in one transaction — one ERC-20 approval,
///         one signature. `amounts[i]` is the cap applied to `loanIds[i]`.
function repayMany(uint256[] calldata loanIds, uint256[] calldata amounts) external whenNotPaused nonReentrant {
    require(loanIds.length > 0 && loanIds.length == amounts.length, "Bad input");
    uint256 total = 0;
    for (uint256 i = 0; i < loanIds.length; i++) {
        total += _repayOne(msg.sender, loanIds[i], amounts[i]);
    }
    require(total > 0, "Nothing to repay");
    myr.transferFrom(msg.sender, address(this), total);
}
```

`_repayOne()` applies interest first, then principal, but only *this* loan's interest and *this* loan's principal:

```solidity
function _repayOne(address user, uint256 loanId, uint256 myrAmount) internal returns (uint256) {
    require(loanId < _userLoans[user].length, "No such loan");
    Loan storage loan = _userLoans[user][loanId];
    if (!loan.active || myrAmount == 0) return 0;

    uint256 interest = _loanInterest(loan);
    uint256 due      = loan.principal + interest;
    uint256 paying   = myrAmount > due ? due : myrAmount;   // never overcharge

    uint256 interestPaid  = paying >= interest ? interest : paying;
    uint256 principalPaid = paying > interest ? paying - interest : 0;

    protocolFees  += interestPaid;
    totalBorrowed -= principalPaid;

    if (principalPaid >= loan.principal) {
        loan.principal = 0; loan.active = false; loan.lastRepayTime = block.timestamp;
        emit Repaid(user, loanId, principalPaid, interestPaid);
        emit LoanClosed(user, loanId);
    } else {
        loan.principal -= principalPaid; loan.lastRepayTime = block.timestamp;
        emit Repaid(user, loanId, principalPaid, interestPaid);
    }
    return paying;
}
```

Two properties fall out of this that matter for the UI. Each loan is capped at **its own** total due, so over-quoting a full payoff can never overcharge; and inactive loans are silently skipped rather than reverting the whole batch.

There is no `onlyKYC` modifier. A user must always be able to pay off debt, even if their verification lapsed.

The bridge takes a list of `{ loanId, amount, rowId }` items and picks the single- or multi-loan entry point automatically:

```ts
const repayTx = loanIds.length === 1
  ? await c.loan.repay(loanIds[0], amounts[0])
  : await c.loan.repayMany(loanIds, amounts);
```

For a full settlement it re-quotes each loan's real payoff from the chain and then pads it by one accrual day, because the repay transaction mines seconds later and may cross a Malaysia-midnight boundary, ticking interest up a step after the quote was taken. Since the contract caps each loan at its true due, over-quoting the *approval* costs nothing, while under-quoting is exactly what leaves sen of principal behind:

```ts
const dues = await Promise.all(items.map(i =>
  c.loan.loanDue(s.address!, BigInt(i.loanId))));
const STEP = BigInt(86_400);       // mirrors ACCRUAL_STEP (1 day)
const YEAR = BigInt(31_536_000);   // mirrors Solidity's `365 days`
amounts = items.map((it, idx) => {
  const chain  = book.find(l => l.loanId === it.loanId);
  const oneDay = chain ? (chain.principal * BigInt(chain.aprBps) * STEP) / (BigInt(10_000) * YEAR) : BigInt(0);
  const padded = dues[idx] + oneDay;
  return padded + padded / BigInt(500) + BigInt(1_000_000); // +0.2% + RM 1 headroom
});
```

Afterwards the per-loan `Repaid` events are decoded into a map, so each ledger row is settled with the principal its own loan reported and the DB cannot drift from the chain:

```ts
const paidByLoan = new Map<number, { principal: bigint; interest: bigint }>();
for (const l of repayReceipt?.logs ?? []) {
  let parsed = null;
  try { parsed = c.loan.interface.parseLog(l); } catch { /* other contract's log */ }
  if (parsed?.name === 'Repaid') {
    paidByLoan.set(Number(parsed.args[1] as bigint), {
      principal: parsed.args[2] as bigint,
      interest:  parsed.args[3] as bigint,
    });
  }
}
```

Consistent with the contract's own comment, collateral is **never auto-returned** by this flow; it stays deposited so the same ETH can back a new borrow without redepositing. **UI**: the Repay tab (Section 7.10) lists every open plan as a selectable row and totals the bill, with an inline Buy MYR top-up if the wallet's balance is short.

#### 9.5.4 Withdraw Collateral

```solidity
function withdrawCollateral(uint256 weiAmount) external whenNotPaused nonReentrant {
    require(collateralOf[msg.sender] >= weiAmount, "Not enough collateral");

    uint256 remaining    = collateralOf[msg.sender] - weiAmount;
    uint256 principalNow = _activePrincipal(msg.sender);
    if (principalNow > 0) {
        uint256 maxBorrow = _maxBorrow(remaining);
        require(principalNow <= maxBorrow, "Would violate LTV");
    }

    // A full withdrawal ends supply accrual — pay out any unclaimed supply
    // interest NOW, before accruedSupplyInterest() starts returning 0.
    uint256 payout = remaining == 0 ? accruedSupplyInterest(msg.sender) : 0;

    collateralOf[msg.sender] -= weiAmount;
    totalCollateral          -= weiAmount;
    if (collateralOf[msg.sender] == 0) supplyStart[msg.sender] = 0;

    if (payout > 0) {
        myr.mint(msg.sender, payout);
        emit SupplyInterestClaimed(msg.sender, payout);
    }
    (bool ok, ) = payable(msg.sender).call{value: weiAmount}("");
    require(ok, "ETH transfer failed");
    emit CollateralWithdrawn(msg.sender, weiAmount);
}
```

Because collateral is one shared pot, the LTV guard runs against `_activePrincipal()`, the sum of every still-active loan, not any single loan. The bridge decodes the (possible) `SupplyInterestClaimed` log emitted alongside `CollateralWithdrawn` so a full exit reports both amounts in one line, rather than looking like the interest was forfeited:

```ts
const claimEvt = (receipt?.logs ?? [])
  .map(l => { try { return c.loan.interface.parseLog(l); } catch { return null; } })
  .find(p => p?.name === 'SupplyInterestClaimed');
if (claimEvt) interestClaimed = Number(claimEvt.args[1] as bigint) / 1e6;

setTx('success', interestClaimed > 0
  ? `Withdrawn ${ethAmt} ETH + RM ${interestClaimed.toFixed(4)} interest claimed`
  : `Withdrawn ${ethAmt} ETH`);
```

**UI**: the Withdraw tab (Section 7.8) computes the maximum safely-withdrawable amount client-side before submit, mirroring the contract's own LTV guard.

#### 9.5.5 Buy MYR

```solidity
function buyMYR(uint256 myrAmount) external payable whenNotPaused nonReentrant {
    require(myrAmount > 0, "Amount must be > 0");
    require(ethPrice > 0, "Price not set");
    uint256 ethNeeded = (myrAmount * 1e18) / (ethPrice * MYR_DECIMALS);
    require(msg.value >= ethNeeded, "Insufficient ETH");
    myr.mint(msg.sender, myrAmount);
    uint256 excess = msg.value - ethNeeded;
    if (excess > 0) {
        payable(msg.sender).transfer(excess);
    }
    emit MYRPurchased(msg.sender, ethNeeded, myrAmount);
}
```

A straight mint-for-ETH swap that never touches the caller's collateral or loans. The bridge pads the ETH it sends by 0.1% to absorb rounding, trusting the contract's own excess-refund to return whatever wasn't needed:

```ts
const units     = BigInt(Math.floor(parseFloat(myrAmt) * 1e6));
const ethPrice  = BigInt(s.ethPriceMYR);
const ethNeeded = (units * BigInt(1e18)) / (ethPrice * BigInt(1e6));
const ethWithBuffer = ethNeeded + ethNeeded / BigInt(1000); // +0.1% buffer for rounding
const tx = await c.loan.buyMYR(units, { value: ethWithBuffer });
```

**UI**: the Buy MYR tab (Section 7.11), most often opened as a top-up shortcut from inside Repay when the wallet's MYR balance falls short.

#### 9.5.6 Claim Supply Interest

```solidity
function claimSupplyInterest() external whenNotPaused nonReentrant {
    uint256 interest = accruedSupplyInterest(msg.sender);
    require(interest > 0, "Nothing to claim");
    supplyStart[msg.sender] = block.timestamp;
    myr.mint(msg.sender, interest);
    emit SupplyInterestClaimed(msg.sender, interest);
}
```

Lets a depositor claim their share of borrow-side interest (38% of the current effective borrow rate, §9.7) without touching their collateral or any loan. The accrual clock resets to `now`.

```ts
const claimSupplyInterest = useCallback(async () => {
  if (!guardTx()) return;
  const c = await getContracts(true);
  if (!c || !s.address) return;
  setTx('pending', 'Claiming supply interest…');
  try {
    const tx = await c.loan.claimSupplyInterest();
    const receipt = await tx.wait();
    setTx('success', 'Supply interest claimed');
    if (receipt && s.address) saveTxToDB(s.address, 'SupplyInterestClaimed', String(Math.round(s.pendingYieldMYR * 1e6)), receipt);
    await refresh(s.address);
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Claim failed: ${reason}` : 'Claim failed');
  }
}, [getContracts, s.address, s.pendingYieldMYR, refresh, guardTx]);
```

**UI**: the **Claim MYR** button on Portfolio's Supply Earnings card (Section 7.5).

### 9.6 Admin & Oracle Functions

Every privileged function on the contract is gated by one role. In Solidity that role is the `Ownable2Step` owner and the modifier is spelled `onlyOwner`, but in this system it is the **admin**: the deploying address, whose key the server holds as `OWNER_PRIVATE_KEY`, and which the admin panel in Section 7.12 acts through. There is no second privileged role and no separate "owner" person. Wherever this document says admin, the code says `onlyOwner`.

| Function                      | Guard       | Effect                                                                                                  |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------------------------------- |
| `setKYC(user, approved)`      | `onlyOwner` | Grants/revokes on-chain borrowing permission for a wallet                                               |
| `setLiquidator(addr, ok)`     | `onlyOwner` | Adds/removes a whitelisted liquidator                                                                   |
| `setEthPrice(price)`          | `onlyOwner` | Updates the oracle, capped at a ±20% move (`MAX_PRICE_CHANGE`)                                          |
| `setBaseRate(rateBps)`        | `onlyOwner` | Adjusts the market base rate, capped at `MAX_BASE_RATE_BPS` (15%)                                       |
| `setSupplyCap(newCap)`        | `onlyOwner` | Resizes the lending pool; never below `totalBorrowed`                                                   |
| `pause()` / `unpause()`       | `onlyOwner` | Emergency stop / resume for every `whenNotPaused` function                                              |
| `withdrawProtocolFees(to)`    | `onlyOwner` | Pays accumulated interest revenue out in MYR (the Overview page's treasury button, Section 7.12)        |

Two of these are called automatically by the server, not by an admin clicking a contract call.

**KYC approval**, via `web/src/lib/kyc/chain.ts`, wraps `setKYC()` so the user never pays gas or signs anything to get approved:

```solidity
function setKYC(address user, bool approved) external onlyOwner {
    require(user != address(0), "Zero address");
    kycApproved[user] = approved;
    emit KYCSet(user, approved);
}
```

```ts
export async function setKycOnChain(wallet: string, approved = true): Promise<void> {
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const signer   = new ethers.Wallet(process.env.OWNER_PRIVATE_KEY!, provider);
  const contract = new ethers.Contract(LOAN_ADDR, LOAN_ABI, signer);
  const tx = await contract.setKYC(wallet, approved);
  await tx.wait();
}
```

This is called from two places: `POST /api/kyc/approve` when an admin approves a submission (Section 7.12), and `POST /api/kyc/self-resync`, which any signed-in user can trigger themselves to *restore* (never grant) their own already-approved flag after it's lost, typically because a local Hardhat restart wiped contract state while the database still says `approved`:

```ts
const entitled = user.isAdmin || user.kyc?.status === 'approved';
if (!entitled) return NextResponse.json({ error: 'KYC not approved.' }, { status: 403 });
await setKycOnChain(user.walletAddress.toLowerCase(), true);
```

**Price & rate oracle**, via `web/src/lib/price-sync.ts`, drives `setEthPrice()` and `setBaseRate()`:

```solidity
function setEthPrice(uint256 _price) external onlyOwner {
    require(_price > 0, "Invalid price");
    uint256 old = ethPrice;
    if (old > 0) {
        uint256 diff = _price > old ? _price - old : old - _price;
        require(diff * 100 / old <= MAX_PRICE_CHANGE, "Price move too large");
    }
    prevEthPrice  = old;
    ethPrice      = _price;
    lastPriceTime = block.timestamp;
    emit PriceUpdated(old, _price, msg.sender);
}
```

Because the contract caps a single call to a ±20% move, closing a large gap to the live CoinGecko price takes a walk of several transactions, each re-reading the current on-chain price so concurrent callers (an auto-trigger and a manual admin click) converge instead of racing each other's nonce:

```ts
for (let guard = 0; guard < 40; guard++) {
  const current = Number(await contract.ethPrice());
  if (current === target) break;
  const diff = Math.abs(target - current) / current;
  const next = diff <= MAX_STEP
    ? target
    : target < current ? Math.round(current * 0.81) : Math.round(current * 1.19); // 19% steps
  const tx = await contract.setEthPrice(BigInt(next));
  await tx.wait();
}
```

The same run also derives a market-driven `baseRateBps` from ETH's 24h change (`300 + clamp(change%, -10, +15) * 20` bps) and pushes it via `setBaseRate()` if it differs from what's on-chain. `POST /api/admin/sync-price` (the admin panel's manual button) and the dashboard's own auto-trigger both call this same deduplicated function, so only one price-walk is ever in flight at a time:

```ts
let inFlight: Promise<SyncResult> | null = null;
export function syncEthPriceDeduped(): Promise<SyncResult> {
  if (!inFlight) inFlight = syncEthPriceToMarket().finally(() => { inFlight = null; });
  return inFlight;
}
```

`setSupplyCap()` guards against a state the protocol cannot recover from:

```solidity
function setSupplyCap(uint256 newCap) external onlyOwner {
    require(newCap >= totalBorrowed, "Cap below outstanding debt");
    ...
}
```

A cap under the outstanding debt would push `utilizationBps()` to 100%, instantly pinning every new borrow at `MAX_BASE_RATE_BPS` (while existing loans keep their locked rate), with no way to lend out of it.

### 9.7 View Functions, Health Factor & Interest Model

| Function                             | Returns                                                                                                         |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `currentAprBps()`                    | Live APR for a new borrow: `baseRateBps + utilPremiumBps()`, clamped to `MAX_BASE_RATE_BPS`                  |
| `utilizationBps()`                   | Share of the pool lent out (0–10,000), `totalBorrowed / supplyCap`                                              |
| `utilPremiumBps()`                   | The utilisation premium alone, exposed so the UI can show it as its own line without re-implementing the formula |
| `availableToBorrowPool()`            | MYR still lendable before the pool cap is reached                                                               |
| `accruedInterestForLoan(user, id)`   | Interest on **one** loan, at that loan's locked APR                                                             |
| `loanDue(user, id)`                  | One loan's principal + accrued interest (the payoff quote the bridge re-reads, §9.5.3)                           |
| `accruedInterest(user)`              | Aggregate interest across every active loan, each at its own locked rate                                        |
| `totalDue(user)`                     | `_activePrincipal(user) + accruedInterest(user)`                                                                |
| `healthFactor(user)`                 | See below                                                                                                       |
| `availableToBorrow(user)`            | Remaining borrowable MYR at `MAX_LTV`, against the shared pot                                                   |
| `currentLTV(user)`                   | The account's loan-to-value, as a whole-number percentage                                                       |
| `isLoanLiquidatable(user, id)`       | `(liquidatable, unhealthy, overdue)`: whether, and why (§9.8)                                               |
| `getUserLoans(user)`                 | The whole loan book plus each loan's live interest; inactive loans included so ids stay stable                  |
| `loanCount(user)`                    | Length of the loan array                                                                                        |
| `getPosition(user)`                  | Minimal `(collateral, principal)` pair, for server-side guards like the wallet-unlink check                     |
| `getLoanInfo(user)`                  | One-call account bundle read by `WalletContext.tsx`'s `refresh()` to populate every live card in Section 7.3    |
| `getPoolStats()`                     | Cap, borrowed, available, utilisation, premium and current APR in one call, for the pool card and rate breakdown |
| `supplyInterestRate()`               | Supply APR in bps: 38% of the **current effective** borrow rate                                                 |
| `accruedSupplyInterest(user)`        | MYR accrued since `supplyStart[user]`                                                                           |
| `getProtocolStats()`                 | Protocol-wide totals, treasury figures for Section 7.12 Admin Overview                                          |

`getPoolStats()` exists as a separate view rather than as extra return values on `getProtocolStats()` for a specific reason the contract documents: several callers decode that older view into a hand-written five-tuple, which would silently misdescribe itself if its arity changed.

**Borrow rate.** The premium is a straight linear function of utilisation, capped:

```solidity
function utilPremiumBps() public view returns (uint256) {
    return (UTIL_SLOPE_BPS * utilizationBps()) / 10_000;
}

function currentAprBps() public view returns (uint256) {
    uint256 rate = baseRateBps + utilPremiumBps();
    return rate > MAX_BASE_RATE_BPS ? MAX_BASE_RATE_BPS : rate;
}
```

This prices new borrows only. Each loan locks the figure into `loan.aprBps` at creation and accrues at that fixed rate for its whole life. The Repay tab's per-plan interest, the payoff quote and the ledger all read the *locked* rate, never this one.

**Health factor**, in `_healthFactor()`, runs against the aggregate:

```solidity
function _healthFactor(address user) internal view returns (uint256) {
    uint256 principal = _activePrincipal(user);
    if (principal == 0) return type(uint256).max;   // no debt = infinitely healthy
    uint256 collateralMYR = (collateralOf[user] * ethPrice * MYR_DECIMALS) / PRECISION;
    return (collateralMYR * LIQ_THRESHOLD * PRECISION) / (100 * principal);
}
```

In plain terms: `HF = (collateral value in MYR × 80%) / total active debt`, scaled to 18-decimal fixed point (`PRECISION = 1e18`, so `MIN_HEALTH = 1e18` reads as `HF = 1.0`). Because collateral is one shared pot and the denominator is the sum of active principals, the health factor is a property of the **account**, not of any single loan. `refresh()` reads it straight off `getLoanInfo()` and converts it once for the whole UI:

```ts
const MAX_U = BigInt('0xff...ff'); // type(uint256).max
const hfRaw = info[2] as bigint;
const hf = hfRaw === MAX_U ? Infinity : Number(hfRaw) / 1e18;
```

Both the plain-English risk sentence on the Dashboard ("At Risk: liquidates if ETH falls to or below RM X") and the Portfolio's escalating risk banner (Section 7.5) are derived from this number.

**Interest accrual** ticks once per calendar day, anchored to Malaysia midnight, so the figure the UI quotes is the figure the contract charges. A per-second rate would already have moved by the time the user signs:

```solidity
function _loanInterest(Loan storage loan) internal view returns (uint256) {
    if (!loan.active || loan.principal == 0) return 0;
    uint256 lastDay = (loan.lastRepayTime + MYR_TZ_OFFSET) / ACCRUAL_STEP;
    uint256 curDay  = (block.timestamp     + MYR_TZ_OFFSET) / ACCRUAL_STEP;
    uint256 dayDiff = curDay - lastDay;
    bool firstAccrual = loan.lastRepayTime == loan.startTime;
    uint256 elapsed = (dayDiff == 0 && firstAccrual ? 1 : dayDiff) * ACCRUAL_STEP;
    return (loan.principal * loan.aprBps * elapsed) / (10_000 * 365 days);
}
```

The rate used is `loan.aprBps`, not `currentAprBps()`, so each loan accrues at the rate it was born with. The `firstAccrual` guard handles one specific case. The minimum-one-day floor must apply before a loan's first repayment, since a same-day borrow-then-repay still owes that day, but it must not apply to a same-day *repeat* repay. Since `lastRepayTime` resets on every repayment, without that guard two partial payments minutes apart would each be charged a full phantom day of interest, silently eating every partial payment.

Supply-side interest uses the same whole-day flooring but on a plain rolling window (UTC, not local-midnight anchored), at 38% of the current effective rate:

```solidity
function supplyInterestRate() public view returns (uint256) {
    return (currentAprBps() * 38) / 100;
}
```

Tying supply yield to `currentAprBps()` rather than the flat base rate means lender yield tracks real market demand instead of a fixed floor.

### 9.8 Liquidation

`liquidate(borrower, loanId, debtAmount)` is whitelisted-liquidator-only and targets one loan, accepting it for either of two independent reasons:

```solidity
bool unhealthy = _healthFactor(borrower) < MIN_HEALTH;
bool overdue   = block.timestamp > loan.dueDate + GRACE_PERIOD;
require(unhealthy || overdue, "Not liquidatable");
```

| Path                     | Trigger                                                        | Notes                                                                      |
| ------------------------ | -------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **A. Collateral unsafe** | Account health factor < 1 (ETH fell, or debt outgrew collateral) | Applies to *any* active loan of that account, at any time                  |
| **B. Loan overdue**      | This loan's `dueDate + GRACE_PERIOD` has passed, still unpaid    | Applies even if the collateral is perfectly healthy; this is the maturity path fixed-term lending needs |

The liquidator repays up to `debtAmount` of that loan's debt and receives the equivalent collateral value plus the 5% `LIQ_BONUS`. Only enough collateral to cover what was actually repaid is seized; the remainder stays in the borrower's pot, withdrawable once their remaining debt allows. If the pot can't cover `covering + bonus`, the contract scales the MYR pulled from the liquidator *down* to match what's actually seizable, so a liquidator is never charged for collateral they don't receive:

```solidity
if (seize > collateralOf[borrower]) {
    seize = collateralOf[borrower];
    collateralValue = (seize * 100) / (100 + LIQ_BONUS);
    covering        = (collateralValue * ethPrice * MYR_DECIMALS) / PRECISION;
}
```

`isLoanLiquidatable()` exposes the same two booleans as a view, so the dashboard can label a loan not just as at-risk but with *which* reason applies. There is still no liquidator-facing UI in the app; the overdue path can be demonstrated on a local chain by advancing the Hardhat clock.

### 9.9 Supporting Contracts

**`MockMYR.sol`**, the 6-decimal ERC-20 that `CryptoLoan` mints on every `borrow()`, `buyMYR()`, and interest payout. Minting is restricted to a single `minter` address, fixed at construction time to whichever contract deployed it:

```solidity
contract MockMYR is ERC20 {
    address public minter;
    constructor() ERC20("Mock Malaysian Ringgit", "MYR") { minter = msg.sender; }
    function decimals() public pure override returns (uint8) { return 6; }
    function mint(address to, uint256 amount) external {
        require(msg.sender == minter, "Not authorized");
        _mint(to, amount);
    }
}
```

Since `CryptoLoan`'s constructor is the one calling `new MockMYR()`, `minter` is always `CryptoLoan`'s own address. There is no separate `setMinter()` step, and no path for any other contract or EOA to mint MYR. `MockUSDC.sol` is an unused sibling with an identical shape (also 6-decimal, same minter pattern); it is not wired into `CryptoLoan` or referenced by the frontend.

**`ICO.sol` / `RinggitToken.sol`**, a separate demo behind the `/ico` page, unrelated to the lending protocol proper. `RinggitToken` is a standard 18-decimal ERC-20 (`MYRC`) that mints its entire fixed supply to the deployer at construction. `ICO` sells that supply for ETH at a fixed price set at deploy time, using `safeTransferFrom(owner() → buyer)` against an allowance the deployer grants it. The ICO contract never custodies the token supply itself, only an approval to move it:

```solidity
function buyToken() external payable whenNotPaused {
    uint256 numberOfToken = (msg.value * 1e18) / price;
    token.safeTransferFrom(owner(), msg.sender, numberOfToken);
    totalTokensSold += numberOfToken;
    emit TokensPurchased(msg.sender, msg.value, numberOfToken);
}
```

It shares the `Pausable` / `Ownable` / `ReentrancyGuard` pattern with `CryptoLoan`, and uses `call` rather than `transfer` for its `withdrawal()` function specifically so an owner that's a smart-contract wallet (not a plain EOA) isn't at risk of reverting on `transfer`'s fixed 2300-gas stipend.
