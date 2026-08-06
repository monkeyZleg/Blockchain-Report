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

The codebase is organised as two independent npm projects in a single
repository: `blockchain/` (Solidity contracts + Hardhat) and `web/`
(the Next.js application, API routes, and Prisma/Postgres data layer).

## 2. System Architecture

```mermaid
flowchart LR
    U[User + MetaMask] -->|signs txs| C[CryptoLoan.sol\nHardhat node :8545]
    U -->|browser| W[Next.js app :3000]
    W -->|API routes| DB[(Postgres via Prisma)]
    W -->|ethers.js reads +\nowner-signed writes| C
    A[Admin] --> W
```

- **On-chain is the source of truth for money**: collateral, debt, interest,
  liquidation, the ETH/MYR price, and a per-wallet `kycApproved` flag all
  live in the `CryptoLoan.sol` smart contract.
- **Off-chain (Postgres) is the source of truth for identity**: user
  accounts, KYC submissions and documents, feature flags, and the admin
  audit log. It never holds money or payout instructions — loan
  disbursement is the on-chain MYR transfer, nothing more.
- **The web app bridges the two**: users sign their own transactions with
  MetaMask; the server (holding the contract owner key) performs
  owner-only calls such as `setKYC` and price updates.

### 2.1 Smart Contracts

`CryptoLoan.sol` is the core protocol — **70% max LTV**, **80% liquidation
threshold**, **5% liquidator bonus**, and a variable APR (3% base + up to
4% utilization premium + up to 3% volatility premium, accrued linearly).
It deploys its own `MockMYR` token and holds the ETH/MYR price on-chain
(owner-updated, capped at ±20% per move).

| Function | Who | What |
|---|---|---|
| `depositCollateral()` | anyone | Send ETH, credited as collateral |
| `borrow(myr)` | KYC-flagged wallets | Mints MYR up to 70% of collateral value |
| `repay(myr)` | borrower | Pulls approved MYR; interest first, then principal |
| `withdrawCollateral(wei)` | borrower | Allowed only if remaining position stays under max LTV |
| `buyMYR(myr)` | anyone | Swap ETH → MYR at the on-chain price |
| `liquidate(borrower, debt)` | whitelisted liquidators | When health factor < 1: repay debt, seize collateral + 5% bonus |
| `setKYC`, `setEthPrice`, `pause`, `withdrawProtocolFees` | owner | Admin/ops |

Health factor = (collateral value × 80%) / debt — below 1.0 the position
is liquidatable. Also present: `MockMYR.sol` (6-decimal ERC-20), and a
separate token-sale demo (`ICO.sol`, `RinggitToken.sol`, `MockUSDC.sol`)
behind the `/ico` page.

### 2.2 Web Application

The Next.js App Router application handles auth (email/password or
wallet-based, with JWT sessions), an account-centric KYC and wallet-linking
model, feature flags, and admin tooling (user management, KYC review,
transaction/audit views, price sync).

## 3. Technology Stack

**Web application (`web/`)**
- Next.js `16.2.6`, React `19.2.4`, TypeScript `^5`
- Database/ORM: Supabase via Prisma `^5.22.0`
- Auth: `bcryptjs` (password hashing), `jose` (JWT)
- Blockchain client: `ethers` `^6.16.0`
- UI: `@mui/material`, `@emotion/react`/`styled`, `tailwindcss ^4`, `recharts`, `lenis` (smooth scroll)

**Smart contracts (`blockchain/`)**
- `hardhat ^2.22.17`, `@nomicfoundation/hardhat-toolbox`
- `@openzeppelin/contracts ^5.1.0`
- Solidity

**Runtime**: Node.js `>= 20.9.0`.

## 4. Installation Guide

### 4.1 Prerequisites
- Node.js 20.x (pinned via `.nvmrc`)
- npm

### 4.2 Environment Variables

Copy `.env.example` to `.env` in **both** the repository root (used by
`blockchain/hardhat.config.ts`) and `web/` (used by Next.js/Prisma):

```bash
cp .env.example .env
cp .env.example web/.env
```

| Variable | Used by | Notes |
|---|---|---|
| `DATABASE_URL` | `web/` (Prisma) | Postgres connection string |
| `OWNER_PRIVATE_KEY` | `blockchain/` | Key used to deploy/own contracts on testnet/mainnet |
| `HARDHAT_RPC_URL` | `blockchain/` | Defaults to local Hardhat node |
| `ALCHEMY_API_KEY` | `blockchain/` | RPC provider for Sepolia/mainnet |
| `ETHERSCAN_API_KEY` | `blockchain/` | Contract verification |
| `JWT_SECRET` | `web/` | Optional locally — falls back to a dev value if unset |

### 4.3 Install Dependencies

```bash
cd blockchain && npm i
cd web && npm i
```

This will install their own node modules, and therefore both folder will have their installed dependecies.  
## 5. Running the System

Run each of these in its own terminal, from `blockchain/`:
```bash
cd blockchain
```

```bash
# Terminal A — local chain (keep running)
npm run chain

# Terminal B — deploy contracts (writes web/src/lib/contractConfig.ts)
npm run deploy:local
```

Then, from `web/`:
```bash
cd web
```

```bash
npm run dev
```

Now u can click Open `http://localhost:3000`.

**Connecting a wallet:**
1. Add a custom network in MetaMask: RPC `http://127.0.0.1:8545`, Chain ID `31337`.
2. Import one of the private keys printed by `npm run chain`.
3. Click "Connect Wallet" on the site.

## 6. Folder Structure
There are two folder for handling this system
```
blockchain/
web/
```


```
crypto-loan/
├── README.md                    Setup guide
├── blockchain/
│   ├── contracts/               
│   ├── scripts/                 deploy.ts, deploy-ico.ts, set-price.ts, verify.ts
│   ├── test/                    CryptoLoan.test.ts
│   └── hardhat.config.ts
└── web/
    ├── prisma/
    │   ├── schema.prisma        
    │   └── migrations/
    ├── public/                  
    └── src/
        ├── app/                 
        │   ├── (screen)/        
        │   ├── api/              
        │   └── home/, login/, signup/, ico/
        ├── components/          
        ├── hooks/               
        └── lib/ 
    ├── env/                
```

## 7. Future Improvements

- **Optimize the SQL database design**
	- Could have improve database normalization, indexing strategies, and query performance to enhance scalability, data integrity, and overall system efficiency.
- **Improve API architecture**
	- reorganize API calls into a more modular and maintainable folder structure, following a clear separation of concerns to improve code readability and accessibility.
- **Expand automated test coverage** 
	- extend testing beyond the current smart contract tests to include liquidation edge cases, price fluctuation validation, API integration tests, and web application authentication and KYC workflows.
- **Enhance rate limiting and audit logging** 
	- implement comprehensive rate limiting to protect against abuse and expand the audit trail to cover critical user activities, such as repeated failed wallet login attempts and other security-sensitive operations.
- **Enhance the KYC verification process** 
	- integrate AI-powered identity verification with facial recognition technology to automate identity validation, improve verification accuracy, reduce fraudulent registrations, and provide a more secure and seamless onboarding experience.
- **Support multiple collateral assets and blockchain networks** 
	- enhance the platform by allowing additional collateral types and deploying on multiple blockchain networks or Layer 2 solutions to improve flexibility, scalability, and user adoption.
