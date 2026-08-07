## 8. Database Design

The web app persists to a single **PostgreSQL** database (Railway in production) via **Prisma** ORM ([`web/prisma/schema.prisma`](../../../web/prisma/schema.prisma)). `BankAccount`/`BankTransfer` never appear in the schema. Bank Transfer was dropped from the product before this schema was written, so there's nothing to remove here.

### Table of contents, Section 8

- [8.1 On-Chain vs Off-Chain Split](#81-on-chain-vs-off-chain-split)
- [8.2 Schema at a Glance](#82-schema-at-a-glance)
- [8.3 Entity-Relationship Diagram](#83-entity-relationship-diagram)
- [8.4 Table-by-Table Walkthrough](#84-table-by-table-walkthrough)

---

### 8.1 On-Chain vs Off-Chain Split

The database is deliberately **not** the source of truth for money. That role belongs entirely to the `CryptoLoan` contract (collateral, debt, interest, the per-wallet `kycApproved` flag; see Section 9 below). Postgres owns everything else (accounts, KYC submissions and documents, feature flags, and the admin audit trail), and one table, `LoanTransaction`, exists purely as a **read-optimised mirror** of on-chain events, not as a payment record. If the database and the chain ever disagree about a balance, the chain wins; Section 7.5 Portfolio (in the full document) makes this explicit by falling back to the database mirror only when a live chain read isn't available.

### 8.2 Schema at a Glance

Six models, defined in [`web/prisma/schema.prisma`](../../../web/prisma/schema.prisma):

- **`User`**, the account: `email`+`password` and/or a linked `walletAddress`, `isAdmin`, `status` (`ACTIVE`/`RESTRICTED`/`SUSPENDED`), `sessionEpoch` (JWT invalidation).
- **`KycSubmission`**, one per user (not per wallet): personal/address/financial fields, ID document images, `status` (`pending`/`approved`/`rejected`).
- **`LoanTransaction`**, a mirrored copy of on-chain protocol events, keyed by wallet + unique `txHash`.
- **`BorrowPosition`**, the per-loan ledger row that annotates each on-chain loan (keyed by its `loanId`) with the installment-plan metadata the contract doesn't store.
- **`FeatureFlag`**, admin overrides (`ON`/`MAINTENANCE`/`HIDDEN`) on top of the `lib/features.ts` registry.
- **`AdminAuditLog`**, append-only record of every admin mutation.

### 8.3 Entity-Relationship Diagram

```mermaid
erDiagram
    User ||--o| KycSubmission : "has (optional, 1:1)"

    User {
        String id PK
        String name
        String email UK
        String password
        String walletAddress UK
        Boolean isAdmin
        String status
        String statusReason
        DateTime statusChangedAt
        Int sessionEpoch
        DateTime createdAt
        DateTime updatedAt
    }

    KycSubmission {
        Int id PK
        String userId FK "UK, one per user"
        String wallet "on-chain anchor, nullable"
        String fullName
        String docType
        String icNumber
        String dob
        String gender
        String nationality
        String phone
        String email
        String addr1
        String addr2
        String postcode
        String city
        String state
        String employment
        String income
        String purpose
        String fundSource
        String icFrontData
        String icBackData
        String selfieData
        String status
        DateTime submittedAt
        DateTime updatedAt
    }

    LoanTransaction {
        String id PK
        String wallet "indexed, no FK"
        String type
        String amount
        String txHash UK
        Int blockNumber
        DateTime createdAt
    }

    BorrowPosition {
        String id PK
        String wallet "indexed, no FK"
        Int loanId "on-chain loan index, nullable"
        DateTime dueDate "denormalized from chain"
        String principal
        String originalPrincipal
        Int aprBps
        Int baseAprBps
        Int termMonths
        String txHash UK
        DateTime borrowedAt
        String status
        DateTime repaidAt
        String repayTxHash
        String interestPaid
    }

    FeatureFlag {
        String key PK
        String state
        String message
        DateTime updatedAt
        String updatedBy
    }

    AdminAuditLog {
        String id PK
        String actorId "no FK"
        String actorEmail
        String action
        String targetType
        String targetId
        String detail
        DateTime createdAt
    }
```

`User → KycSubmission` is the only enforced foreign key in the schema. `LoanTransaction`, `BorrowPosition`, and `AdminAuditLog` deliberately key off a **wallet address / actor ID string** rather than a relation. They're append-only, event-sourced tables mirroring on-chain or admin activity that must keep existing even if the referenced account is later restricted or its wallet unlinked.

The same seven tables as deployed, from Supabase's own Schema Visualizer (`public` schema, `jianzhi86's Project`):

![DatabaseSchema_Supabase](DatabaseSchema_Supabase.png)

### 8.4 Table-by-Table Walkthrough

**`User`**, the account record:

```prisma
model User {
  id            String        @id @default(cuid())
  name          String?
  email         String?       @unique
  password      String?
  walletAddress String?       @unique
  isAdmin       Boolean       @default(false)
  status        String        @default("ACTIVE")
  statusReason  String?
  statusChangedAt DateTime?
  sessionEpoch  Int           @default(0)
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @default(now()) @updatedAt
  kyc           KycSubmission?
}
```

Either `email`+`password`, `walletAddress`, or both, may be set (never `walletAddress`-only *and* email-only-without-a-password; see the Settings invariant in Section 7.14). `sessionEpoch` is bumped to invalidate every JWT already issued to that user, since sessions are stateless cookies rather than server-tracked.

`status` has three values, and the difference between the last two matters:

| `status`     | Effect                                                                                                                       |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| `ACTIVE`     | Normal account                                                                                                               |
| `RESTRICTED` | May still sign in and read, but every write is refused                                                                       |
| `SUSPENDED`  | Refused at sign-in entirely: `getSessionUser()` treats the session as absent, and suspending bumps `sessionEpoch` to kill any token already live |

All three are **off-chain only**. As the schema's own comment states, a user in any of these states can still call the contract directly from their own wallet; `lib/authz.ts` carries the full caveat. Restricting an account stops CryptoLend's app from acting for them, not the blockchain from accepting their signature.

The live `User` table in Supabase's Table Editor:

![DatabaseTable_User](DatabaseTable_User.png)

**`KycSubmission`**, one row per **user**, not per wallet:

```prisma
model KycSubmission {
  id          Int      @id @default(autoincrement())
  userId      String   @unique
  user        User     @relation(fields: [userId], references: [id])
  wallet      String?
  fullName    String
  docType     String   @default("ic")
  icNumber    String
  dob         String
  gender      String
  nationality String
  phone       String
  email       String
  addr1       String
  addr2       String   @default("")
  postcode    String
  city        String
  state       String
  employment  String
  income      String
  purpose     String
  fundSource  String
  icFrontData String?
  icBackData  String?
  selfieData  String?
  status      String   @default("pending")
  submittedAt DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([wallet])
}
```

`userId` being `@unique` is what lets verification survive a wallet being unlinked and relinked, and stops a freed wallet from inheriting a stranger's approval (see Section 7.13). `wallet` is only the on-chain anchor used when the server calls `setKYC()` ([9.6](#96-admin--oracle-functions)), and is `null` until a wallet is linked. The five wizard steps map directly onto its fields: personal info (`fullName`…`nationality`), contact (`phone`, `email`), address (`addr1`…`state`), financial declaration (`employment`, `income`, `purpose`, `fundSource`), and documents (`icFrontData`, `icBackData`, `selfieData`, stored as base64 data URIs *in the row itself*, served back out through `GET /api/kyc/documents` rather than an external object store).

**`LoanTransaction`**, a denormalised mirror of on-chain protocol events:

```prisma
model LoanTransaction {
  id          String   @id @default(cuid())
  wallet      String
  type        String
  amount      String
  txHash      String   @unique
  blockNumber Int
  createdAt   DateTime @default(now())

  @@index([wallet])
}
```

Indexed by `wallet` and unique on `txHash` so re-indexing the same block twice can't double-insert. This is what backs the Explorer (Section 7.6) and Portfolio history (Section 7.5) without hitting an RPC node for every page load. It's populated by `POST /api/loan-tx`, which `WalletContext.tsx` calls right after every confirmed transaction (see the `saveTxToDB` calls throughout [9.5](#95-core-function-walkthroughs)).

The live `LoanTransaction` table in Supabase's Table Editor:

![DatabaseTable_LoanTransaction](DatabaseTable_LoanTransaction.png)

**`BorrowPosition`**, one row per on-chain loan, carrying the plan metadata the contract doesn't store:

```prisma
model BorrowPosition {
  id           String    @id @default(cuid())
  wallet       String
  /// On-chain loan index for this wallet (CryptoLoan._userLoans[wallet][loanId]).
  loanId       Int?
  /// This loan's on-chain due date (startTime + term), denormalized at borrow time.
  dueDate      DateTime?
  principal    String
  originalPrincipal String @default("0")
  aprBps       Int
  /// Base rate at the same moment; display only, for the "base + premium" split.
  baseAprBps   Int       @default(0)
  termMonths   Int       @default(1)
  txHash       String    @unique
  borrowedAt   DateTime  @default(now())
  status       String    @default("OPEN") // OPEN | REPAID | LIQUIDATED
  repaidAt     DateTime?
  repayTxHash  String?
  interestPaid String?

  @@index([wallet, status])
  @@index([wallet, loanId])
}
```

This table's role changed when the contract moved to per-loan state. The chain now keeps a real `Loan[]` array per wallet (§9.3), so principal, APR, due date and interest are all authoritative on-chain. This row no longer *reconstructs* itemisation the contract lacks, it **annotates** an existing on-chain loan.

`loanId` is the join key: it's read out of the `LoanCreated` event right after the borrow confirms, and it's what lets a repayment settle exactly the rows its per-loan `Repaid(loanId, …)` events covered (§9.5.3). It is nullable only for rows written before the per-loan contract existed.

What genuinely lives only here:

| Field               | Why the chain doesn't have it                                                                                                                                                                  |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `termMonths`        | The 1/3/6/12 installment plan. The contract stores `termDays` (30/90/180/365) and a `dueDate`, but has no concept of monthly instalments; that is a UI construct                                |
| `originalPrincipal` | Frozen at creation while `principal` shrinks as payments settle. The difference between the two is what lets an early payoff advance "Month M of N" correctly, instead of dividing by elapsed calendar time and showing a plan as behind when it is actually paid ahead |
| `baseAprBps`        | The base rate at borrow time, so the Repay tab can show "3.00% base + 0.36% utilisation" rather than just the blended figure. Display only; `aprBps` is what interest is priced off everywhere. It defaults to `0`, meaning "unknown", and the UI falls back to an APR-only display for those rows |
| `dueDate`           | Denormalised for admin/reporting queries. The dashboard reads the live value from the chain, never from here                                                                                    |

The live `BorrowPosition` table in Supabase's Table Editor:

![DatabaseTable_BorrowPosition](DatabaseTable_BorrowPosition.png)

**`FeatureFlag`**, one optional override row per feature key declared in `lib/features.ts`:

```prisma
model FeatureFlag {
  key       String   @id
  state     String   @default("ON")
  message   String   @default("")
  updatedAt DateTime @default(now()) @updatedAt
  updatedBy String?
}
```

An **empty table means every feature defaults to on**. `state` is `"ON"`, `"MAINTENANCE"` (visible, blocked, shows `message`), or `"HIDDEN"` (removed from navigation, direct routes 404). This is the mechanism behind every "temporarily paused" state referenced throughout Section 7: signups (7.1), logins (7.2), and each of the five loan actions (7.7-7.11) independently.

The live `FeatureFlag` table in Supabase's Table Editor:

![DatabaseTable_FeatureFlag](DatabaseTable_FeatureFlag.png)

**`AdminAuditLog`**, append-only by design:

```prisma
model AdminAuditLog {
  id         String   @id @default(cuid())
  actorId    String
  actorEmail String?
  action     String
  targetType String
  targetId   String
  detail     String?
  createdAt  DateTime @default(now())

  @@index([createdAt])
  @@index([targetType, targetId])
}
```

Never edited or deleted by the app. There is deliberately no update/delete endpoint for it (see Section 7.12). `detail` stores a JSON blob (`{ before, after }`) as text rather than a native JSON column, to stay portable across the raw-SQL paths the app uses elsewhere. Indexed on `createdAt` (for the audit feed) and on `(targetType, targetId)` (for "show me every action taken on this user/submission").

The live `AdminAuditLog` table in Supabase's Table Editor:

![DatabaseTable_AdminAuditLog](DatabaseTable_AdminAuditLog.png)

---
