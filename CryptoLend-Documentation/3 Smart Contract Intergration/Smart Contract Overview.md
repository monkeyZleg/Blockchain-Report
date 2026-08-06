
## 9. Smart Contract Overview

This section walks [`CryptoLoan.sol`](../../../blockchain/contracts/CryptoLoan.sol) end to end: its architecture, risk parameters, and every function, each core action paired with the real **frontend bridge** code that calls it from [`web/src/lib/WalletContext.tsx`](../../../web/src/lib/WalletContext.tsx), the same pairing Section 7 in the full document already walked from the UI's point of view. It closes with the supporting token contracts.

### Table of contents, Section 9

- [9.1 Contract Architecture](#91-contract-architecture)
- [9.2 Risk Parameters & Constants](#92-risk-parameters--constants)
- [9.3 State Variables](#93-state-variables)
- [9.4 Events](#94-events)
- [9.5 Core Function Walkthroughs](#95-core-function-walkthroughs)
- [9.6 Admin & Oracle Functions](#96-admin--oracle-functions)
- [9.7 View Functions, Health Factor & Interest Model](#97-view-functions-health-factor--interest-model)
- [9.8 Supporting Contracts](#98-supporting-contracts)

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
| `Pausable`        | `whenNotPaused` guards + owner-only `pause()`/`unpause()`, an emergency stop                                                                                                |
| `Ownable2Step`    | Owner-gated admin functions, with a two-step ownership transfer (safer than one-shot `Ownable`, since a typo'd `transferOwnership` address can't permanently brick control) |

Its constructor deploys its **own** `MockMYR` token instance (`myr = new MockMYR()`), so the loan contract is the token's `minter` by construction, with no separate wiring step and no way to end up with a `CryptoLoan` pointed at the wrong MYR token (see [6.5](#65-what-deployts-actually-does) for how this shapes the deploy script's fresh-node requirement).

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
uint256 public constant MYR_DECIMALS     = 1e6;
uint256 public constant MAX_PRICE_CHANGE = 20;   // max 20% price move per update
```

| Constant           | Value                  | Meaning                                                                                                                      |
| ------------------ | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `MAX_LTV`          | 70%                    | The ceiling `borrow()` enforces against collateral value (matches the "70% Max LTV" reference card in Section 7.5 Portfolio) |
| `LIQ_THRESHOLD`    | 80%                    | The collateral-value multiplier used in the health-factor formula (§9.7)                                                     |
| `LIQ_BONUS`        | 5%                     | Extra collateral a liquidator seizes on top of the debt they cover, in `liquidate()`                                         |
| `MIN_HEALTH`       | `1e18` (i.e. HF = 1.0) | Below this, a position is liquidatable                                                                                       |
| `MYR_DECIMALS`     | `1e6`                  | MYR token precision (matches `MockMYR.decimals()`)                                                                           |
| `MAX_PRICE_CHANGE` | 20%                    | Max allowed move per `setEthPrice()` call (a circuit breaker against a fat-fingered or manipulated oracle update)            |

The borrow-rate model adds a second set of constants:

```solidity
uint256 public baseRateBps            = 300;  // owner-adjustable (initial 3.0%)
uint256 public constant UTIL_SLOPE_BPS   = 400;  // up to +4.0% as lending capacity fills
uint256 public constant VOL_SLOPE_BPS    = 300;  // up to +3.0% on a max (20%) ETH/MYR move
uint256 public constant MAX_BASE_RATE_BPS = 1500; // hard cap: 15%
```

`UTIL_SLOPE_BPS` and `VOL_SLOPE_BPS` are defined but **not currently applied**. `currentAprBps()` (§9.7) returns `baseRateBps` alone. The contract's own comment explains why: layering utilization/volatility premiums on top made the effective rate quoted to borrowers diverge from what they were later charged on repay, so the simpler always-base-rate behaviour was kept instead. Worth stating plainly in the write-up as a known simplification, not an oversight, and a natural entry for Section 12, Future Improvements, in the full document.

### 9.3 State Variables

| Variable          | Type                         | Purpose                                                                                          |
| ----------------- | ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `ethPrice`        | `uint256`                    | MYR per ETH, whole number (e.g. `18000`), the on-chain price oracle                              |
| `prevEthPrice`    | `uint256`                    | Price before the last update, input to the (currently unused) volatility premium                 |
| `lastPriceTime`   | `uint256`                    | Timestamp of the last `setEthPrice()` call                                                       |
| `protocolFees`    | `uint256`                    | Accumulated interest revenue (MYR units), payable out via `withdrawProtocolFees()`               |
| `totalBorrowed`   | `uint256`                    | Protocol-wide outstanding debt (MYR units)                                                       |
| `totalCollateral` | `uint256`                    | Protocol-wide collateral (wei)                                                                   |
| `kycApproved`     | `mapping(address ⇒ bool)`    | Per-wallet gate for `borrow()`/`buyMYR()`, set only by the owner via `setKYC()` (§9.6)           |
| `loans`           | `mapping(address ⇒ Loan)`    | Each wallet's aggregate position                                                                 |
| `liquidators`     | `mapping(address ⇒ bool)`    | Whitelisted addresses allowed to call `liquidate()`                                              |
| `supplyStart`     | `mapping(address ⇒ uint256)` | Timestamp a depositor's supply-interest accrual clock started; `0` means "not currently earning" |

```solidity
struct Loan {
    uint256 collateral;    // wei
    uint256 principal;     // MYR units (6 decimals), current outstanding
    uint256 startTime;     // unix timestamp of first borrow
    uint256 lastRepayTime; // unix timestamp of last repayment (interest reset point)
}
```

`Loan` is a single aggregate per wallet. The contract has no concept of separate borrow "tranches". That itemisation (per-borrow APR, term, and progress) is reconstructed off-chain in the `BorrowPosition` table (see [8.4](#84-table-by-table-walkthrough)) purely for the Repay tab's UI.

### 9.4 Events

| Event                                                         | Emitted by                                                     | Carries                                                |
| ------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------ |
| `CollateralDeposited(user, amount)`                           | `depositCollateral()`                                          | wei deposited                                          |
| `Borrowed(user, myrAmount, newTotal)`                         | `borrow()`                                                     | MYR minted, wallet's new principal                     |
| `Repaid(user, principal, interest)`                           | `repay()`                                                      | how much of the payment went to principal vs. interest |
| `CollateralWithdrawn(user, amount)`                           | `withdrawCollateral()`                                         | wei returned                                           |
| `Liquidated(user, liquidator, debtCovered, collateralSeized)` | `liquidate()`                                                  | full liquidation outcome                               |
| `PriceUpdated(oldPrice, newPrice, updatedBy)`                 | `setEthPrice()`                                                | oracle update, with who pushed it                      |
| `BaseRateUpdated(oldRateBps, newRateBps)`                     | `setBaseRate()`                                                | market-rate change                                     |
| `KYCSet(user, approved)`                                      | `setKYC()`                                                     | entitlement flip                                       |
| `LiquidatorSet(liquidator, approved)`                         | `setLiquidator()`                                              | whitelist change                                       |
| `ProtocolFeesWithdrawn(to, amount)`                           | `withdrawProtocolFees()`                                       | treasury payout (Section 7.12)                         |
| `MYRPurchased(buyer, ethSpent, myrReceived)`                  | `buyMYR()`                                                     | ETH→MYR swap                                           |
| `SupplyInterestClaimed(user, amount)`                         | `claimSupplyInterest()`, and implicitly `withdrawCollateral()` | depositor interest payout                              |

These are the events `LoanTransaction` rows in the database mirror ([8.4](#84-table-by-table-walkthrough)), and what the public Explorer (Section 7.6) filters by type.

### 9.5 Core Function Walkthroughs

Every state-changing user action pairs one `CryptoLoan` function with one callback in `WalletContext.tsx`. All six share the same shape: build the transaction, `await tx.wait()`, mirror the receipt into Postgres via `saveTxToDB`, then call `refresh()` to re-read the wallet's on-chain position.

#### 9.5.1 Deposit Collateral

```solidity
function depositCollateral() external payable whenNotPaused nonReentrant {
    require(msg.value > 0, "Send ETH");
    if (supplyStart[msg.sender] == 0) {
        supplyStart[msg.sender] = block.timestamp;
    }
    loans[msg.sender].collateral += msg.value;
    totalCollateral              += msg.value;
    emit CollateralDeposited(msg.sender, msg.value);
}
```

Any ETH sent with the call becomes collateral and, if this is the wallet's first deposit, starts the supply-interest accrual clock (§9.7) in the same transaction.

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

`guardTx()` runs first on every action (not just this one). It refuses the call unless the connected MetaMask account *is* the account's own linked wallet, surfacing a precise message ("wrong account selected", "no wallet linked") instead of letting a mismatched signer silently fail on-chain. **UI**: the Deposit tab (Section 7.7), with 25/50/75/MAX quick-fill buttons and a live "max you could borrow at 70% LTV" preview computed from the same `_maxBorrow` math the contract uses (§9.7).

#### 9.5.2 Borrow MYR

```solidity
function borrow(uint256 myrAmount) external whenNotPaused nonReentrant onlyKYC {
    require(myrAmount > 0, "Amount must be > 0");
    Loan storage loan = loans[msg.sender];
    require(loan.collateral > 0, "Deposit collateral first");

    uint256 maxBorrow = _maxBorrow(loan.collateral);
    require(loan.principal + myrAmount <= maxBorrow, "Exceeds max LTV");

    if (loan.startTime == 0) {
        loan.startTime     = block.timestamp;
        loan.lastRepayTime = block.timestamp;
    }
    loan.principal += myrAmount;
    totalBorrowed  += myrAmount;

    myr.mint(msg.sender, myrAmount);
    emit Borrowed(msg.sender, myrAmount, loan.principal);
}
```

The `onlyKYC` modifier reverts before touching any state if `kycApproved[msg.sender]` is `false`. The frontend bridge builds in a **self-healing retry** for exactly that revert reason:

```ts
try {
  await attemptBorrow();
  return true;
} catch (e) {
  const reason = revertReason(e);
  if (reason === 'KYC required') {
    // On-chain KYC lost (e.g. contract redeploy) — re-sync silently and retry once.
    setTx('pending', 'Re-syncing KYC…');
    await resyncKyc(addr);
    try {
      await attemptBorrow();
      return true;
    } catch (e2) {
      const r2 = revertReason(e2);
      setTx('error', r2 ? `Borrow failed: ${r2}` : 'Borrow failed — check LTV or collateral');
      return false;
    }
  }
  setTx('error', reason ? `Borrow failed: ${reason}` : 'Borrow failed — check LTV or collateral');
  return false;
}
```

`attemptBorrow()` itself also writes an itemised tranche row to `BorrowPosition` ([8.4](#84-table-by-table-walkthrough)) right after the on-chain confirmation, capturing the APR and term the UI just quoted:

```ts
await fetch('/api/borrows', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    wallet: addr, principal: units.toString(), aprBps,
    termMonths: opts?.termMonths ?? 1, txHash: receipt.hash,
  }),
});
```

**UI**: the Borrow tab (Section 7.9), with a loan-term picker, MYR amount, full rate breakdown, and a risk-acknowledgement checkbox before submit.

#### 9.5.3 Repay Loan

```solidity
function repay(uint256 myrAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.principal > 0, "No active loan");
    require(myrAmount > 0, "Amount must be > 0");

    uint256 interest = accruedInterest(msg.sender);
    uint256 due      = loan.principal + interest;
    uint256 paying   = myrAmount > due ? due : myrAmount;

    myr.transferFrom(msg.sender, address(this), paying);

    uint256 interestPaid   = paying >= interest ? interest : paying;
    uint256 principalPaid  = paying > interest ? paying - interest : 0;

    protocolFees   += interestPaid;
    totalBorrowed  -= principalPaid;

    if (principalPaid >= loan.principal) {
        loan.principal = 0; loan.startTime = 0; loan.lastRepayTime = 0;
    } else {
        loan.principal    -= principalPaid;
        loan.lastRepayTime = block.timestamp;
    }
    emit Repaid(msg.sender, principalPaid, interestPaid);
}
```

Note there is **no `onlyKYC` modifier**. A user must always be able to pay off debt, even if their verification lapsed. Because the contract must pull MYR from the caller, the bridge is a genuine two-step transaction (`approve`, then `repay`), reflected in the UI's step counter:

```ts
setTx('pending', 'Approving MYR spend…', 1, 2);
// ...pre-flight balance check against the LIVE on-chain MYR balance, not
// the possibly-stale s.myrBalance closure...
const approveTx = await c.myr.approve(CONTRACT_ADDRESSES.CryptoLoan, units);
await approveTx.wait();
setTx('pending', `Repaying RM ${myrAmt}…`, 2, 2);
const repayTx = await c.loan.repay(units);
const repayReceipt = await repayTx.wait();
```

The Repaid event's principal/interest split is decoded straight out of the transaction receipt (not recomputed client-side) and used to settle the exact `BorrowPosition` tranches the payment covered:

```ts
const repaidEvt = (repayReceipt?.logs ?? [])
  .map(l => { try { return c.loan.interface.parseLog(l); } catch { return null; } })
  .find(p => p?.name === 'Repaid');
if (repaidEvt) {
  principalPaid = Number(repaidEvt.args[1] as bigint) / 1e6;
  interestPaid  = Number(repaidEvt.args[2] as bigint) / 1e6;
}
```

Consistent with the contract, collateral is **never auto-returned** by this flow. The receipt dialog says so explicitly on a full payoff. **UI**: the Repay tab (Section 7.10) lists every open tranche as a selectable row and totals "This Month's Bill", with an inline Buy MYR top-up if the wallet's balance is short.

#### 9.5.4 Withdraw Collateral

```solidity
function withdrawCollateral(uint256 weiAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.collateral >= weiAmount, "Not enough collateral");

    uint256 remaining = loan.collateral - weiAmount;
    if (loan.principal > 0) {
        uint256 maxBorrow = _maxBorrow(remaining);
        require(loan.principal <= maxBorrow, "Would violate LTV");
    }

    // A full withdrawal ends supply accrual: pay out any unclaimed supply
    // interest NOW, before accruedSupplyInterest() starts returning 0.
    uint256 payout = remaining == 0 ? accruedSupplyInterest(msg.sender) : 0;

    loan.collateral  -= weiAmount;
    totalCollateral  -= weiAmount;
    if (loan.collateral == 0) supplyStart[msg.sender] = 0;

    if (payout > 0) {
        myr.mint(msg.sender, payout);
        emit SupplyInterestClaimed(msg.sender, payout);
    }
    (bool ok, ) = payable(msg.sender).call{value: weiAmount}("");
    require(ok, "ETH transfer failed");
    emit CollateralWithdrawn(msg.sender, weiAmount);
}
```

The bridge decodes the (possible) `SupplyInterestClaimed` log emitted alongside `CollateralWithdrawn` so a full exit's success message correctly reports both amounts in one line, rather than looking like the interest was simply forfeited:

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

A straight mint-for-ETH swap that never touches `loans[msg.sender]`. The bridge pads the ETH it sends by 0.1% to absorb rounding, trusting the contract's own excess-refund to return whatever wasn't needed:

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

Lets a depositor claim their share of borrow-side interest (38% of the borrow APR, §9.7) without touching their collateral or debt position. The accrual clock simply resets to `now`.

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

Two admin-only levers are exercised automatically by the server (using `OWNER_PRIVATE_KEY`) rather than by a human clicking a contract call directly.

**KYC approval**, via [`web/src/lib/kyc/chain.ts`](../../../web/src/lib/kyc/chain.ts), wraps `setKYC()` so the user never pays gas or signs anything to get approved:

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

**Price & rate oracle**, via [`web/src/lib/price-sync.ts`](../../../web/src/lib/price-sync.ts), drives `setEthPrice()` and `setBaseRate()`:

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

The remaining admin levers are simple owner-gated one-liners: `setLiquidator()` (whitelist toggle), `pause()`/`unpause()` (emergency stop for every `whenNotPaused` function), and `withdrawProtocolFees(to)` (pays accumulated interest revenue to `to`, exposed as the Overview page's treasury button, Section 7.12).

### 9.7 View Functions, Health Factor & Interest Model

| Function                      | Returns                                                                                                                                                                                                                           |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `currentAprBps()`             | Live borrow APR in basis points (currently `baseRateBps` only; §9.2)                                                                                                                                                              |
| `accruedInterest(user)`       | Interest owed since `lastRepayTime`, at the current rate                                                                                                                                                                          |
| `totalDue(user)`              | `principal + accruedInterest(user)`                                                                                                                                                                                               |
| `healthFactor(user)`          | See below                                                                                                                                                                                                                         |
| `availableToBorrow(user)`     | Remaining borrowable MYR at `MAX_LTV`                                                                                                                                                                                             |
| `currentLTV(user)`            | Wallet's current loan-to-value, as a whole-number percentage                                                                                                                                                                      |
| `getLoanInfo(user)`           | One-call bundle of collateral, debt, health factor, available-to-borrow, collateral value, interest, LTV, and a liquidatable flag, read by `WalletContext.tsx`'s `refresh()` to populate every live card in Section 7.3 Dashboard |
| `supplyInterestRate()`        | Supply APR in bps, `38%` of the borrow rate (the depositor's cut, per Section 7.5)                                                                                                                                                |
| `accruedSupplyInterest(user)` | MYR accrued since `supplyStart[user]`                                                                                                                                                                                             |
| `getProtocolStats()`          | Protocol-wide totals, treasury figures for Section 7.12 Admin Overview                                                                                                                                                            |

**Health factor**, in `_healthFactor()`:

```solidity
function _healthFactor(address user) internal view returns (uint256) {
    if (loan.principal == 0) return type(uint256).max; // no debt = infinitely healthy
    uint256 collateralMYR = (loan.collateral * ethPrice * MYR_DECIMALS) / PRECISION;
    return (collateralMYR * LIQ_THRESHOLD * PRECISION) / (100 * loan.principal);
}
```

In plain terms: `HF = (collateral value in MYR × 80%) / debt`, scaled to 18-decimal fixed point (`PRECISION = 1e18`, so `MIN_HEALTH = 1e18` reads as `HF = 1.0`). A position becomes liquidatable the moment this drops below `1.0`. `refresh()` reads this straight off `getLoanInfo()` and converts it once for the whole UI:

```ts
const MAX_U = BigInt('0xff...ff'); // type(uint256).max
const hfRaw = info[2] as bigint;
const hf = hfRaw === MAX_U ? Infinity : Number(hfRaw) / 1e18;
```

This is exactly the number the plain-English risk sentence on the Dashboard ("At Risk: liquidates if ETH falls to or below RM X") and the Portfolio's escalating risk banner (Section 7.5) are both derived from.

**Interest accrual**, in `accruedInterest()`:

```solidity
uint256 elapsed = block.timestamp - loan.lastRepayTime;
return (loan.principal * currentAprBps() * elapsed) / (10_000 * 365 days);
```

Simple (non-compounding) interest, applied over the *whole* elapsed period at whatever `currentAprBps()` is **right now**, not indexed per rate-change the way a production money-market protocol would. The contract's own comment accepts this as "a demo-grade simplification", noting the error window stays small because the rate only moves on explicit admin actions. Supply-side interest (`accruedSupplyInterest()`) uses the identical linear formula, just at `supplyInterestRate()` (38% of the borrow rate) instead.

`liquidate(borrower, debtAmount)` (whitelisted liquidators only) uses this same health-factor check as its entry guard (`require(_healthFactor(borrower) < MIN_HEALTH, "Not liquidatable")`), then repays up to `debtAmount` of the borrower's debt and seizes the equivalent collateral plus the 5% `LIQ_BONUS`. There is no liquidation UI in the current app; see Section 12, Future Improvements, in the full document.

### 9.8 Supporting Contracts

**[`MockMYR.sol`](../../../blockchain/contracts/MockMYR.sol)**, the 6-decimal ERC-20 that `CryptoLoan` mints on every `borrow()`, `buyMYR()`, and interest payout. Minting is restricted to a single `minter` address, fixed at construction time to whichever contract deployed it:

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

**`ICO.sol` / `RinggitToken.sol`**, a separate demo behind the `/ico` page, unrelated to the lending protocol proper (see [6.8](#68-optional-ico-demo-deployment)). `RinggitToken` is a standard 18-decimal ERC-20 (`MYRC`) that mints its entire fixed supply to the deployer at construction. `ICO` sells that supply for ETH at a fixed price set at deploy time, using `safeTransferFrom(owner() → buyer)` against an allowance the deployer grants it. The ICO contract never custodies the token supply itself, only an approval to move it:

```solidity
function buyToken() external payable whenNotPaused {
    uint256 numberOfToken = (msg.value * 1e18) / price;
    token.safeTransferFrom(owner(), msg.sender, numberOfToken);
    totalTokensSold += numberOfToken;
    emit TokensPurchased(msg.sender, msg.value, numberOfToken);
}
```

It shares the `Pausable` / `Ownable` / `ReentrancyGuard` pattern with `CryptoLoan`, and uses `call` rather than `transfer` for its `withdrawal()` function specifically so an owner that's a smart-contract wallet (not a plain EOA) isn't at risk of reverting on `transfer`'s fixed 2300-gas stipend.
