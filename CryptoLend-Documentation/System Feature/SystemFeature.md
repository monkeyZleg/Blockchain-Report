
## 7. System Features

This section documents every user-facing and admin-facing feature of CryptoLend as it is actually implemented, verified by reading the live source rather than by re-describing the proposal. Each feature lists what it's for, who can use it, the exact user flow, and what happens behind the scenes (which API route or smart-contract function it calls).
### Table of contents, Section 7

- [7.1 User Registration (Sign Up)](#71-user-registration-sign-up)
- [7.2 User Login](#72-user-login)
- [7.3 Dashboard](#73-dashboard)
- [7.4 Markets](#74-markets)
- [7.5 Portfolio](#75-portfolio)
- [7.6 Explorer](#76-explorer)
- [Main Lending Features (5 Core Actions)](#main-lending-features-5-core-actions)
  - [7.7 Deposit Collateral](#77-deposit-collateral)
  - [7.8 Withdraw Collateral](#78-withdraw-collateral)
  - [7.9 Borrow MYR](#79-borrow-myr)
  - [7.10 Repay Loan](#710-repay-loan)
  - [7.11 Buy MYR](#711-buy-myr)
- [7.12 Admin Panel](#712-admin-panel)
- [7.13 KYC Verification](#713-kyc-verification)
- [7.14 Settings](#714-settings)

---

### 7.1 User Registration (Sign Up)

**Access:** Public (unauthenticated). **Route:** `/signup`.

CryptoLend supports two independent ways to open an account, and neither is the "primary" method.

**A. Email & password**

| Field            | Rule                                                                                                                       |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Full name        | Optional                                                                                                                   |
| Email address    | Required, must be a syntactically valid email                                                                              |
| Password         | Required, minimum 8 characters. A live strength meter (Very weak to Very strong) scores length, casing, digits and symbols |
| Confirm password | Must match                                                                                                                 |

*Flow:*
1. Fill in the form.
2. Click **Create Account** (disabled until all required fields are valid and the passwords match).
3. The account is created and the user is **immediately signed in** (no separate login step).
4. Redirect to `/dashboard`.

![CreateAcc_Email](CreateAcc_Email.png)
<center><u>Figure 7.1.1: Sign up page</u></center>


**B. Continue with MetaMask**
![](images/SignUpMM.png)
<center><u>Figure 7.1.2: Login with MetaMask (Confirmation) page</u></center>
*Flow:*
1. Click **Continue with MetaMask**, and MetaMask's account picker opens.
2. The app requests a one-time nonce and has the wallet sign a `"Sign in to CryptoLend"` message over it (this proves ownership without ever touching a private key).
3. The signature is verified server-side.
4. Since the wallet is new, an account is created automatically with an auto-generated display name (e.g. `0x7099…79c8`) and no email/password.
5. The user is signed in and redirected to `/dashboard`.

![](images/MMUsername&Password.png)
<center><u>Figure 7.1.3: MetaMask Registered Account without Email and Password</u></center>

Both paths write to the same `User` table and produce a normal signed-in session. A wallet-only account can add an email/password later from [Settings](#714-settings), and an email/password account can link a wallet later the same way.

Sign-ups can be temporarily closed by an admin via the `auth.signup` feature flag (see [7.12](#712-admin-panel)). This is checked on every submit, not just at page load.


```ts
import { NextResponse } from 'next/server';
import bcrypt from 'bcryptjs';
import { prisma } from '@/lib/db/prisma';
import { createToken, setAuthCookie } from '@/lib/auth-jwt';
import { featureBlocked } from '@/lib/features-server';

export async function POST(req: Request) {

  // Registration can be closed from the admin feature panel.
  const blocked = await featureBlocked('auth.signup');
  if (blocked) return blocked;

  const { name, email, password } = await req.json();
  if (!email || !password) return NextResponse.json({ error: 'Email and password are required' }, { status: 400 });
  if (password.length < 8) return NextResponse.json({ error: 'Password must be at least 8 characters' }, { status: 400 });

  const normalEmail = (email as string).toLowerCase().trim();
  const existing = await prisma.user.findUnique({ where: { email: normalEmail } });
  if (existing) return NextResponse.json({ error: 'Email already registered' }, { status: 409 });

  const hash = await bcrypt.hash(password, 12);
  const user = await prisma.user.create({ data: { name: name?.trim() || null, email: normalEmail, password: hash } });
  const token = await createToken({
    id: user.id, email: user.email, name: user.name, epoch: user.sessionEpoch,
  });
  await setAuthCookie(token);
  return NextResponse.json({ id: user.id, email: user.email, name: user.name }, { status: 201 });
}
```

---

### 7.2 User Login

**Access:** Public (unauthenticated). **Route:** `/login`.

Mirrors Sign Up: **email/password**, or **Continue with MetaMask** (same nonce-and-signature proof, message text `"Sign in to CryptoLend"`). Logging in with a MetaMask account that isn't registered yet transparently creates one. There is no separate "wallet not recognised" error, by design.

Flow:
1. Enter credentials, or connect a wallet.
2. Click **Sign In**.
3. On success, admins are always routed to `/admin` regardless of where they came from. Everyone else lands on `/dashboard` (or the page they were originally trying to reach, if they were redirected to login first).

Sign-in can be paused independently via the `auth.login` flag. Admin accounts are explicitly exempt from this, so the team can never lock itself out of the admin panel.

![Login page showing the email/password form and the "Continue with MetaMask" option](images/loginSelection.png)
<center><u>*Figure 7.2.1: Login page*</u></center>

```ts
import { NextResponse } from 'next/server';
import bcrypt from 'bcryptjs';
import { prisma } from '@/lib/db/prisma';
import { createToken, setAuthCookie } from '@/lib/auth-jwt';
import { featureBlocked } from '@/lib/features-server';

  
export async function POST(req: Request) {
  try {
    const { email, password } = await req.json();
    if (!email || !password) return NextResponse.json({ error: 'Email and password are required' }, { status: 400 });

    const normalEmail = (email as string).toLowerCase().trim();
    let user;
    try {
      user = await prisma.user.findUnique({ where: { email: normalEmail } });
    } catch (err) {
      console.error('[login] DB error:', err);
      return NextResponse.json({ error: 'Database error' }, { status: 500 });
    }

    if (!user?.password) return NextResponse.json({ error: 'Invalid email or password' }, { status: 401 });

    const valid = await bcrypt.compare(password, user.password);
    if (!valid) return NextResponse.json({ error: 'Invalid email or password' }, { status: 401 });

    // Sign-in can be paused from the admin feature panel — but only after the
    // password is verified, and never for admins. Checking it here rather than
    // at the top means a paused login cannot be used to probe which emails
    // exist, and guarantees an admin can always get back in to un-pause it.
    if (!user.isAdmin) {
      const blocked = await featureBlocked('auth.login');
      if (blocked) return blocked;
    }

    const token = await createToken({
      id: user.id,
      email: user.email,
      name: user.name,
      isAdmin: user.isAdmin,
      epoch: user.sessionEpoch,
    });

    await setAuthCookie(token);

    return NextResponse.json({
      id: user.id, email: user.email, name: user.name,
      isAdmin: user.isAdmin, status: user.status,
    });
  } catch (err) {
    console.error('[login] Unexpected error:', err);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

---

### 7.3 Dashboard

**Access:** Public to view. Live figures and all actions require a connected, linked wallet. **Route:** `/dashboard`.

The Dashboard is the app's home screen once signed in, and the entry point to every lending action.


![Dashboard overview showing the status header, protocol stats, and the four live position cards](images/DashboardOverview.png)
<center><u>Figure 7.3.1: Dashboard overview (User)</u></center>

- **Status header**: connection state (`Connected · 0x7099…79c8` · Chain 31337), a *Live* chip, and a *KYC Verified* / *Admin, full access* chip once entitled.
- **Protocol stats banner**: four figures, of which only the **ETH/MYR price** is genuinely live (read from the contract's price oracle). Total Value Locked, Active Loans and Total Borrowed here are illustrative demo figures, not computed from real state. Worth being explicit about this in the write-up rather than presenting them as live analytics.
- **Your position**: four live cards once connected: My Collateral, Outstanding Debt, Net Position, Health Factor (with a plain-English risk sentence, e.g. *"At Risk: liquidates if ETH falls to or below RM X"*).
- **Price self-healing**: if the on-chain price oracle drifts more than 3% from the live market price, a warning appears with a manual **Sync Price** button. The app also retries this automatically in the background.
- **Markets & Calculator panel**: a 9-asset rate table (price, 24h sparkline, Max LTV, Borrow/Supply APR, liquidity) feeding a "what could I borrow" calculator: collateral amount, an LTV slider, a loan-term picker, and a "Hold vs. Sell" comparison tool that estimates the payoff of borrowing against an asset instead of selling it. **Important caveat for the write-up:** this calculator is illustrative across all 9 assets, but the platform's real collateral asset is **ETH only**. The five action tabs below only ever move ETH and MYR.
- **Manage Position dialog**: one modal, opened via `/dashboard?tab=deposit|withdraw|borrow|repay|buy`, holding all five core actions described in [7.7 to 7.11](#main-lending-features-5-core-actions). A shared header shows a KYC-pending banner where relevant, a live transaction status banner (pending/success/error, with step-by-step progress for multi-step transactions), and, once the user has a position, a summary strip (Collateral / Borrowed / Earning / Health).


![Markets & Calculator panel with an asset selected and the borrow calculator expanded](images/MarketCalculator.png)
<center><u>Figure 7.3.2: Markets & Calculator panel</u></center>

```ts
// health-factor / live-position read logic from web/src/lib/WalletContext.tsx

function readCachedPosition(addr: string): CachedPosition | null {
  if (typeof window === 'undefined') return null;
  try {
    const raw = localStorage.getItem(LS_KEY(addr));
    if (!raw) return null;
    const d = JSON.parse(raw);
    return {
      info: {
        collateral:         BigInt(d.info.collateral),
        borrowed:           BigInt(d.info.borrowed),
        available:          BigInt(d.info.available),
        accruedInterest:    BigInt(d.info.accruedInterest),
        startTime:          BigInt(d.info.startTime),
        // Falls back to startTime for cache entries written before this field.
        lastRepayTime:      BigInt(d.info.lastRepayTime ?? d.info.startTime ?? 0),
        healthFactor:       d.info.healthFactor === 'Infinity' ? Infinity : Number(d.info.healthFactor),
        collateralValueMYR: Number(d.info.collateralValueMYR),
        ltv:                Number(d.info.ltv),
        isLiquidatable:     Boolean(d.info.isLiquidatable),
      },

      ethBalance: d.ethBalance ?? '0',
      myrBalance: d.myrBalance ?? '0',
      ethPriceMYR: Number(d.ethPriceMYR) || 18000,
      savedAt: Number(d.savedAt) || 0,
    };
  } catch { return null; }
}
```

---

### 7.4 Markets

**Access:** Public. **Route:** `/markets`.

A dedicated rates-comparison page listing 10 assets (the Dashboard's 9 plus MATIC), meant as a browsing/discovery entry point rather than a transaction screen.

- **Market stats row**: Total Value Locked and Total Borrowed are static demo figures (documented here for the same reason as the Dashboard). **Avg Borrow APR** and **Avg Supply APR** are genuinely live, averaged from the on-chain base rate.
- **Filtering**: risk-profile chips (*Conservative*, *Balanced*, *High APR*), free-text search, and a table/card view toggle.
- **Sortable table**: Price, 24h change (with sparkline), Max LTV, Liquidation Threshold, Borrow APR, Supply APR, Utilisation (colour-coded).
- **Row actions**: **Borrow** deep-links into the Dashboard's Borrow tab with that asset pre-selected in the calculator. **Earn** is enabled for ETH only (the only real supply/collateral asset, see the 7.3 caveat above). Other assets show "Supply coming soon".

![Markets page showing the sortable rate table with the risk-profile filter chips](images/7.4-markets.png)
*Figure 7.4.1: Markets page*

```ts
// TODO: paste the dynamicApr()/supplyApr() rate formulas from web/src/lib/rates.ts
```

---

### 7.5 Portfolio

**Access:** Requires sign-in and a connected wallet. **Route:** `/portfolio`.

The user's personal position and history page.

- **Risk alert**: a red banner appears once health factor drops below 1.5, with direct **Add Collateral** / **Repay Now** shortcuts into the Dashboard. Wording escalates to "Liquidation Imminent" below 1.2.
- **Overview cards**: ETH Collateral, Total Borrowed (plus % LTV used), Accrued Interest, Health Factor.
- **Active Position card**: progress bars for Collateral, Loan-to-Value (against the 70% max) and Health Factor, plus the estimated **liquidation price** in MYR.
- **Supply Earnings**: deposited ETH earns interest automatically (38% of the borrow APR is the depositor's share). This card shows the accrued amount and a **Claim MYR** button.
- **Loan Timeline & Interest**: an illustrative 90-day repayment projection (principal, accrued interest, projected total if held the full term) with a **Repay Loan** shortcut.
- **Transaction History**: the wallet's own on-chain events (deposits, borrows, repayments, withdrawals, MYR purchases), read live from the blockchain, falling back to the database mirror if a live read isn't available.
- **Wallet Balances**: ETH and MYR balances, with a one-click **Add MockMYR to MetaMask** button.
- **Sidebar**: a static Loan Terms reference card (APR, 70% Max LTV, 80% Liquidation Threshold, 10% Liquidation Penalty), Quick Actions, Net Position (Collateral - Debt - Accrued Interest), and a Connection status summary.

![Portfolio page showing the active position, health factor bars, and Supply Earnings card](images/7.5-portfolio-position.png)
*Figure 7.5.1: Portfolio, active position*

![Portfolio page showing the transaction history list](images/7.5-portfolio-history.png)
*Figure 7.5.2: Portfolio, transaction history*

```solidity
// TODO: paste claimSupplyInterest() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.6 Explorer

**Access:** Fully public, no sign-in required. **Route:** `/explorer`.

A public, read-only feed of on-chain protocol activity, deliberately scrubbed of anything identifying: wallet addresses and transaction hashes are shortened, and no KYC or personal data is ever exposed here, only what the blockchain already makes public.

- **Live feed**: auto-refreshes every 15 seconds.
- **Activity summary**: a running total plus one count per event type (Deposited / Withdrawn / Borrowed / Repaid / MYR Purchased). Clicking a type filters the table to it.
- **Filter bar**: free-text search (matches a wallet, a transaction hash, or a block number), event type, sort order, and a date range.
- **Table**: Block number, Type (colour-coded), Amount, Wallet (shortened), Tx Hash (shortened), and a relative timestamp ("2m ago"). The table is paginated.

![Explorer page showing the live transaction feed with type filters and the activity summary cards](images/ExplorerActive.png)
<center><u>Figure 7.6.1: Public transaction explorer</u></center>

```ts
// Query builder from web/src/app/api/explorer/route.ts (or lib/tx-query.ts)
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/db/prisma';
import { getFlags } from '@/lib/features-server';
import { isHidden } from '@/lib/features';
import { buildTxWhere, parseTxFilters, shortWallet } from '@/lib/tx-query';

export async function GET(req: NextRequest) {
  const flags = await getFlags();
  if (isHidden(flags, 'page.explorer')) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }

  
  const sp      = req.nextUrl.searchParams;
  const filters = parseTxFilters(sp);
  const page    = Math.max(1, Number(sp.get('page') ?? 1) || 1);
  const pageSize = Math.min(100, Math.max(10, Number(sp.get('pageSize') ?? 25) || 25));

  // Chronological (oldest first) by default — reads like a ledger; the UI
  // offers newest-first as an option.

  const order: 'asc' | 'desc' = sp.get('order') === 'desc' ? 'desc' : 'asc';
  const where = buildTxWhere(filters);


  try {
    const [total, rows, stats] = await Promise.all([
      prisma.loanTransaction.count({ where }),
      prisma.loanTransaction.findMany({
        where,
        orderBy: [{ blockNumber: order }, { createdAt: order }],
        skip: (page - 1) * pageSize,
        take: pageSize,
        select: { id: true, wallet: true, type: true, amount: true, txHash: true, blockNumber: true, createdAt: true },
      }),
      prisma.loanTransaction.groupBy({ by: ['type'], _count: { _all: true } }),
    ]);

    return NextResponse.json({
      txs: rows.map(t => ({
        id:          t.id,
        type:        t.type,
        amount:      t.amount,
        blockNumber: t.blockNumber,
        createdAt:   t.createdAt,

        // Truncated server-side: the full address is never sent to a public
        // client, so it cannot be recovered from the network tab.

        wallet:      shortWallet(t.wallet),
        txHash:      `${t.txHash.slice(0, 12)}…${t.txHash.slice(-8)}`,
      })),

      total,
      page,
      pageSize,
      pages: Math.max(1, Math.ceil(total / pageSize)),
      byType: Object.fromEntries(stats.map(s => [s.type, s._count._all])),
    });

  } catch (err) {
    console.error('[GET /api/explorer]', err);
    return NextResponse.json({ error: 'Database error' }, { status: 500 });
  }
}
```

---

### Main Lending Features (5 Core Actions)

The five actions below are the heart of the product. All five live inside the **same** "Manage Position" dialog on the Dashboard, switched via an in-modal tab strip. Each maps to exactly one function on the `CryptoLoan` smart contract, and to one of the five independently pausable **"Loan actions"** feature flags an admin can control (see [7.12](#712-admin-panel)).

| # | Action | KYC required? | Contract function | What it does |
|---|---|:---:|---|---|
| 7.7 | Deposit Collateral | Yes | `depositCollateral()` | Sends ETH into the contract as collateral and starts earning supply interest |
| 7.8 | Withdraw Collateral | No | `withdrawCollateral(weiAmount)` | Returns ETH collateral, as long as the remaining position stays under 70% LTV |
| 7.9 | Borrow MYR | Yes | `borrow(myrAmount)` | Mints MYR against deposited collateral, up to 70% of its value |
| 7.10 | Repay Loan | No | `repay(myrAmount)` | Repays outstanding MYR debt, interest first, then principal |
| 7.11 | Buy MYR | Yes | `buyMYR(myrAmount)` | Swaps ETH for MYR at the live on-chain price (e.g. to top up before repaying) |

Withdraw and Repay are intentionally **never** KYC-gated. A user must always be able to reduce risk and exit a position, even if their verification lapses or was never completed.

---

### 7.7 Deposit Collateral

**Access:** Signed in, wallet linked, **KYC approved**. **Route:** `/dashboard?tab=deposit`.

Flow:
1. Open the Deposit tab.
2. Enter an ETH amount (25% / 50% / 75% / MAX quick-fill buttons read the live wallet balance).
3. Review the computed panels (maximum borrowable amount at 70% LTV, live estimated Supply APR earnings, estimated gas).
4. Confirm in MetaMask.

On confirmation, the contract's `depositCollateral()` is called with the ETH amount attached as `value`. This both credits the collateral and starts the clock on supply-interest accrual for that wallet. A receipt dialog confirms the deposit and the new collateral total.

![Deposit tab showing amount entry, quick-fill buttons, and the "Max you can borrow" panel](images/7.7-deposit.png)
*Figure 7.7.1: Deposit Collateral*

```solidity
// TODO: paste depositCollateral() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.8 Withdraw Collateral

**Access:** Signed in, wallet linked (no KYC required). **Route:** `/dashboard?tab=withdraw`.

Flow:
1. Open the Withdraw tab.
2. Enter an ETH amount, capped so the *remaining* position can never drop below the 70% max-LTV floor while a loan is open.
3. Review the "After Withdrawal" panel.
4. Confirm in MetaMask.

On confirmation, `withdrawCollateral(weiAmount)` is called. A detail worth documenting: **withdrawing 100% of collateral automatically pays out any unclaimed supply interest in the same transaction**, since the contract settles it before the accrual clock resets, so nothing is left stranded.

![Withdraw tab showing amount entry with the LTV-safe ceiling and the "After Withdrawal" panel](images/7.8-withdraw.png)
*Figure 7.8.1: Withdraw Collateral*

```solidity
// TODO: paste withdrawCollateral() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.9 Borrow MYR

**Access:** Signed in, wallet linked, **KYC approved**. **Route:** `/dashboard?tab=borrow`.

Flow:
1. Open the Borrow tab.
2. Pick a loan term (1, 3, 6 or 12 months). This is illustrative, since interest itself is not fixed-term.
3. Enter a MYR amount (MAX reads the real on-chain borrowing limit).
4. Review the full rate breakdown (base rate, market premium, effective APR, projected interest and health factor).
5. Tick the risk-acknowledgement checkbox.
6. Click **Review** to open a confirmation dialog summarising the loan, then **Confirm Borrow**.

On confirmation, `borrow(myrAmount)` mints MYR straight to the wallet, up to 70% of collateral value. An itemised ledger row is also saved off-chain (principal, locked APR, term) to power the instalment breakdown used in [Repay](#710-repay-loan).

![Borrow tab showing amount entry and the rate/repayment breakdown](images/7.9-borrow.png)
*Figure 7.9.1: Borrow MYR*

![Borrow confirmation dialog summarising the loan before it is submitted](images/7.9-borrow-confirm.png)
*Figure 7.9.2: Borrow confirmation*

```solidity
// TODO: paste borrow() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.10 Repay Loan

**Access:** Signed in, wallet linked (no KYC required). **Route:** `/dashboard?tab=repay`.

The most involved of the five tabs, modelled on a bill-pay experience rather than a single "repay all" button.

Flow:
1. The tab lists every open borrow as a selectable row (principal, locked APR, "Month M of N", interest accrued on that tranche).
2. Ticking one or more rows totals **This Month's Bill**.
3. The Repay Amount field follows that total automatically, or can be hand-edited, with 25% / 50% / 75% / FULL quick-fill.
4. If the wallet's MYR balance is short, a **Buy MYR** top-up step is offered inline, and can be chained into the repayment in one click.
5. Click **Repay**.

On confirmation this is a two-step transaction: the user first `approve()`s the contract to pull the MYR, then `repay(myrAmount)` executes. Interest is always settled before principal. A full payoff **does not automatically return collateral**. It stays deposited so it can back a new loan immediately, and is only returned via a separate [Withdraw](#78-withdraw-collateral). The confirmed on-chain amounts are then reconciled back into the off-chain instalment ledger.

![Repay tab showing selectable borrow rows, "This Month's Bill", and the MYR top-up-then-repay flow](images/7.10-repay.png)
*Figure 7.10.1: Repay Loan*

```solidity
// TODO: paste repay() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.11 Buy MYR

**Access:** Signed in, wallet linked, **KYC approved**. **Route:** `/dashboard?tab=buy`.

Flow:
1. Open the Buy MYR tab.
2. Enter the amount of MYR wanted. The ETH cost is computed live from the on-chain price.
3. Confirm in MetaMask.

On confirmation, `buyMYR(myrAmount)` is called with slightly more ETH attached than strictly required (a small rounding buffer). The contract mints the requested MYR and **automatically refunds any excess ETH** in the same transaction. This is most commonly used from inside the Repay flow, to top up a shortfall before repaying.

![Buy MYR tab showing amount entry and the live ETH cost breakdown](images/7.11-buy-myr.png)
*Figure 7.11.1: Buy MYR*

```solidity
// TODO: paste buyMYR() from blockchain/contracts/CryptoLoan.sol
```

---

### 7.12 Admin Panel

**Access:** Admin accounts only. **Route:** `/admin/*`. Enforced three times independently (edge middleware, a server-side layout guard that re-reads the live account on every request, and a per-route API guard), so that revoking someone's admin flag takes effect immediately rather than waiting for their session to expire.


![](images/DashboardOverviewAdmin.png)
<center><u>Figure 7.12.1: Dashboard overview (Admin)</u></center>

The admin panel is explicitly scoped to **off-chain enforcement**: restricting a user, for example, stops CryptoLend's own app from acting for them, but cannot stop their wallet from calling the smart contract directly. That boundary is stated in the product itself, and is worth restating in the write-up as a deliberate design decision, not a gap. Six areas, plus one fund-moving action:

- **Overview** (`/admin`): a protocol health dashboard with live treasury figures (contract ETH balance, MYR supply, protocol fees), live on-chain rates, aggregate KPIs (total borrowed, unique wallets, transaction count), charts, and recent activity/audit feeds. Read-only except for the fee-withdrawal button described below.
- **Users** (`/admin/users`): a searchable, paginated directory of every account, with per-user actions: restrict/unrestrict (read-only lockout), reset password (one-time temporary password shown once), reset KYC to pending, unlink wallet (blocked while that wallet still has an open position), and edit profile/admin role. Every action is confirmed before it runs and is written to the audit log.
- **KYC Review** (`/admin/kyc`): the approval queue for submissions from [7.13](#713-kyc-verification), with a searchable table, a detail view per submission (personal info, address, financial declaration, uploaded documents), and Approve / Reject / Delete actions. **Approving is the single most consequential action in the admin panel**: it flips the submission to `approved` and, if the account already has a linked wallet, immediately grants the on-chain `setKYC` flag. This is the only chain-state-changing action exposed through everyday admin use.
- **Transactions** (`/admin/transactions`): a read-only ledger of on-chain activity mirrored from contract events, with full filtering and CSV export.
- **Feature Flags** (`/admin/features`): turns individual pages or loan actions **On**, into **Maintenance** (visible, with a custom message, but blocked), or **Hidden** (removed from navigation, direct visits 404). This is what powers the "temporarily paused" states referenced throughout this section. Admins always bypass their own flags so a paused feature can still be verified.
- **Audit Log** (`/admin/audit`): an append-only, unfilterable-by-design trail of every admin mutation across all of the above (who, what, when, before/after detail). There is deliberately no edit or delete endpoint for it.
- **Withdraw Protocol Fees**: a single button on the Overview page's treasury card. The contract's `withdrawProtocolFees` always pays out to the contract's own on-chain `owner()` address, never anywhere client-supplied, and is gated behind a confirmation prompt.


![Admin KYC review queue with a submission's detail view open](images/7.12-admin-kyc-queue.png)
*Figure 7.12.2: Admin Panel, KYC review*

```solidity
// TODO: paste withdrawProtocolFees() from blockchain/contracts/CryptoLoan.sol,
// and/or setKYC() since that's the function the KYC-approve action calls
```

---

### 7.13 KYC Verification

**Access:** Requires sign-in (no wallet needed to submit). **Route:** `/kyc`.

Identity verification belongs to the **account**, not the wallet. This is a deliberate choice so that verification survives linking a different wallet later, and so a shared test wallet can never "inherit" someone else's approved status. A wallet only needs to be linked afterwards, in [Settings](#714-settings), to actually turn the approval into on-chain borrowing permission.

A 5-step wizard:
1. **Personal Information**: ID type and number, name, date of birth, gender, nationality, contact details.
2. **Address**: Malaysian address with a state selector.
3. **Financial Declaration**: employment status, income bracket, loan purpose, source of funds.
4. **Document Upload**: front and back of ID, compressed and stored securely.
5. **Review & Submit**: a read-only summary plus mandatory Terms-of-Service and truthfulness declarations.

Submitting creates a `pending` record and shows a reference number (`KYC-######`). Returning to the page while pending shows a status screen instead of the form. A rejected submission reopens the wizard pre-filled, with the previous rejection called out, ready to correct and resubmit. There is no automated document verification. Every submission is reviewed by a human admin (see [7.12](#712-admin-panel)). Approval is what ultimately allows the account to deposit, borrow, or buy MYR once a wallet is linked.

![KYC wizard showing the Document Upload step with both ID sides attached](images/7.13-kyc-wizard.png)
*Figure 7.13.1: KYC submission wizard*

![KYC "Under Review" status screen with the reference number](images/7.13-kyc-status.png)
*Figure 7.13.2: KYC status screen*

```ts
// TODO: paste the POST handler from web/src/app/api/kyc/route.ts
```

---

### 7.14 Settings

**Access:** Requires sign-in. **Route:** `/settings`.

Two self-service panels:

- **Account & Sign-in**: display name, email, and password. The system enforces one safety invariant worth calling out explicitly: an email can only be added together with a password, never email-only, so the account can never end up with a login method that's impossible to authenticate with. Changing the password immediately signs out every other active session.
- **Linked Wallet**: this is where the app's **"Connect = Link"** model lives. There is no separate "connected but not linked" state. Clicking **Link MetaMask Wallet** opens MetaMask's account picker and, in one step, proves ownership with a signature and links that wallet to the signed-in account, instantly granting on-chain borrowing permission if the account is already KYC-approved (or is an admin), with no extra review step. If the account already has a linked wallet and a different one is selected in MetaMask, the app refuses and asks the user to switch back, rather than silently re-linking. **Unlinking** is self-service via a confirmation dialog, but only allowed once the account has another way to sign in (email + password) and the wallet's on-chain position is fully clear (no collateral, no debt). KYC approval itself is preserved and reapplies automatically to whatever wallet is linked next.

![Settings page showing the Account & Sign-in panel](images/7.14-settings-account.png)
*Figure 7.14.1: Settings, Account & Sign-in*

![Settings page showing the Linked Wallet panel and the Unlink confirmation dialog](images/7.14-settings-wallet.png)
*Figure 7.14.2: Settings, Linked Wallet*

```ts
// TODO: paste the wallet link/unlink safety checks from
// web/src/app/api/wallet/link/route.ts and .../unlink/route.ts
```

---
