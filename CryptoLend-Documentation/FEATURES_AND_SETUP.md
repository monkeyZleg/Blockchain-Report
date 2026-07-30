# CryptoLend — Feature & Setup Documentation

CryptoLend is a crypto-collateralized MYR (Malaysian Ringgit) lending platform.
Users deposit ETH as collateral, borrow MYR against it, and repay with interest —
all enforced by an on-chain Solidity smart contract, wrapped in a Next.js web app
with email/password and MetaMask authentication, a database-backed KYC flow, and
a live crypto market view.

Stack summary:

| Layer | Technology |
|---|---|
| Smart contracts | Solidity 0.8.24, Hardhat, OpenZeppelin |
| Blockchain client | ethers.js v6, MetaMask |
| Frontend | Next.js 16 (App Router), React 19, MUI v9, Tailwind v4 |
| Backend / API | Next.js API routes, Prisma ORM |
| Database | PostgreSQL (hosted, e.g. Supabase or Railway) |
| Auth | JWT (`jose`), bcrypt, wallet-signature (SIWE-style) login |

---

## 1. System Setup

The repository is a **monorepo with two independent npm projects**, each installed
separately:

```
crypto-loan/
├── blockchain/     # Solidity contracts + Hardhat toolchain
└── web/            # Next.js app (frontend + API routes + Prisma)
```

Node version is pinned to **20** via `.nvmrc` (`nvm use` if you have nvm).

### 1.1 Environment variables

Copy `.env.example` to `.env` **in both places** — the repo root (read by
`blockchain/hardhat.config.ts`) and inside `web/` (read by Next.js/Prisma). These
are two independent physical files, not symlinks — editing one does not update
the other.

```bash
cp .env.example .env
cp .env.example web/.env
```

| Variable | Used by | Purpose |
|---|---|---|
| `DATABASE_URL` | `web/` (Prisma) | PostgreSQL connection string. Default local value: `postgresql://postgres:postgres@localhost:5432/cryptoloan`. For a hosted Postgres (Supabase, Railway, etc.) paste the connection string the provider gives you. |
| `OWNER_PRIVATE_KEY` | `blockchain/` and `web/` | Private key of the contract owner. The **server** uses this to call `setKYC()` on a user's behalf (gasless KYC approval) and to sync the on-chain ETH price. On a local Hardhat node, Hardhat account #0's key is printed to the terminal (also given in the `.env.example` comment). |
| `HARDHAT_RPC_URL` | both | Blockchain RPC endpoint. Defaults to `http://127.0.0.1:8545` (local Hardhat node). |
| `ALCHEMY_API_KEY` | `blockchain/` | RPC provider key for the Sepolia/Mainnet networks (get one at dashboard.alchemy.com). |
| `ETHERSCAN_API_KEY` | `blockchain/` | Needed only to verify contracts on public networks. |
| `REPORT_GAS` | `blockchain/` | Set `true` to print a gas-usage report when running contract tests. |
| `JWT_SECRET` | `web/` | Signs the auth session JWT. Optional in local dev — falls back to a dev value if unset; **must** be set in production. |

### 1.2 Install dependencies

```bash
cd blockchain && npm i
cd ../web && npm i
```

**What gets installed and why:**

`blockchain/` (Hardhat / Solidity toolchain):
| Package | Role |
|---|---|
| `hardhat` | Local Ethereum node, compiler, test runner, deploy scripts |
| `@nomicfoundation/hardhat-toolbox` | Bundles ethers, chai matchers, gas reporter, Etherscan verification plugin |
| `@openzeppelin/contracts` | Battle-tested contract base classes: `ReentrancyGuard`, `Pausable`, `Ownable2Step`, ERC-20, `SafeERC20` |
| `dotenv` | Loads `.env` values into `hardhat.config.ts` |

`web/` (Next.js app):
| Package | Role |
|---|---|
| `next`, `react`, `react-dom` | App framework / UI runtime (Next.js 16, React 19) |
| `ethers` | Talks to MetaMask + the smart contracts from the browser and from server API routes |
| `@prisma/client`, `prisma` | Type-safe ORM + migrations against PostgreSQL |
| `bcryptjs` | Hashes user passwords (cost factor 12) |
| `jose` | Signs/verifies the HS256 session JWT stored in an httpOnly cookie |
| `@mui/material`, `@emotion/react`, `@emotion/styled` | Component library + its styling engine |
| `tailwindcss` v4, `@tailwindcss/postcss` | Utility CSS, wired through PostCSS (no separate `tailwind.config.js` — v4 configures via the PostCSS plugin) |
| `lenis` | Smooth-scroll library used on the marketing/home page |
| `dotenv` | Fallback `.env` loading for the Prisma client outside of Next's own env handling |

The `postinstall` script (`prisma generate`) runs automatically after `npm i` to
generate the Prisma client from `schema.prisma`.

### 1.3 Database setup (Supabase / PostgreSQL)

The project has **no Supabase SDK or Supabase-specific code** — it uses
**Prisma ORM** against any standard PostgreSQL database, and a hosted
**Supabase Postgres instance** is simply one convenient way to get that
database (Railway is another, referenced directly in `schema.prisma`'s
comments).

Steps:
1. Create a Postgres database (e.g. a new Supabase project, or `supabase start`
   locally if you have the Supabase CLI, or any Postgres instance).
2. Copy its connection string into `DATABASE_URL` in **`web/.env`**.
3. Apply the bundled migrations:

```bash
cd web
npx prisma migrate deploy   # applies the 3 existing migrations
```

During active schema development you would instead use:

```bash
npx prisma migrate dev      # create + apply a new migration
```

**Schema** (`web/prisma/schema.prisma`) — five models:

- **User** — `id`, `name`, `email` (unique), `password` (bcrypt hash, nullable for wallet-only accounts), `walletAddress` (unique, nullable for email-only accounts), `isAdmin`, relations to `BankAccount` and `BankTransfer[]`.
- **BankAccount** — one per user: `bankName`, `accountNumber`, `accountHolder`, `recipientAddress` (an Ethereum address MYR can be sent to on-chain).
- **BankTransfer** — simulated DuitNow disbursement record: `amountMYR`, `status` (`PENDING` → `PROCESSING` → `COMPLETED`), `referenceNo`, `bankName`, `accountLast4`.
- **LoanTransaction** — mirror of on-chain events (`type`, `amount`, `txHash`, `blockNumber`) kept as a fallback when reading events straight from the chain isn't possible.
- **KycSubmission** — full KYC application: personal/address/financial fields plus `icFrontData` / `icBackData` / `selfieData`, which store the uploaded ID photos as base64 data URIs directly in the row, and a `status` (`pending` / `approved`).

### 1.4 Hardhat local blockchain setup

The smart contracts run on a **local Hardhat node** that behaves like a real
Ethereum chain (chain ID `31337`).

```bash
cd blockchain

# Terminal A — start the local chain and keep it running
npm run chain
# (alias for `hardhat node`; prints 20 funded test accounts + private keys)

# Terminal B — compile & deploy the lending contract
npm run deploy:local
# runs scripts/deploy.ts against --network localhost

# optional — also deploy the ICO token-sale contracts
npm run deploy:ico
```

What `deploy:local` does (`blockchain/scripts/deploy.ts`):
1. Fetches the live ETH→MYR price from CoinGecko (falls back to RM 18,000 if the request fails).
2. Requires the deployer's nonce to be `0` (i.e. a freshly-started node) so the contract address is deterministic across restarts — override with `FORCE_DEPLOY=1` if you really want to redeploy on a dirty node.
3. Deploys `CryptoLoan`, which internally deploys its own `MockMYR` token.
4. **Auto-generates** `web/src/lib/contractConfig.ts` with the deployed addresses, chain ID, RPC URL, and full contract ABIs — the frontend picks this up with zero manual wiring.

`hardhat.config.ts` highlights:
- Solidity `0.8.24`, optimizer on (200 runs), `viaIR: true`.
- Networks: `hardhat` (in-process, 31337), `localhost` (`http://127.0.0.1:8545`, 31337), `sepolia` and `mainnet` (via Alchemy, using `ALCHEMY_API_KEY`).
- Etherscan verification wired for the two public networks.

If the Hardhat node is ever restarted, the contract must be redeployed — the
deploy script's nonce check exists precisely to keep the frontend's hardcoded
addresses valid.

### 1.5 MetaMask setup

1. Install the [MetaMask](https://metamask.io) browser extension.
2. Add a custom network:
   - **RPC URL**: `http://127.0.0.1:8545`
   - **Chain ID**: `31337`
   - **Currency symbol**: `ETH`
   (The app can also do this for you — `WalletContext.switchToHardhat()` calls `wallet_addEthereumChain` / `wallet_switchEthereumChain` automatically when it detects the wrong network.)
3. Import one of the private keys printed by `npm run chain` as a MetaMask account — these accounts start pre-funded with 10,000 test ETH.
4. Open the app and click **Connect Wallet**.
5. Optionally click **"+ Add MYR to MetaMask"** on the dashboard once you've borrowed — this calls `wallet_watchAsset` to import the `MockMYR` ERC-20 token so its balance is visible inside MetaMask (ERC-20 balances don't show up automatically).

### 1.6 Run the app

```bash
cd web
npm run dev
```

Open **http://localhost:3000**. Day-to-day, once contracts are deployed, you
only need the Hardhat node (`npm run chain`) running in one terminal and the
Next.js dev server (`npm run dev`) in another.

### 1.7 Testnet / mainnet deployment (optional)

With `OWNER_PRIVATE_KEY`, `ALCHEMY_API_KEY`, and `ETHERSCAN_API_KEY` set:

```bash
cd blockchain
npm run deploy:sepolia    # or deploy:mainnet
npm run verify:sepolia    # or verify:mainnet
```

---

## 2. File Structure

```
crypto-loan/
├── package.json                    # Root convenience scripts (proxy into web/ and blockchain/)
├── .env.example                    # Env var template (copy → .env at root AND in web/)
├── README.md
│
├── blockchain/                     # ── Solidity / Hardhat project ──────────────
│   ├── package.json
│   ├── hardhat.config.ts           # Networks, Solidity version, Etherscan, gas reporter
│   ├── contracts/
│   │   ├── CryptoLoan.sol          # Core lending protocol (deposit/borrow/repay/withdraw/buyMYR/liquidate)
│   │   ├── MockMYR.sol             # ERC-20 "Mock Malaysian Ringgit" (6 decimals), minted only by CryptoLoan
│   │   ├── MockUSDC.sol            # ERC-20 test token (not used by the main lending flow)
│   │   ├── ICO.sol                 # Sells RinggitToken for ETH at a fixed/admin-set price
│   │   ├── RinggitToken.sol        # ERC-20 "MYRC" token sold by the ICO
│   │   └── practice/               # Educational Solidity examples (unrelated to the app)
│   ├── scripts/
│   │   ├── deploy.ts               # Deploys CryptoLoan, writes web/src/lib/contractConfig.ts
│   │   ├── deploy-ico.ts           # Deploys RinggitToken + ICO, writes web/src/lib/icoConfig.ts
│   │   ├── set-price.ts            # Manually push a new on-chain ETH/MYR price
│   │   └── verify.ts               # Etherscan verification helper
│   └── test/
│       └── CryptoLoan.test.ts      # Full contract test suite (deposit, borrow, repay, liquidation, etc.)
│
└── web/                            # ── Next.js application ──────────────────────
    ├── package.json
    ├── next.config.ts              # transpilePackages: ['ethers']; outputFileTracingRoot fix
    ├── postcss.config.mjs          # Tailwind v4 PostCSS plugin
    ├── tsconfig.json               # `@/*` path alias → `./src/*`
    ├── prisma/
    │   ├── schema.prisma           # User, BankAccount, BankTransfer, LoanTransaction, KycSubmission
    │   └── migrations/             # 3 migrations (init, add docType, add isAdmin)
    └── src/
        ├── app/
        │   ├── layout.tsx          # Root layout: fonts, MUI theme provider, WalletProvider, smooth scroll
        │   ├── page.tsx            # Redirects to /home
        │   ├── home/               # Public marketing landing page
        │   ├── login/page.tsx      # Login (email/password + MetaMask)
        │   ├── signup/page.tsx     # Signup (email/password + MetaMask)
        │   ├── ico/page.tsx        # ICO token purchase page
        │   ├── (screen)/           # Route group sharing the authenticated AppShell layout
        │   │   ├── layout.tsx      # Navbar + collapsible sidebar wrapper
        │   │   ├── dashboard/      # Main app: stats, markets/calculator, Deposit/Withdraw/Borrow/Repay/Buy MYR dialog
        │   │   ├── markets/        # Sortable live market table
        │   │   ├── portfolio/      # User position, on-chain history, bank transfers
        │   │   ├── kyc/            # 5-step KYC submission wizard
        │   │   ├── settings/       # Bank account for MYR disbursement
        │   │   ├── admin/          # Admin-only KYC review queue
        │   │   └── docs/           # In-app documentation page
        │   └── api/
        │       ├── auth/           # signup, login, wallet-nonce, wallet-login, me, logout
        │       ├── kyc/            # KYC CRUD, approve, documents
        │       ├── prices/         # CoinGecko proxy (10 coins, MYR+USD, 15s cache)
        │       ├── admin/          # sync-price, sync-ico-price
        │       ├── transfers/      # Simulated bank transfer records
        │       ├── loan-tx/        # On-chain transaction mirror (DB fallback)
        │       └── profile/bank-account/  # Bank account CRUD
        ├── lib/
        │   ├── WalletContext.tsx   # Central React context: MetaMask connection + all contract calls
        │   ├── contractConfig.ts   # AUTO-GENERATED: CryptoLoan/MockMYR addresses + ABIs
        │   ├── icoConfig.ts        # AUTO-GENERATED: ICO/RinggitToken addresses + ABIs
        │   ├── auth-jwt.ts         # JWT sign/verify + cookie helpers (jose)
        │   ├── nonce-store.ts      # In-memory nonce store for wallet-signature login
        │   ├── theme.ts / tokens.ts # MUI theme + design tokens ("Passbook" light neobank palette)
        │   ├── db/prisma.ts        # Prisma client singleton
        │   └── kyc/chain.ts        # Server-side `setKYC()` caller (uses OWNER_PRIVATE_KEY)
        ├── hooks/
        │   ├── useAuth.ts          # Polls /api/auth/me — current user session
        │   ├── usePrices.ts        # Polls /api/prices every 15s — live market data
        │   └── useTransactionHistory.ts  # Reads on-chain events, falls back to /api/loan-tx
        └── components/
            └── sidebar/            # AppShell sidebar, nav items, ACL (auth/admin/KYC gating)
```

---

## 3. Feature Documentation

### 3.1 Deposit

Locks ETH as collateral against which the user can later borrow MYR.

**Contract** — `blockchain/contracts/CryptoLoan.sol`

```solidity
function depositCollateral() external payable whenNotPaused nonReentrant {
    require(msg.value > 0, "Send ETH");
    loans[msg.sender].collateral += msg.value;
    totalCollateral              += msg.value;
    emit CollateralDeposited(msg.sender, msg.value);
}
```
Any amount of ETH sent with the call is simply added to the caller's collateral
balance and to the protocol-wide `totalCollateral` counter. No KYC is required
to deposit — only to *borrow* against it.

**Frontend bridge** — `web/src/lib/WalletContext.tsx`

```typescript
const depositCollateral = useCallback(async (ethAmt: string) => {
  const c = await getContracts(true);           // signer-connected contract
  if (!c || !s.address) return;
  setTx('pending', `Depositing ${ethAmt} ETH…`);
  try {
    const tx = await c.loan.depositCollateral({ value: ethers.parseEther(ethAmt) });
    const receipt = await tx.wait();
    if (receipt && s.address) saveTxToDB(s.address, 'CollateralDeposited', ethers.parseEther(ethAmt).toString(), receipt);
    setTx('success', `Deposited ${ethAmt} ETH as collateral`);
    await refresh(s.address);                    // re-read balances/loan info from chain
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Deposit failed: ${reason}` : 'Deposit failed');
  }
}, [getContracts, s.address, refresh]);
```
`ethers.parseEther` converts the human-entered "1.5" ETH into wei before
sending it as `msg.value`. After the transaction confirms, the receipt is
persisted to the database (`POST /api/loan-tx`) as a durable transaction-history
fallback, and `refresh()` re-reads on-chain state so the UI updates instantly.

**UI** — `web/src/app/(screen)/dashboard/page.tsx` (Deposit tab inside the
Action Dialog): an ETH amount field with 25/50/75/MAX quick-fill buttons, a
live "max you could borrow at 70% LTV" preview, and a KYC gate
(`KycRequiredCard`) that blocks the form until the wallet's KYC is approved.

---

### 3.2 Borrow (Loan from Market)

Mints MYR to the user against their deposited ETH collateral, subject to a
70% max loan-to-value ratio and a KYC gate.

**Contract**

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

function _maxBorrow(uint256 collateralWei) internal view returns (uint256) {
    return (collateralWei * ethPrice * MAX_LTV * MYR_DECIMALS) / (PRECISION * 100);
}
```
The `onlyKYC` modifier (`require(kycApproved[msg.sender], "KYC required")`)
means an unverified wallet's transaction reverts before touching any state.
`_maxBorrow` converts the collateral (wei) into its MYR value at the current
on-chain `ethPrice`, then caps it at `MAX_LTV` (70%).

**Frontend bridge**

```typescript
const borrow = useCallback(async (myrAmt: string): Promise<boolean> => {
  const c = await getContracts(true);
  if (!c || !s.address) return false;
  setTx('pending', `Borrowing RM ${myrAmt}…`);
  try {
    const units = BigInt(Math.floor(parseFloat(myrAmt) * 1e6));  // MYR has 6 decimals
    const tx = await c.loan.borrow(units);
    const receipt = await tx.wait();
    if (receipt && s.address) saveTxToDB(s.address, 'Borrowed', units.toString(), receipt);
    setTx('success', `Borrowed RM ${myrAmt}`);
    await refresh(s.address);
    return true;
  } catch (e) {
    const reason = revertReason(e);
    const msg = reason === 'KYC required'
      ? 'On-chain KYC is out of sync — re-syncing. Try again in a moment.'
      : reason ? `Borrow failed: ${reason}` : 'Borrow failed — check LTV or collateral';
    setTx('error', msg);
    if (reason === 'KYC required' && s.address) void resyncKyc(s.address);  // self-heal
    return false;
  }
}, [getContracts, s.address, refresh, resyncKyc]);
```
Note the self-healing behavior: if the chain rejects the borrow because
on-chain KYC is missing — which can happen after a local Hardhat node is
redeployed and wipes contract state, even though the database still says
"approved" — the client automatically calls `resyncKyc()`, which asks the
server (holder of `OWNER_PRIVATE_KEY`) to re-run `setKYC()` on-chain.

**UI** — Borrow tab: loan-term selector, MYR amount input, a delivery-method
toggle (receive MYR as an ERC-20 token vs. simulate a bank transfer), and a
live breakdown of interest (4.8% APR), the 0.1% origination fee, monthly
payment, and projected health factor.

---

### 3.3 Withdraw

Returns previously-deposited ETH collateral, as long as any remaining debt
still satisfies the 70% max LTV.

**Contract**

```solidity
function withdrawCollateral(uint256 weiAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.collateral >= weiAmount, "Not enough collateral");

    uint256 remaining = loan.collateral - weiAmount;
    if (loan.principal > 0) {
        uint256 maxBorrow = _maxBorrow(remaining);
        require(loan.principal <= maxBorrow, "Would violate LTV");
    }

    loan.collateral -= weiAmount;
    totalCollateral  -= weiAmount;

    (bool ok, ) = payable(msg.sender).call{value: weiAmount}("");
    require(ok, "ETH transfer failed");
    emit CollateralWithdrawn(msg.sender, weiAmount);
}
```
If the user has an outstanding loan, the contract simulates the LTV *after*
the withdrawal and reverts with `"Would violate LTV"` if it would exceed 70%
— this prevents withdrawing collateral out from under an active loan.

**Frontend bridge**

```typescript
const withdrawCollateral = useCallback(async (ethAmt: string) => {
  const c = await getContracts(true);
  if (!c || !s.address) return;
  setTx('pending', `Withdrawing ${ethAmt} ETH…`);
  try {
    const tx = await c.loan.withdrawCollateral(ethers.parseEther(ethAmt));
    const receipt = await tx.wait();
    if (receipt && s.address) saveTxToDB(s.address, 'CollateralWithdrawn', ethers.parseEther(ethAmt).toString(), receipt);
    setTx('success', `Withdrawn ${ethAmt} ETH`);
    await refresh(s.address);
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Withdraw failed: ${reason}` : 'Withdraw failed — would violate LTV');
  }
}, [getContracts, s.address, refresh]);
```

**UI** — Withdraw tab computes the maximum safely-withdrawable amount
client-side before the user even submits, mirroring the contract's guard:

```typescript
const minColEth   = borMYR > 0 ? (borMYR / 0.70) / price : 0; // ETH needed to stay at 70% LTV
const maxWithdraw = Math.max(0, colEth - minColEth);
```

---

### 3.4 Repay

Pays down an outstanding MYR loan — interest first, then principal — and
requires the standard ERC-20 two-step "approve, then spend" pattern since the
contract must pull MYR from the caller.

**Contract**

```solidity
function repay(uint256 myrAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.principal > 0, "No active loan");
    require(myrAmount > 0, "Amount must be > 0");

    uint256 interest = accruedInterest(msg.sender);
    uint256 due      = loan.principal + interest;
    uint256 paying   = myrAmount > due ? due : myrAmount;   // never overpay

    myr.transferFrom(msg.sender, address(this), paying);

    uint256 interestPaid  = paying >= interest ? interest : paying;
    uint256 principalPaid = paying > interest ? paying - interest : 0;

    protocolFees  += interestPaid;
    totalBorrowed -= principalPaid;

    if (principalPaid >= loan.principal) {
        loan.principal = 0; loan.startTime = 0; loan.lastRepayTime = 0;  // loan fully closed
    } else {
        loan.principal    -= principalPaid;
        loan.lastRepayTime = block.timestamp;                            // interest clock resets
    }

    emit Repaid(msg.sender, principalPaid, interestPaid);
}

function accruedInterest(address user) public view returns (uint256) {
    Loan storage loan = loans[user];
    if (loan.principal == 0 || loan.startTime == 0) return 0;
    uint256 elapsed = block.timestamp - loan.lastRepayTime;
    return (loan.principal * BORROW_APR_BPS * elapsed) / (10_000 * 365 days);
}
```
Interest accrues linearly from `lastRepayTime` at a fixed 4.8% APR
(`BORROW_APR_BPS = 480`). Any payment is applied to interest first — the
remainder reduces principal. Note there is **no `onlyKYC` modifier** on
`repay` — a user must be able to pay off debt even if their KYC status
changed.

**Frontend bridge** — a two-step transaction (approve, then repay), reflected
in the UI's multi-step progress bar:

```typescript
const repay = useCallback(async (myrAmt: string) => {
  const c = await getContracts(true);
  if (!c || !s.address) return;
  setTx('pending', 'Approving MYR spend…', 1, 2);
  try {
    const units = BigInt(Math.floor(parseFloat(myrAmt) * 1e6));
    const approveTx = await c.myr.approve(CONTRACT_ADDRESSES.CryptoLoan, units);
    await approveTx.wait();
    setTx('pending', `Repaying RM ${myrAmt}…`, 2, 2);
    const repayTx = await c.loan.repay(units);
    const repayReceipt = await repayTx.wait();
    if (repayReceipt && s.address) saveTxToDB(s.address, 'Repaid', units.toString(), repayReceipt);
    setTx('success', `Repaid RM ${myrAmt}`, 2, 2);
    await refresh(s.address);
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Repay failed: ${reason}` : 'Repay failed — check MYR balance or approve amount', 1, 2);
  }
}, [getContracts, s.address, refresh]);
```

**UI** — Repay tab shows outstanding principal + live accrued interest + total
due, and a shortcut to the Buy MYR tab (pre-filled with the exact shortfall)
if the wallet's MYR balance is insufficient to cover the repayment.

---

### 3.5 Buy MYR

Lets a user swap ETH directly for MYR tokens at the contract's current
on-chain exchange rate — useful for topping up MYR to make a repayment
without needing to borrow more.

**Contract**

```solidity
function buyMYR(uint256 myrAmount) external payable whenNotPaused nonReentrant {
    require(myrAmount > 0, "Amount must be > 0");
    require(ethPrice > 0, "Price not set");

    // ETH needed = myrAmount / ethPrice, expressed in wei
    uint256 ethNeeded = (myrAmount * 1e18) / (ethPrice * MYR_DECIMALS);
    require(msg.value >= ethNeeded, "Insufficient ETH");

    myr.mint(msg.sender, myrAmount);

    uint256 excess = msg.value - ethNeeded;   // refund any ETH overpaid
    if (excess > 0) {
        payable(msg.sender).transfer(excess);
    }
    emit MYRPurchased(msg.sender, ethNeeded, myrAmount);
}
```
This is a straight mint-for-ETH swap, unrelated to the user's collateral/loan
position — it doesn't touch `loans[msg.sender]` at all. Any ETH sent beyond
what's needed is refunded automatically.

**Frontend bridge**

```typescript
const buyMYR = useCallback(async (myrAmt: string) => {
  const c = await getContracts(true);
  if (!c || !s.address) return;
  setTx('pending', `Buying RM ${myrAmt} of MYR…`);
  try {
    const units    = BigInt(Math.floor(parseFloat(myrAmt) * 1e6));
    const ethPrice = BigInt(s.ethPriceMYR);
    const ethNeeded = (units * BigInt(1e18)) / (ethPrice * BigInt(1e6));
    const ethWithBuffer = ethNeeded + ethNeeded / BigInt(1000);   // +0.1% buffer for rounding
    const tx = await c.loan.buyMYR(units, { value: ethWithBuffer });
    const receipt = await tx.wait();
    if (receipt && s.address) saveTxToDB(s.address, 'MYRPurchased', units.toString(), receipt);
    setTx('success', `Bought RM ${myrAmt} MYR`);
    await refresh(s.address);
  } catch (e) {
    const reason = revertReason(e);
    setTx('error', reason ? `Buy failed: ${reason}` : 'Buy MYR failed — check ETH balance');
  }
}, [getContracts, s.address, s.ethPriceMYR, refresh]);
```
The client adds a 0.1% ETH buffer on top of the computed `ethNeeded` to avoid
a revert from rounding dust — the contract refunds any excess anyway.

**UI** — Buy MYR tab: a MYR amount field, the live ETH/MYR exchange rate read
from the contract, the computed ETH cost, and a submit button.

---

### 3.6 KYC (Know Your Customer)

A database-backed identity-verification flow required (per Malaysian BNM
AML/CFT rules, as framed in the UI copy) before a wallet is allowed to borrow.
KYC has two tracks that are kept in sync: a **database record** (rich form
data + document images) and an **on-chain boolean flag** (`kycApproved` on the
`CryptoLoan` contract) that the smart contract actually checks.

**Contract (admin-only setter)**

```solidity
mapping(address => bool) public kycApproved;

function setKYC(address user, bool approved) external onlyOwner {
    require(user != address(0), "Zero address");
    kycApproved[user] = approved;
    emit KYCSet(user, approved);
}

modifier onlyKYC() {
    require(kycApproved[msg.sender], "KYC required");
    _;
}
```

**Server-side chain call** — `web/src/lib/kyc/chain.ts` uses the server-held
`OWNER_PRIVATE_KEY` so the *user* never pays gas or signs anything to get
approved:

```typescript
export async function setKycOnChain(wallet: string, approved = true): Promise<void> {
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const signer   = new ethers.Wallet(process.env.OWNER_PRIVATE_KEY!, provider);
  const contract = new ethers.Contract(LOAN_ADDR, SET_KYC_ABI, signer);
  const tx = await contract.setKYC(wallet, approved);
  await tx.wait();
}
```

**Submission flow** — `web/src/app/(screen)/kyc/page.tsx` is a 5-step wizard:

1. **Personal Info** — document type (MyKad/passport/license), full name, ID number, DOB, gender, nationality, phone, email.
2. **Address** — address lines, postcode, city, and a Malaysian-state dropdown.
3. **Financial Declaration** — employment status, income bracket, loan purpose, source of funds.
4. **Documents** — front/back ID photo upload, compressed client-side before upload:

```typescript
function compressImage(file: File, maxPx = 1200, quality = 0.82): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      const img = new Image();
      img.onload = () => {
        const scale = Math.min(1, maxPx / Math.max(img.width, img.height));
        const w = Math.round(img.width * scale), h = Math.round(img.height * scale);
        const canvas = document.createElement('canvas');
        canvas.width = w; canvas.height = h;
        canvas.getContext('2d')!.drawImage(img, 0, 0, w, h);
        resolve(canvas.toDataURL('image/jpeg', quality));   // → base64 data URI
      };
      img.src = reader.result as string;
    };
    reader.readAsDataURL(file);
  });
}
```
5. **Review & Submit** — summary of all entered fields plus Terms/Declaration
   checkboxes. Submitting `POST`s the whole form (including the compressed
   image data URIs) to `/api/kyc`, which upserts a `KycSubmission` row with
   `status: "pending"`.

**Admin approval** — `POST /api/kyc/approve` (called from the admin panel)
updates the DB row to `status: "approved"` and then calls `setKycOnChain()` to
flip the on-chain flag, unlocking `borrow()` for that wallet.

**Auto-resync** — `WalletContext.tsx` polls `/api/kyc?wallet=...` every 10
seconds. If the DB says `approved` but the contract's `kycApproved` mapping
says `false` (typically because a local Hardhat node was restarted and wiped
contract state), the client automatically re-triggers the on-chain approval
without the user having to resubmit anything:

```typescript
useEffect(() => {
  if (s.address && s.isCorrectNetwork && s.isDeployed &&
      s.kycApprovedDb && !s.kycApprovedChain && s.loanInfo) {
    void resyncKyc(s.address);
  }
}, [s.address, s.isCorrectNetwork, s.isDeployed, s.kycApprovedDb, s.kycApprovedChain, s.loanInfo, resyncKyc]);
```

**UI states**: the KYC page renders one of four screens depending on status —
the multi-step form itself, a "Pending Review" screen (with a `KYC-000123`
style reference number), an "Already Verified" success screen, or (elsewhere
in the app) a `KycRequiredCard` gate shown in place of the Deposit/Borrow/Repay
forms until the wallet is approved.

---

### 3.7 Login

Supports two independent sign-in methods that both end in the same JWT
session cookie.

**Email/password** — `POST /api/auth/login`:
```typescript
const user = await prisma.user.findUnique({ where: { email } });
const valid = await bcrypt.compare(password, user.password);
const token = await createToken({ id: user.id, email: user.email, name: user.name, isAdmin });
await setAuthCookie(token);
```

**MetaMask wallet login** — a SIWE-style nonce/sign challenge:
1. `GET /api/auth/wallet-nonce?address=...` generates a random nonce (in-memory, 5-minute expiry).
2. The client asks MetaMask to `personal_sign` the message `Sign in to CryptoLend\nNonce: <nonce>`.
3. `POST /api/auth/wallet-login` verifies the signature and issues the session cookie:

```typescript
const message = `Sign in to CryptoLend\nNonce: ${nonce}`;
const recovered = ethers.verifyMessage(message, signature);
if (recovered.toLowerCase() !== address.toLowerCase()) {
  return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
}
let user = await prisma.user.findUnique({ where: { walletAddress: address } });
if (!user) {
  user = await prisma.user.create({ data: { walletAddress: address, name: `${address.slice(0,6)}...${address.slice(-4)}` } });
}
```
A brand-new wallet address is silently registered as a new user on first
sign-in — there's no separate "connect wallet to an existing account" step.

**Client** — `web/src/app/login/page.tsx`:
```typescript
const handleWalletLogin = async () => {
  const [address] = await eth.request({ method: 'eth_requestAccounts' });
  const { nonce } = await fetch(`/api/auth/wallet-nonce?address=${address}`).then(r => r.json());
  const message = `Sign in to CryptoLend\nNonce: ${nonce}`;
  const signature = await eth.request({ method: 'personal_sign', params: [message, address] });
  const res = await fetch('/api/auth/wallet-login', {
    method: 'POST', headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ address, signature, nonce }),
  });
  ...
  router.push(nextPath);
};
```
Admins are redirected to `/admin`; everyone else goes to `?next=` or
`/dashboard`. Route protection is enforced centrally in
`web/src/proxy.ts` (Next.js middleware), which redirects unauthenticated
visitors to `/login` for every route except `/`, `/home`, `/login`, and
`/signup`.

---

### 3.8 Sign Up

Mirrors the login page's two methods.

**Email/password** — `POST /api/auth/signup`:
```typescript
const hash = await bcrypt.hash(password, 12);
const user = await prisma.user.create({ data: { name: name?.trim() || null, email, password: hash } });
const token = await createToken({ id: user.id, email: user.email, name: user.name });
await setAuthCookie(token);
```

**Client-side password-strength meter** — `web/src/app/signup/page.tsx`:
```typescript
function passwordStrength(pwd: string): { score: number; label: string; color: string } {
  let score = 0;
  if (pwd.length >= 8)  score++;
  if (pwd.length >= 12) score++;
  if (/[A-Z]/.test(pwd)) score++;
  if (/[0-9]/.test(pwd)) score++;
  if (/[^A-Za-z0-9]/.test(pwd)) score++;
  const map = [
    { label: '', color: '#E2E7EE' }, { label: 'Very weak', color: '#ef4444' },
    { label: 'Weak', color: '#f97316' }, { label: 'Fair', color: '#eab308' },
    { label: 'Strong', color: '#22c55e' }, { label: 'Very strong', color: '#2A3FD6' },
  ];
  return { score, ...map[score] };
}
```
The submit button is disabled if `password !== confirm` or the strength score
is too low. **MetaMask signup** uses the exact same nonce/sign/verify flow as
wallet login (see 3.7) — `wallet-login` doubles as signup since it
auto-creates the user record on first appearance of a wallet address.

---

### 3.9 Dashboard

The main authenticated landing page (`web/src/app/(screen)/dashboard/page.tsx`,
~2000 lines) — home base for every lending action.

Contents:
- **Protocol stats banner** — Total Value Locked, Active Loans, Total
  Borrowed, and the live ETH/MYR price (from CoinGecko via `/api/prices`).
- **User stat cards** — My Collateral, Outstanding Debt, Net Position, Health
  Factor, each computed from live on-chain reads once a wallet is connected,
  or shown as illustrative demo data otherwise.
- **Price-mismatch banner** — appears when the contract's stored `ethPrice`
  drifts more than 3% from the live market price, with a one-click **"⟳ Sync
  Price"** button that calls `POST /api/admin/sync-price` (see below).
- **Markets & Calculator panel** — a clickable asset table plus an
  interactive LTV/loan-term/"hold vs. sell" calculator.
- **Action Dialog** — a single modal with five inline tabs — **Deposit,
  Withdraw, Borrow, Repay, Buy MYR** — opened by clicking a stat card or by
  navigating to `/dashboard?tab=<name>`, which the page syncs against
  component state:

```typescript
useEffect(() => {
  const tab = searchParams.get('tab');
  if (tab === 'deposit' || tab === 'withdraw' || tab === 'borrow' || tab === 'repay' || tab === 'buy') {
    setActiveTab(tab);
  } else {
    setActiveTab(null);
  }
}, [searchParams]);
```

**Live vs. demo mode**: `isLive = wallet.isConnected && wallet.isCorrectNetwork && wallet.isDeployed`
gates whether real on-chain numbers or illustrative placeholder numbers are
shown, so the dashboard is still browsable before connecting a wallet.

**Price sync API** — `POST /api/admin/sync-price` fetches the live ETH/MYR
rate and nudges the on-chain price toward it in ≤20% steps per call (the
contract's `setEthPrice` guard rejects any single jump larger than 20%).

---

### 3.10 Market

`web/src/app/(screen)/markets/page.tsx` — a searchable, sortable table of ten
crypto assets (BTC, ETH, SOL, BNB, XRP, AVAX, LINK, DOT, ADA, MATIC).

```typescript
const MARKETS = [
  { symbol: 'BTC', name: 'Bitcoin', icon: '₿', color: '#F7931A', id: 'bitcoin',
    supplyAPR: 2.1, borrowAPR: 5.2, maxLTV: 70, liqThresh: 80,
    liquidity: 'RM 11.2B', totalBorrowed: 'RM 7.8B', util: 70 },
  // … ETH, SOL, BNB, XRP, AVAX, LINK, DOT, ADA, MATIC
];
```
Each row shows the live MYR price and 24h change (from `usePrices()`), Max
LTV, Liquidation Threshold, Borrow/Supply APR, and a utilization bar. Columns
are click-to-sort:
```typescript
const handleSort = (key: SortKey) => {
  if (sortKey === key) setSortDir(d => d === 'asc' ? 'desc' : 'asc');
  else { setSortKey(key); setSortDir('desc'); }
};
```
Only ETH is actually borrowable against on this deployment (it's the only
collateral type `CryptoLoan.sol` supports) — the other assets are shown for
market-browsing context and their "Borrow" buttons route to
`/dashboard?tab=borrow&asset=SYMBOL`.

**Prices API** — `GET /api/prices` proxies CoinGecko for all ten coins in MYR
and USD with 24h change, cached at the edge for 15 seconds. **`usePrices()`**
polls this endpoint every 15 seconds and flags each price with a brief
up/down "flash" class for the CSS flash animation.

---

### 3.11 Portfolio

`web/src/app/(screen)/portfolio/page.tsx` — the user's personal position and
history.

- **Position summary** — collateral (ETH + MYR value), outstanding debt,
  accrued interest, current LTV, and health factor, sourced from
  `wallet.loanInfo`.
- **Transaction history** — `useTransactionHistory(address)` reads on-chain
  events directly:

```typescript
const events = await contract.queryFilter(contract.filters.Borrowed(address));
// … plus CollateralDeposited, CollateralWithdrawn, Repaid, MYRPurchased
```
  and falls back to the database-mirrored history at `GET /api/loan-tx` if a
  direct chain query isn't available (e.g. an older Hardhat node without full
  event-log retention).
- **Bank transfers** — polls `GET /api/transfers` every 5 seconds to reflect
  the simulated DuitNow disbursement status as it moves from `PENDING` →
  `PROCESSING` → `COMPLETED`.
- **Wallet balances** — live ETH and MockMYR balances, with the same "Add MYR
  to MetaMask" shortcut used on the dashboard.

---

### 3.12 Settings

`web/src/app/(screen)/settings/page.tsx` — manages the bank account MYR is
disbursed to when a user chooses "Bank Transfer" instead of receiving the
raw MYR token.

```typescript
const BANKS = [
  'Maybank', 'CIMB Bank', 'Public Bank', 'RHB Bank', 'Hong Leong Bank',
  'AmBank', 'Bank Islam', 'Bank Muamalat', 'Bank Rakyat', 'Affin Bank',
  'Alliance Bank', 'HSBC Bank Malaysia', 'OCBC Bank Malaysia',
  'Standard Chartered', 'UOB Malaysia', 'BSN (Bank Simpanan Nasional)',
];
```
The form collects `bankName`, `accountNumber`, `accountHolder`, and an
on-chain `recipientAddress` (with quick-fill buttons for the standard Hardhat
test accounts, for local development convenience). A "passbook" preview card
renders the saved account with the number masked to its last 4 digits:
```typescript
function maskAccount(s: string) {
  const last4 = s.slice(-4);
  return groupDigits('•'.repeat(Math.max(0, s.length - 4)) + last4);
}
```
Persisted via `GET`/`PUT /api/profile/bank-account`, which validates a 6–20
digit account number and a well-formed Ethereum address for the recipient.

---

### 3.13 Docs

`web/src/app/(screen)/docs/page.tsx` — an in-app static documentation page
with a sticky section-jump sidebar covering: **Overview**, **How It Works**
(the 5-step lifecycle: connect → KYC → deposit → borrow → repay), **KYC
Verification**, **Collateral & LTV** (with the exact borrow-limit formula),
**Health Factor** (safe ≥ 2.0, moderate 1.5–2.0, at-risk < 1.5, liquidatable <
1.0), **Buy MYR**, **Interest & Fees** (4.80% APR / 0.10% origination / 5%
liquidation bonus), and an **FAQ**. It is pure presentational content — no
API calls — built from small reusable building blocks (`Section`, `P`,
`Strong`, `InlineCode`).

---

## 4. Reference: Core Lending Parameters

These constants, defined in `CryptoLoan.sol`, govern every feature above:

| Constant | Value | Meaning |
|---|---|---|
| `MAX_LTV` | 70% | Maximum loan-to-value a borrow/withdraw may reach |
| `LIQ_THRESHOLD` | 80% | LTV at which a position becomes liquidatable |
| `LIQ_BONUS` | 5% | Bonus collateral awarded to a liquidator |
| `BORROW_APR_BPS` | 480 (4.8%) | Annual interest rate charged on borrowed MYR |
| `MAX_PRICE_CHANGE` | 20% | Max allowed single-call move in the admin-set ETH/MYR price |
| `MYR_DECIMALS` | 1e6 | MockMYR uses 6 decimal places (like real-world stablecoins) |
