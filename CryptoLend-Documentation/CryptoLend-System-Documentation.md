# CryptoLend — Blockchain-Based Crypto-Collateralized MYR Lending Platform

```
CryptoLend — System Documentation
│
├── 1. Introduction
│   ├── 1.1 Project Overview
│   ├── 1.2 Objectives
│   └── 1.3 System Architecture
│
├── 2. Technology Stack
│   ├── Frontend
│   ├── Backend
│   ├── Blockchain
│   ├── Database
│   └── Development Tools
│
├── 3. Prerequisites
│   ├── Required Software
│   └── Minimum Hardware Requirements
│
├── 4. Project Structure
│   ├── Folder Structure
│   └── Description of Each Folder
│
├── 5. Installation Guide
│   ├── Clone Repository
│   ├── Install Dependencies
│   ├── Configure Environment Variables
│   ├── Database Setup
│   ├── Smart Contract Deployment
│   ├── Start Backend Server
│   └── Start Frontend
│
├── 6. Environment Variables
│   ├── Root / Hardhat .env
│   └── Web (Frontend + API) .env
│
├── 7. Running the System
│   ├── Run Local Blockchain
│   ├── Deploy Smart Contracts
│   ├── Run Backend (API Routes)
│   ├── Run Frontend
│   └── Test User Accounts
│
├── 8. System Features
│   ├── 8.1 User Authentication
│   ├── 8.2 Dashboard
│   ├── 8.3 Deposit Collateral
│   ├── 8.4 Borrow (Loan from Market)
│   ├── 8.5 Withdraw Collateral
│   ├── 8.6 Repay Loan
│   ├── 8.7 Buy MYR
│   ├── 8.8 KYC Identity Verification
│   ├── 8.9 Tamper-Proof Audit Trail
│   ├── 8.10 Liquidation & Risk Controls
│   └── 8.11 Admin Management
│
├── 9. Smart Contract
│   ├── Contract Overview
│   ├── Functions
│   ├── Events
│   └── Contract Address
│
├── 10. Database Design
│   ├── ER Diagram
│   ├── Tables
│   └── Relationships
│
├── 11. API Documentation
│   ├── Authentication APIs
│   ├── Lending / Wallet APIs
│   ├── KYC APIs
│   └── Supporting APIs
│
├── 12. User Guide
│   ├── Login
│   ├── Submit KYC
│   ├── Borrow / Repay
│   └── View Blockchain Records
│
├── 13. Troubleshooting
│   ├── Common Errors
│   └── Solutions
│
└── 14. Future Improvements
```

---

## 1. Introduction

### 1.1 Project Overview

CryptoLend is a blockchain-based lending platform that lets a user lock
Ethereum (ETH) as collateral and borrow Malaysian Ringgit (MYR) against it,
without selling the underlying asset. Every custody-sensitive action —
depositing collateral, borrowing, repaying, withdrawing, and buying MYR — is
executed and permanently recorded by a Solidity smart contract on an Ethereum
network (a local Hardhat chain for development, with Sepolia/Mainnet
deployment scripts provided). A Next.js web application provides the user
interface, account/session management, and a database-backed KYC (Know Your
Customer) workflow that gates borrowing until a wallet is verified.

### 1.2 Objectives

- Let users unlock spending liquidity from crypto holdings without triggering a taxable/irreversible sale.
- Enforce loan-to-value (LTV), interest accrual, and liquidation rules transparently and immutably on-chain, rather than in a mutable off-chain database.
- Gate borrowing behind a KYC identity-verification process, satisfying AML/CFT-style compliance expectations (modelled on Malaysian BNM guidance).
- Provide a single dashboard where a user can deposit, borrow, repay, withdraw, and buy MYR, and see live market prices and their position's health in real time.
- Keep a durable, queryable transaction history that is sourced from on-chain events but backed by a database mirror for resilience.

### 1.3 System Architecture

```
┌───────────────────────┐        ┌──────────────────────────┐
│   Browser (React UI)   │        │        MetaMask           │
│  Next.js App Router    │◄──────►│  (signs & sends txs)      │
│  MUI + Tailwind         │        └────────────┬─────────────┘
└───────────┬────────────┘                     │  JSON-RPC
            │  fetch()                          ▼
            │                        ┌──────────────────────────┐
            ▼                        │   Hardhat Local Chain     │
┌───────────────────────┐            │   (or Sepolia / Mainnet)  │
│ Next.js API Routes      │           │                            │
│ (app/api/**/route.ts)  │──ethers──►│   CryptoLoan.sol contract │
│  - auth (JWT/bcrypt)   │  (owner    │   MockMYR.sol (ERC-20)    │
│  - KYC submission       │   key,     └──────────────────────────┘
│  - price proxy          │   server-
│  - transfer simulation  │   side)
└───────────┬────────────┘
            │ Prisma ORM
            ▼
┌───────────────────────┐
│   PostgreSQL Database  │
│ (e.g. Supabase/Railway)│
│  User, KycSubmission,   │
│  BankAccount/Transfer,  │
│  LoanTransaction        │
└───────────────────────┘
```

Three cooperating layers:
1. **Client** — React components call MetaMask directly via `ethers.js` for
   every fund-moving action (deposit/borrow/repay/withdraw/buy), and call the
   app's own API routes for everything else (auth, KYC data, prices, bank
   transfer simulation).
2. **Server (API routes)** — stateless Next.js route handlers. The only place
   the server itself touches the blockchain is the KYC approval endpoint,
   which signs a `setKYC()` transaction with a server-held owner private key
   so the **user** never pays gas to get approved.
3. **Blockchain** — the source of truth for collateral, debt, interest, and
   KYC approval. The database is a convenience mirror (rich KYC form data,
   transaction history fallback, bank account details) — it is never the
   authority on loan balances.

---

## 2. Technology Stack

### Frontend
| Technology | Purpose |
|---|---|
| Next.js 16 (App Router) | Routing, server components, API routes |
| React 19 | UI rendering |
| MUI (Material UI) v9 | Component library (forms, dialogs, tables, steppers) |
| Tailwind CSS v4 | Utility CSS, wired through the PostCSS plugin (`@tailwindcss/postcss`) |
| Emotion (`@emotion/react`, `@emotion/styled`) | CSS-in-JS engine that MUI is built on |
| ethers.js v6 | Browser-side contract calls, wallet signature requests |
| lenis | Smooth-scroll on the marketing/home page |

### Backend
| Technology | Purpose |
|---|---|
| Next.js API Routes (`app/api/**/route.ts`) | REST-style endpoints — no separate backend service |
| jose | Signs/verifies the HS256 session JWT |
| bcryptjs | Password hashing (cost factor 12) |
| Prisma Client | Type-safe database access from route handlers |

### Blockchain
| Technology | Purpose |
|---|---|
| Solidity 0.8.24 | Smart contract language |
| Hardhat 2.22 | Local chain, compiler, test runner, deploy/verify scripts |
| OpenZeppelin Contracts v5 | `ReentrancyGuard`, `Pausable`, `Ownable2Step`, `SafeERC20`, ERC-20 base |
| ethers.js v6 | Contract deployment scripts, server-side owner-key transactions |

### Database
| Technology | Purpose |
|---|---|
| PostgreSQL | Relational data store (hosted — e.g. Supabase or Railway Postgres) |
| Prisma ORM | Schema (`schema.prisma`), migrations, typed client generation |

### Development Tools
| Tool | Purpose |
|---|---|
| TypeScript 5 | Static typing across frontend, API routes, and Hardhat scripts |
| ESLint 9 | Linting (`eslint-config-next`) |
| Hardhat Toolbox | Bundles ethers, chai matchers, gas reporter, Etherscan plugin |
| npm | Package manager for both sub-projects |
| nvm / `.nvmrc` (Node 20) | Pinned Node runtime version |

---

## 3. Prerequisites

### Required Software
- **Node.js 20+** (pinned via `.nvmrc`; `nvm use` if available)
- **npm** (ships with Node)
- **MetaMask** browser extension (Chrome, Firefox, Brave, or Edge)
- **PostgreSQL database** — local install, or a hosted instance (Supabase, Railway, etc.)
- **Git** to clone the repository
- (Optional, for public-network deploys) an **Alchemy** account and an **Etherscan** account for RPC access and contract verification

### Minimum Hardware Requirements
| Component | Minimum | Recommended |
|---|---|---|
| CPU | Dual-core 2 GHz | Quad-core 2.5 GHz+ |
| RAM | 4 GB | 8 GB+ |
| Disk | 2 GB free | 5 GB+ free (node_modules + Hardhat artifacts) |
| OS | Windows 10 / macOS 12 / Linux (any modern distro) | — |
| Network | Broadband internet (for CoinGecko price feed, npm installs, RPC calls) | — |

---

## 4. Project Structure

### Folder Structure

```
crypto-loan/
├── package.json                 # Root convenience scripts
├── .env.example                 # Env template (copy to .env at root AND in web/)
├── README.md
│
├── blockchain/                  # Solidity contracts + Hardhat project
│   ├── contracts/
│   │   ├── CryptoLoan.sol       # Core lending protocol
│   │   ├── MockMYR.sol          # ERC-20 MYR token (6 decimals)
│   │   ├── MockUSDC.sol         # Unused test token
│   │   ├── ICO.sol              # Token-sale contract
│   │   ├── RinggitToken.sol     # ERC-20 "MYRC" sold by the ICO
│   │   └── practice/            # Educational Solidity examples
│   ├── scripts/
│   │   ├── deploy.ts            # Deploys CryptoLoan, writes contractConfig.ts
│   │   ├── deploy-ico.ts        # Deploys ICO, writes icoConfig.ts
│   │   ├── set-price.ts         # Manual ETH/MYR price push
│   │   └── verify.ts            # Etherscan verification
│   ├── test/
│   │   └── CryptoLoan.test.ts   # Full contract test suite
│   └── hardhat.config.ts
│
└── web/                          # Next.js application
    ├── prisma/
    │   ├── schema.prisma          # Data models
    │   └── migrations/            # 3 migrations
    └── src/
        ├── app/
        │   ├── layout.tsx          # Root layout (theme, wallet provider)
        │   ├── home/                # Public marketing page
        │   ├── login/, signup/     # Auth pages
        │   ├── ico/                # ICO purchase page
        │   ├── (screen)/           # Authenticated app shell
        │   │   ├── dashboard/
        │   │   ├── markets/
        │   │   ├── portfolio/
        │   │   ├── kyc/
        │   │   ├── settings/
        │   │   ├── admin/
        │   │   └── docs/
        │   └── api/                 # All backend endpoints
        │       ├── auth/
        │       ├── kyc/
        │       ├── prices/
        │       ├── admin/
        │       ├── transfers/
        │       ├── loan-tx/
        │       └── profile/bank-account/
        ├── lib/
        │   ├── WalletContext.tsx    # Wallet state + all contract calls
        │   ├── contractConfig.ts    # AUTO-GENERATED addresses + ABIs
        │   ├── auth-jwt.ts           # JWT + cookie helpers
        │   ├── nonce-store.ts        # In-memory nonce store
        │   ├── db/prisma.ts          # Prisma client singleton
        │   └── kyc/chain.ts          # Server-side setKYC() caller
        ├── hooks/
        │   ├── useAuth.ts
        │   ├── usePrices.ts
        │   └── useTransactionHistory.ts
        └── components/sidebar/       # AppShell nav + access control
```

### Description of Each Folder

| Folder | Description |
|---|---|
| `blockchain/contracts/` | All Solidity source files. `CryptoLoan.sol` is the only contract the web app talks to for lending; the others (`ICO.sol`, `RinggitToken.sol`) support a separate token-sale feature, and `practice/` is unrelated teaching material. |
| `blockchain/scripts/` | TypeScript deploy/verify/utility scripts run via `hardhat run`. `deploy.ts` is the most important — it also regenerates the frontend's contract config. |
| `blockchain/test/` | Hardhat/Chai test suite exercising every contract function and revert path. |
| `web/prisma/` | Database schema and migration history, applied with `npx prisma migrate deploy`. |
| `web/src/app/(screen)/` | A Next.js *route group* — pages inside share one authenticated layout (sidebar + navbar) without the `(screen)` segment appearing in the URL. |
| `web/src/app/api/` | All server-side logic. Each `route.ts` exports `GET`/`POST`/`PUT`/`DELETE` handlers — this *is* the backend, there is no separate server process. |
| `web/src/lib/` | Shared, non-UI logic: wallet/contract bridge, auth, database client, design tokens. |
| `web/src/hooks/` | Reusable React hooks that wrap polling/fetch logic for auth, prices, and transaction history. |
| `web/src/components/sidebar/` | Navigation shell and its access-control rules (which links require auth, admin, or KYC). |

---

## 5. Installation Guide

### Clone Repository
```bash
git clone <repository-url> crypto-loan
cd crypto-loan
```

### Install Dependencies
The two sub-projects are installed independently:
```bash
cd blockchain && npm i
cd ../web && npm i
```
`web`'s `postinstall` script runs `prisma generate` automatically, so the
Prisma client is ready immediately after install.

### Configure Environment Variables
```bash
cp .env.example .env          # read by blockchain/hardhat.config.ts
cp .env.example web/.env      # read by Next.js / Prisma
```
These are two independent physical files — see Section 6 for what to fill in.

### Database Setup
```bash
cd web
npx prisma migrate deploy     # applies the 3 bundled migrations
```
Point `DATABASE_URL` (in `web/.env`) at any PostgreSQL instance — a hosted
Supabase project, Railway Postgres, or a local Postgres server all work
identically since the app only talks to Prisma, never to a
provider-specific SDK.

### Smart Contract Deployment
```bash
cd blockchain
npm run chain            # Terminal A — keep running
npm run deploy:local     # Terminal B — deploys CryptoLoan + MockMYR
npm run deploy:ico       # optional — deploys the ICO contracts
```
`deploy:local` auto-writes `web/src/lib/contractConfig.ts` with the deployed
addresses and ABIs — no manual copy-pasting required.

### Start Backend Server
There is no separate backend process — the "backend" is the set of API
routes bundled into the Next.js app and served by the same dev server
started below.

### Start Frontend
```bash
cd web
npm run dev
```
Open **http://localhost:3000**.

---

## 6. Environment Variables

### Root / Hardhat `.env`
(`crypto-loan/.env` — read by `blockchain/hardhat.config.ts`)

| Variable | Purpose |
|---|---|
| `OWNER_PRIVATE_KEY` | Deploys/owns the contracts; on Hardhat local, account #0's key (printed by `npm run chain`) |
| `HARDHAT_RPC_URL` | RPC endpoint; defaults to `http://127.0.0.1:8545` |
| `ALCHEMY_API_KEY` | RPC provider for Sepolia/Mainnet deploys |
| `ETHERSCAN_API_KEY` | Contract verification on public networks |
| `REPORT_GAS` | `true` to print gas usage during `npm test` |

### Web (Frontend + API) `.env`
(`crypto-loan/web/.env` — read by Next.js and Prisma)

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string, e.g. `postgresql://postgres:postgres@localhost:5432/cryptoloan` |
| `OWNER_PRIVATE_KEY` | Same owner key, used server-side by API routes to call `setKYC()` and sync the on-chain ETH price — the end user never pays this gas |
| `HARDHAT_RPC_URL` | Blockchain RPC the API routes and browser client connect to |
| `JWT_SECRET` | Signs the session cookie; optional locally (dev fallback), **required** in production |

> Both files must be kept in sync manually — they are independent copies of the same template, not symlinks.

---

## 7. Running the System

### Run Local Blockchain
```bash
cd blockchain
npm run chain
```
Starts an in-process Ethereum node (chain ID `31337`) with 20 pre-funded test
accounts; their private keys are printed to the terminal.

### Deploy Smart Contracts
```bash
npm run deploy:local
```
Requires the deployer's nonce to be `0` (a freshly started node) so the
deployed address stays deterministic — pass `FORCE_DEPLOY=1` to override on a
dirty node. Regenerates `web/src/lib/contractConfig.ts`.

### Run Backend (API Routes)
The API routes are served automatically by the Next.js dev server — see
"Run Frontend" below. There is no separate `npm run backend` command.

### Run Frontend
```bash
cd web
npm run dev
```
Serves the UI **and** all `/api/*` routes at `http://localhost:3000`.

### Test User Accounts
1. In MetaMask, add a network: RPC `http://127.0.0.1:8545`, Chain ID `31337`.
2. Import any private key printed by `npm run chain` — each account starts with 10,000 test ETH.
3. Click **Connect Wallet** in the app.
4. To test the email/password path instead, use the Sign Up page to create an account with any email/password (no real email delivery occurs).
5. To act as an admin, manually set `isAdmin = true` on a `User` row in the database (there is no self-service admin signup).

---

## 8. System Features

### 8.1 User Authentication
Two independent, interchangeable sign-in methods, both ending in the same
JWT session cookie (`auth-token`, httpOnly, 7-day expiry):

- **Email/password** — `bcrypt`-hashed (cost 12), verified in `/api/auth/login`.
- **MetaMask wallet (SIWE-style)** — the client requests a one-time nonce,
  signs `Sign in to CryptoLend\nNonce: <nonce>` with `personal_sign`, and the
  server verifies the recovered address with `ethers.verifyMessage`:

```typescript
// web/src/app/api/auth/wallet-login/route.ts
const message = `Sign in to CryptoLend\nNonce: ${nonce}`;
const recovered = verifyMessage(message, signature);
if (recovered.toLowerCase() !== address.toLowerCase()) {
  return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
}
let user = await prisma.user.findUnique({ where: { walletAddress: address } });
if (!user) {
  user = await prisma.user.create({
    data: { walletAddress: address, name: `${address.slice(0, 6)}…${address.slice(-4)}` },
  });
}
```
A wallet address is auto-registered as a new user the first time it signs
in — there's no separate "link wallet to existing account" step. Route
protection is centralized in `web/src/proxy.ts` middleware, which redirects
unauthenticated visitors to `/login` for every route except `/`, `/home`,
`/login`, and `/signup`.

### 8.2 Dashboard
`web/src/app/(screen)/dashboard/page.tsx` is the authenticated home base:
protocol-wide stats (TVL, active loans, ETH/MYR price), the user's own
position (collateral, debt, net position, health factor), a live market +
loan calculator, and a single Action Dialog with five tabs — Deposit,
Withdraw, Borrow, Repay, Buy MYR — driven by the URL (`/dashboard?tab=borrow`).
A `isLive` flag toggles between real on-chain numbers and illustrative demo
data so the page is still browsable before a wallet connects.

### 8.3 Deposit Collateral
Locks ETH against which the user can later borrow.

```solidity
// blockchain/contracts/CryptoLoan.sol
function depositCollateral() external payable whenNotPaused nonReentrant {
    require(msg.value > 0, "Send ETH");
    loans[msg.sender].collateral += msg.value;
    totalCollateral              += msg.value;
    emit CollateralDeposited(msg.sender, msg.value);
}
```
```typescript
// web/src/lib/WalletContext.tsx
const depositCollateral = useCallback(async (ethAmt: string) => {
  const c = await getContracts(true);
  const tx = await c.loan.depositCollateral({ value: ethers.parseEther(ethAmt) });
  const receipt = await tx.wait();
  saveTxToDB(s.address!, 'CollateralDeposited', ethers.parseEther(ethAmt).toString(), receipt);
  await refresh(s.address!);
}, [getContracts, s.address, refresh]);
```
No KYC is required to deposit — only to borrow against it.

### 8.4 Borrow (Loan from Market)
Mints MYR against deposited collateral, capped at 70% LTV, gated by KYC.

```solidity
function borrow(uint256 myrAmount) external whenNotPaused nonReentrant onlyKYC {
    Loan storage loan = loans[msg.sender];
    require(loan.collateral > 0, "Deposit collateral first");
    uint256 maxBorrow = _maxBorrow(loan.collateral);
    require(loan.principal + myrAmount <= maxBorrow, "Exceeds max LTV");
    if (loan.startTime == 0) { loan.startTime = block.timestamp; loan.lastRepayTime = block.timestamp; }
    loan.principal += myrAmount;
    totalBorrowed  += myrAmount;
    myr.mint(msg.sender, myrAmount);
    emit Borrowed(msg.sender, myrAmount, loan.principal);
}
```
If a borrow reverts with `"KYC required"` — typically after a local chain
redeploy wipes on-chain state even though the database still shows
"approved" — the client self-heals by calling `resyncKyc()`, which re-runs
`setKYC()` from the server using the owner key.

### 8.5 Withdraw Collateral
Returns ETH collateral, but only if any remaining debt still satisfies 70%
LTV after the withdrawal:

```solidity
function withdrawCollateral(uint256 weiAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.collateral >= weiAmount, "Not enough collateral");
    uint256 remaining = loan.collateral - weiAmount;
    if (loan.principal > 0) {
        require(loan.principal <= _maxBorrow(remaining), "Would violate LTV");
    }
    loan.collateral -= weiAmount;
    totalCollateral  -= weiAmount;
    (bool ok, ) = payable(msg.sender).call{value: weiAmount}("");
    require(ok, "ETH transfer failed");
    emit CollateralWithdrawn(msg.sender, weiAmount);
}
```

### 8.6 Repay Loan
Interest is paid first, then principal; requires an ERC-20 `approve` step
before the contract can pull MYR from the caller:

```solidity
function repay(uint256 myrAmount) external whenNotPaused nonReentrant {
    Loan storage loan = loans[msg.sender];
    require(loan.principal > 0, "No active loan");
    uint256 interest = accruedInterest(msg.sender);
    uint256 due      = loan.principal + interest;
    uint256 paying   = myrAmount > due ? due : myrAmount;
    myr.transferFrom(msg.sender, address(this), paying);
    uint256 interestPaid  = paying >= interest ? interest : paying;
    uint256 principalPaid = paying > interest ? paying - interest : 0;
    protocolFees  += interestPaid;
    totalBorrowed -= principalPaid;
    if (principalPaid >= loan.principal) { loan.principal = 0; loan.startTime = 0; loan.lastRepayTime = 0; }
    else { loan.principal -= principalPaid; loan.lastRepayTime = block.timestamp; }
    emit Repaid(msg.sender, principalPaid, interestPaid);
}
```
The frontend performs this as a two-step transaction (`approve` → `repay`),
reflected as a 2-step progress bar in the UI.

### 8.7 Buy MYR
Swaps ETH directly for MYR at the contract's on-chain rate — useful for
topping up MYR to make a repayment:

```solidity
function buyMYR(uint256 myrAmount) external payable whenNotPaused nonReentrant {
    require(ethPrice > 0, "Price not set");
    uint256 ethNeeded = (myrAmount * 1e18) / (ethPrice * MYR_DECIMALS);
    require(msg.value >= ethNeeded, "Insufficient ETH");
    myr.mint(msg.sender, myrAmount);
    uint256 excess = msg.value - ethNeeded;
    if (excess > 0) { payable(msg.sender).transfer(excess); }
    emit MYRPurchased(msg.sender, ethNeeded, myrAmount);
}
```
This is independent of the user's loan position — it never touches
`loans[msg.sender]`.

### 8.8 KYC Identity Verification
A 5-step wizard (`web/src/app/(screen)/kyc/page.tsx`) collects personal info,
address, financial declaration, and ID document photos (compressed
client-side to a base64 data URI and stored in the `KycSubmission` row).
Submission status starts `pending`; an admin approves it from `/admin`,
which updates the database **and** calls the contract's `setKYC()` using the
server-held owner key — so the user never signs or pays gas to get verified:

```typescript
// web/src/lib/kyc/chain.ts
export async function setKycOnChain(wallet: string, approved = true) {
  const signer = new ethers.Wallet(process.env.OWNER_PRIVATE_KEY!, provider);
  const contract = new ethers.Contract(LOAN_ADDR, SET_KYC_ABI, signer);
  const tx = await contract.setKYC(wallet, approved);
  await tx.wait();
}
```
`WalletContext` polls `/api/kyc?wallet=` every 10 seconds and auto-heals any
DB↔chain mismatch it detects.

### 8.9 Tamper-Proof Audit Trail
Every fund-moving action emits a Solidity event
(`CollateralDeposited`, `Borrowed`, `Repaid`, `CollateralWithdrawn`,
`MYRPurchased`, `Liquidated`, `KYCSet`, `PriceUpdated`). The portfolio page's
`useTransactionHistory()` hook reads these directly from the chain via
`contract.queryFilter(...)`, which is immutable and independently
verifiable — the append-only `LoanTransaction` database table is only a
convenience fallback (used when a direct chain query isn't available), never
the source of truth.

### 8.10 Liquidation & Risk Controls
An undercollateralized position (health factor < 1.0, i.e. LTV ≥ 80%) can be
liquidated by a whitelisted liquidator, who repays part of the debt and
receives the equivalent collateral plus a 5% bonus:

```solidity
function liquidate(address borrower, uint256 debtAmount) external onlyLiquidator {
    require(_healthFactor(borrower) < MIN_HEALTH, "Not liquidatable");
    ...
    uint256 collateralValue = (covering * PRECISION) / (ethPrice * MYR_DECIMALS);
    uint256 bonus           = (collateralValue * LIQ_BONUS) / 100;
    uint256 seize           = collateralValue + bonus;
    ...
}
```
The admin-set ETH/MYR price (`setEthPrice`) is also capped at a 20% move per
update, preventing a single bad price push from mass-liquidating positions.

### 8.11 Admin Management
`web/src/app/(screen)/admin/page.tsx` (admin-only, enforced by middleware)
lists all `KycSubmission` rows, lets an admin inspect uploaded ID photos, and
approve/reject applications — driving both the database status and the
on-chain `kycApproved` flag described in 8.8.

---

## 9. Smart Contract

### Contract Overview
`CryptoLoan.sol` is an ETH-collateralized MYR lending protocol
("Nexo-style"). It inherits OpenZeppelin's `ReentrancyGuard`, `Pausable`, and
`Ownable2Step`, and deploys its own `MockMYR` ERC-20 token in its
constructor. Key parameters:

| Constant | Value | Meaning |
|---|---|---|
| `MAX_LTV` | 70% | Maximum loan-to-value for borrow/withdraw |
| `LIQ_THRESHOLD` | 80% | LTV at which a position becomes liquidatable |
| `LIQ_BONUS` | 5% | Bonus collateral paid to a liquidator |
| `BORROW_APR_BPS` | 480 (4.8%) | Annual interest rate on borrowed MYR |
| `MAX_PRICE_CHANGE` | 20% | Max allowed single-call move in the admin-set ETH/MYR price |
| `MYR_DECIMALS` | 1e6 | MockMYR decimal precision |

### Functions
| Function | Access | Description |
|---|---|---|
| `depositCollateral()` | any wallet | Locks `msg.value` ETH as collateral |
| `borrow(uint256 myrAmount)` | KYC-approved only | Mints MYR up to 70% of collateral value |
| `repay(uint256 myrAmount)` | any wallet with a loan | Pays interest, then principal |
| `withdrawCollateral(uint256 weiAmount)` | any wallet | Returns ETH if remaining LTV stays ≤ 70% |
| `buyMYR(uint256 myrAmount)` | any wallet | Swaps ETH for MYR at the current price |
| `liquidate(address borrower, uint256 debtAmount)` | whitelisted liquidator / owner | Repays debt on an undercollateralized position, seizes collateral + 5% bonus |
| `setKYC(address user, bool approved)` | owner only | Approves/revokes a wallet's borrowing eligibility |
| `setLiquidator(address, bool)` | owner only | Whitelists/removes a liquidator |
| `setEthPrice(uint256)` | owner only | Updates the MYR-per-ETH price (max 20% move) |
| `pause()` / `unpause()` | owner only | Emergency circuit breaker |
| `withdrawProtocolFees(address to)` | owner only | Sweeps accrued interest revenue |
| `accruedInterest`, `totalDue`, `healthFactor`, `availableToBorrow`, `currentLTV`, `getLoanInfo`, `getProtocolStats` | public view | Read-only helpers used throughout the UI |

### Events
| Event | Emitted when |
|---|---|
| `CollateralDeposited(user, amount)` | after `depositCollateral` |
| `Borrowed(user, myrAmount, newTotal)` | after `borrow` |
| `Repaid(user, principal, interest)` | after `repay` |
| `CollateralWithdrawn(user, amount)` | after `withdrawCollateral` |
| `MYRPurchased(buyer, ethSpent, myrReceived)` | after `buyMYR` |
| `Liquidated(user, liquidator, debtCovered, collateralSeized)` | after `liquidate` |
| `KYCSet(user, approved)` | after `setKYC` |
| `PriceUpdated(oldPrice, newPrice, updatedBy)` | after `setEthPrice` |
| `LiquidatorSet(liquidator, approved)` | after `setLiquidator` |
| `ProtocolFeesWithdrawn(to, amount)` | after `withdrawProtocolFees` |

### Contract Address
Deployed addresses are **not hardcoded in documentation** — the deploy script
(`blockchain/scripts/deploy.ts`) writes them automatically to
`web/src/lib/contractConfig.ts` on every local deployment (and to
`icoConfig.ts` for the separate ICO contracts). For a public network, record
the address printed by `npm run deploy:sepolia` / `deploy:mainnet` and its
Etherscan-verified link after running `npm run verify:sepolia` / `verify:mainnet`.

---

## 10. Database Design

### ER Diagram
```
┌───────────────┐        1:1        ┌───────────────────┐
│     User        │───────────────────│    BankAccount      │
│ id (PK)          │                    │ id (PK)              │
│ name             │                    │ userId (FK, unique)  │
│ email (unique)   │                    │ bankName             │
│ password         │                    │ accountNumber        │
│ walletAddress    │                    │ accountHolder        │
│ isAdmin          │                    │ recipientAddress     │
│ createdAt        │                    └───────────────────┘
└───────┬─────────┘
        │ 1:many
        ▼
┌───────────────────┐
│   BankTransfer       │
│ id (PK)               │
│ userId (FK)           │
│ amountMYR              │
│ status                 │
│ referenceNo (unique)   │
│ bankName / accountLast4│
│ createdAt / completedAt│
└───────────────────┘

┌───────────────────┐        ┌───────────────────┐
│   LoanTransaction     │        │    KycSubmission      │
│ id (PK)                │        │ id (PK)                │
│ wallet (indexed)       │        │ wallet (unique)        │
│ type                    │        │ personal/address/      │
│ amount                  │        │ financial fields       │
│ txHash (unique)         │        │ icFrontData/icBackData/│
│ blockNumber              │       │ selfieData (base64)    │
│ createdAt                │       │ status                 │
└───────────────────┘         └───────────────────┘
   (linked to a wallet address, not a foreign key to User —
    a wallet can transact before/without an email account)
```

### Tables
| Table | Key Fields | Notes |
|---|---|---|
| `User` | `id`, `email` (unique, nullable), `password` (nullable), `walletAddress` (unique, nullable), `isAdmin` | Either `email`/`password` or `walletAddress` may be null — a user can sign up with just one method |
| `BankAccount` | `userId` (unique FK), `bankName`, `accountNumber`, `accountHolder`, `recipientAddress` | One-to-one with `User`; `recipientAddress` is the on-chain address MYR can be sent to |
| `BankTransfer` | `userId` (FK), `amountMYR`, `status`, `referenceNo` (unique) | Simulated DuitNow disbursement; status auto-progresses `PENDING → PROCESSING → COMPLETED` |
| `LoanTransaction` | `wallet` (indexed), `type`, `amount`, `txHash` (unique), `blockNumber` | Mirrors on-chain events for a wallet; upserted by `txHash` so re-submits are idempotent |
| `KycSubmission` | `wallet` (unique), full application fields, `status` | One row per wallet, upserted on resubmission; document photos stored as base64 data URIs directly in the row |

### Relationships
- `User 1—1 BankAccount` (a user may have zero or one bank account on file)
- `User 1—* BankTransfer` (a user can have many historical transfer records)
- `LoanTransaction` and `KycSubmission` key off `wallet` (a lowercased address string), not a `User` foreign key — this lets on-chain activity and KYC be tracked even for a wallet that hasn't (yet) created an email-linked `User` row.

---

## 11. API Documentation

All endpoints live under `web/src/app/api/` and are implemented as Next.js
route handlers (no separate backend server). JSON in, JSON out.

### Authentication APIs
| Endpoint | Method | Description |
|---|---|---|
| `/api/auth/signup` | POST | `{ name, email, password }` → creates a `User`, hashes the password with bcrypt (cost 12), sets the session cookie. Rejects passwords under 8 characters and duplicate emails (409). |
| `/api/auth/login` | POST | `{ email, password }` → verifies bcrypt hash, sets the session cookie. |
| `/api/auth/wallet-nonce` | GET | `?address=0x..` → issues a one-time nonce (5-minute expiry, in-memory store) for the client to sign. |
| `/api/auth/wallet-login` | POST | `{ address, signature, nonce }` → verifies the nonce and the `ethers.verifyMessage` recovered signer, finds-or-creates the `User`, sets the session cookie. |
| `/api/auth/me` | GET | Returns the current session's user (from the JWT cookie), or `401` if unauthenticated. |
| `/api/auth/logout` | POST | Clears the session cookie. |

### Lending / Wallet APIs
> Deposit, Borrow, Withdraw, Repay, and Buy MYR are **not** REST endpoints —
> they are direct `ethers.js` calls from the browser to the smart contract
> (see Section 8.3–8.7). The API layer only *mirrors* the resulting
> transactions for history/reporting:

| Endpoint | Method | Description |
|---|---|---|
| `/api/loan-tx` | POST | `{ wallet, type, amount, txHash, blockNumber }` → upserts a `LoanTransaction` row (idempotent on `txHash`) right after a transaction confirms client-side. |
| `/api/loan-tx` | GET | `?wallet=0x..` → returns up to 100 recent transactions for a wallet, newest block first. |
| `/api/transfers` | GET | Returns the authenticated user's 50 most recent simulated bank transfers. |
| `/api/transfers` | POST | `{ amountMYR }` → creates a `PENDING` transfer against the user's saved bank account, then auto-advances it to `PROCESSING` (+3s) and `COMPLETED` (+7s) in the background. |
| `/api/prices` | GET | Proxies CoinGecko for 10 assets in MYR/USD with 24h change; edge-cached 15 seconds. |
| `/api/admin/sync-price` | POST | Fetches the live ETH/MYR rate and pushes the on-chain price toward it in ≤20%-per-call steps (owner-key signed). |
| `/api/admin/sync-ico-price` | POST | Same idea, re-pegging the ICO contract's sale price. |

### KYC APIs
| Endpoint | Method | Description |
|---|---|---|
| `/api/kyc` | POST | Saves the full KYC form (personal, address, financial fields + compressed ID photos as data URIs) via `upsert` keyed on `wallet`; sets `status: "pending"`. |
| `/api/kyc` | GET | `?wallet=0x..` → returns the submission's status/fields with the heavy image blobs stripped out. |
| `/api/kyc` | DELETE | `?wallet=0x..` → removes a submission (admin action). |
| `/api/kyc/documents` | GET / POST | Serves or updates the stored ID-photo data URIs separately from the lightweight status check. |
| `/api/kyc/approve` | POST | `{ wallet }` → sets `status: "approved"` in the database, then calls `setKycOnChain(wallet, true)` using the server's `OWNER_PRIVATE_KEY`. Also used as the self-healing resync endpoint when the DB and chain disagree. |

### Supporting APIs
| Endpoint | Method | Description |
|---|---|---|
| `/api/profile/bank-account` | GET | Returns the authenticated user's saved bank account, if any. |
| `/api/profile/bank-account` | PUT | Upserts the bank account (validates a 6–20 digit account number and a well-formed Ethereum `recipientAddress`). |

---

## 12. User Guide

### Login
1. Go to `/login`.
2. Either fill in email + password and click **Sign in**, or click
   **Continue with MetaMask**, approve the connection request, then sign the
   one-time message MetaMask prompts you with (no gas is charged for
   signing).
3. On success you land on `/dashboard` (or `/admin` if your account is an
   admin).

### Submit KYC
1. From the dashboard, click **Complete KYC** on any gated action, or
   navigate to `/kyc` directly.
2. Fill the 5 steps: Personal Info → Address → Financial Declaration →
   Document Upload (ID front/back photos) → Review & Submit.
3. Accept the Terms and Declaration checkboxes and submit. You'll receive a
   reference number (`KYC-000123`) and a "Pending Review" status.
4. Once an admin approves your submission (Section 8.11), your wallet is
   unlocked for borrowing — typically reflected within 10 seconds via the
   app's background polling, no page reload required.

### Borrow / Repay
1. Connect your wallet and ensure it's on the Hardhat/Sepolia/Mainnet network the app expects.
2. Open the dashboard's Action Dialog and select **Deposit**, enter an ETH
   amount, and confirm the MetaMask transaction.
3. Switch to the **Borrow** tab, enter a MYR amount within your available
   credit (shown live), and confirm.
4. To repay, switch to the **Repay** tab, enter an amount, and confirm
   **two** MetaMask prompts — one to `approve` the MYR spend, one for the
   actual `repay` call.
5. Use **Buy MYR** at any time if your MYR balance is short of what you owe.

### View Blockchain Records
- The **Portfolio** page shows your live position (collateral, debt, health
  factor) and a transaction history read directly from on-chain events
  (`Borrowed`, `Repaid`, `CollateralDeposited`, `CollateralWithdrawn`,
  `MYRPurchased`).
- Every transaction hash shown can be independently verified with any block
  explorer pointed at the same RPC/network.

---

## 13. Troubleshooting

### Common Errors

| Error | Where it appears |
|---|---|
| `KYC required` | Borrow transaction reverts |
| `Deposit collateral first` | Borrow transaction reverts with zero collateral |
| `Exceeds max LTV` | Borrow amount would push LTV past 70% |
| `Would violate LTV` | Withdraw amount would leave remaining debt above 70% LTV |
| `No active loan` | Repay attempted with zero outstanding principal |
| `Insufficient ETH` | Buy MYR sent less ETH than the current price requires |
| `Price move too large` | `setEthPrice` called with a jump greater than 20% |
| "Failed to fetch" / `ECONNREFUSED` in the wallet state | Frontend can't reach the RPC endpoint |
| "Invalid or expired nonce" | Wallet login signature step took too long or was replayed |
| "On-chain price differs from live market" banner | Contract's stored `ethPrice` has drifted from the CoinGecko rate |

### Solutions

| Problem | Fix |
|---|---|
| `KYC required` despite an approved-looking account | Wait ~10 seconds for the app's auto-resync poll, or manually re-trigger it by revisiting the dashboard; check that `OWNER_PRIVATE_KEY` is correctly set in `web/.env` |
| Any "Failed to fetch" / RPC errors | Confirm `npm run chain` is still running in its terminal, and that MetaMask is pointed at `http://127.0.0.1:8545`, chain ID `31337` |
| Contract calls fail after restarting the Hardhat node | Redeploy — a fresh node resets all contract state and addresses may shift. Run `npm run deploy:local` again (and `npm run deploy:ico` if used) |
| "Price move too large" when syncing | Click **Sync Price** again — the sync endpoint walks the price in ≤20% steps per call, so multiple clicks (or a scheduled re-run) may be needed to fully close a large gap |
| Prisma / database errors on startup | Confirm `DATABASE_URL` in `web/.env` is correct and reachable, then run `npx prisma migrate deploy` |
| Wallet login "Invalid or expired nonce" | Retry the sign-in flow from the start — nonces expire after 5 minutes and are single-use |
| MYR balance not visible in MetaMask after borrowing | Click **"+ Add MYR to MetaMask"** on the dashboard/portfolio — ERC-20 balances don't appear automatically and must be imported via `wallet_watchAsset` |
| Admin page inaccessible | Confirm the logged-in user's `isAdmin` column is `true` in the database — there is no self-service admin signup |
| Contract address mismatch after a teammate's redeploy | Pull the latest `web/src/lib/contractConfig.ts` (auto-generated, should be committed or regenerated locally) rather than editing addresses by hand |

---

## 14. Future Improvements

- **Multi-collateral support** — extend `CryptoLoan.sol` beyond ETH to accept other assets shown in the Markets page (BTC, SOL, etc.), likely via a per-asset collateral mapping and price feed.
- **Chainlink price oracle integration** — replace the admin-set `ethPrice` with a decentralized oracle feed to remove reliance on a manually-triggered sync and the 20%-per-update guard.
- **On-chain KYC attestation** — move away from a centrally-approved `setKYC()` call toward a verifiable-credential or zero-knowledge KYC proof, reducing trust in the server-held owner key.
- **Automated liquidation bots** — a keeper service that watches health factors and calls `liquidate()` automatically instead of relying on a manually whitelisted liquidator.
- **Variable/tiered interest rates** — rates that respond to protocol utilization rather than the fixed 4.8% APR constant.
- **Real bank-transfer integration** — replace the simulated DuitNow `BankTransfer` flow with an actual payment-rail integration.
- **Mobile app / responsive PWA** — a dedicated mobile experience beyond the current responsive web layout.
- **Multi-language support** — Bahasa Malaysia alongside English, given the MYR/Malaysian regulatory framing.
- **Formal security audit** — a third-party audit of `CryptoLoan.sol` before any mainnet deployment with real funds.
