
## 6. Smart Contract Deployment

CryptoLend's blockchain layer lives entirely inside `blockchain/`, an independent Hardhat + TypeScript project (Solidity `0.8.24`, optimizer on at 200 runs, `viaIR` enabled) with no runtime dependency on the `web/` Next.js app. The two projects only ever touch through one generated file, `web/src/lib/contractConfig.ts`, produced by the deploy step below.

| Layer              | Technology                                                                                                 |
| ------------------ | ---------------------------------------------------------------------------------------------------------- |
| Smart contracts    | Solidity `0.8.24`, Hardhat, OpenZeppelin `5.1`                                                             |
| Deployment tooling | Hardhat scripts (`deploy.ts`, `deploy-ico.ts`, `verify.ts`, `set-price.ts`, `time-travel.ts`), ethers.js v6 |
| Networks           | Hardhat local (chain `31337`), Sepolia, Ethereum Mainnet (via Alchemy)                                     |

A single `npm run deploy:local` puts **two** contracts on chain, because `CryptoLoan`'s constructor deploys its own token:

| Contract        | How it gets deployed                              | Purpose                                              |
| --------------- | ------------------------------------------------- | ---------------------------------------------------- |
| `CryptoLoan`    | Directly by `deploy.ts`, with the ETH price as its constructor argument | The lending protocol itself (Section 9)      |
| `MockMYR`       | Indirectly, by `CryptoLoan`'s constructor (`new MockMYR()`) | The 6-decimal MYR token the protocol mints  |

### Table of contents, Section 6

- [6.1 Prerequisites & Environment Variables](#61-prerequisites--environment-variables)
- [6.2 Installing Dependencies](#62-installing-dependencies)
- [6.3 Network Configuration](#63-network-configuration)
- [6.4 Local Deployment (Hardhat Node)](#64-local-deployment-hardhat-node)
- [6.5 What `deploy.ts` Actually Does](#65-what-deployts-actually-does)
- [6.6 Testnet / Mainnet Deployment & Verification](#66-testnet--mainnet-deployment--verification)
- [6.7 Post-Deploy Operations](#67-post-deploy-operations)
- [6.8 Optional: ICO Demo Deployment](#68-optional-ico-demo-deployment)
- [6.9 Common Deployment Errors](#69-common-deployment-errors)

---

### 6.1 Prerequisites & Environment Variables

`hardhat.config.ts` reads its secrets from a single `.env` file at the **repo root** (`dotenv.config({ path: "../.env" })`), not from inside `blockchain/`. Copy `.env.example` to `.env` and fill in:

| Variable            | Required for                                       | Notes                                                                                                                             |
| ------------------- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `OWNER_PRIVATE_KEY` | Sepolia / Mainnet deploys, and any owner-only call | On local Hardhat, account #0's well-known test key (`0xac09...2ff80`) is used by convention; never reuse it outside a local chain |
| `ALCHEMY_API_KEY`   | Sepolia / Mainnet                                  | Feeds the RPC URL for both networks                                                                                               |
| `ETHERSCAN_API_KEY` | Contract verification                              | Used by `verify.ts` / `hardhat-toolbox`'s `verify:verify` task                                                                    |
| `REPORT_GAS`        | Optional                                           | `"true"` enables a gas-cost report on `npm test`                                                                                  |

Local deployment needs none of these. Hardhat's built-in `hardhat`/`localhost` network signs with its own funded test accounts.

![The .env file at the repo root, filled in from .env.example](asset/env.png)

*Figure 6.1: the repo-root `.env`, which both `hardhat.config.ts` and the Next.js server read.*

### 6.2 Installing Dependencies

```bash
cd blockchain
npm install
```

| Package                            | Role                                                                                                              |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `hardhat`                          | Local Ethereum node, Solidity compiler, test runner, deploy-script host                                           |
| `@nomicfoundation/hardhat-toolbox` | Bundles ethers.js, Chai matchers, the gas reporter, and the Etherscan verification plugin                         |
| `@openzeppelin/contracts`          | Audited base contracts the protocol inherits: `ReentrancyGuard`, `Pausable`, `Ownable2Step`, `ERC20`, `SafeERC20` |
| `dotenv`                           | Loads the repo-root `.env` into `hardhat.config.ts`                                                               |

Node is pinned to **20** via the repo's `.nvmrc` (`nvm use` if available). `blockchain/` and `web/` are installed completely independently. There is no workspace/monorepo tooling tying their `node_modules` together, so `npm install` must be run in each folder separately.

Compiling is worth doing once before the first deploy, since `deploy.ts` reads the ABIs it writes out of `artifacts/`:

```bash
npm run compile
```

```
Compiled 12 Solidity files successfully (evm target: paris).
```

### 6.3 Network Configuration

Four networks are declared in [`hardhat.config.ts`](../../blockchain/hardhat.config.ts):

```ts
networks: {
  hardhat:  { chainId: 31337 },
  localhost:{ url: "http://127.0.0.1:8545", chainId: 31337 },
  sepolia:  { url: `https://eth-sepolia.g.alchemy.com/v2/${ALCHEMY_KEY}`, chainId: 11155111, accounts: [DEPLOYER_PK] },
  mainnet:  { url: `https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`, chainId: 1,        accounts: [DEPLOYER_PK] },
}
```

`hardhat` and `localhost` both target chain ID `31337`, which is also the chain ID MetaMask must be configured with for local testing (see Section 7.1 onward, and Section 11, in the full document).

### 6.4 Local Deployment (Hardhat Node)

Deployment is a fixed three-terminal sequence; this is also how Section 5, Running the System, starts the whole stack:

```bash
# Terminal A (keep running)        # Terminal B (one-shot)              # Terminal C
cd blockchain                      cd blockchain                       cd web
npm run chain                      npm run deploy:local                npm run dev
```

- `npm run chain` runs `hardhat node`: starts an in-memory Ethereum node on `127.0.0.1:8545`, printing 20 funded test accounts and their private keys (import one into MetaMask).
- `npm run deploy:local` runs `hardhat run scripts/deploy.ts --network localhost`: compiles and deploys the contracts, detailed function-by-function in [6.5](#65-what-deployts-actually-does).
- `npm run dev` starts the Next.js app, which reads the contract addresses/ABIs the deploy step just wrote.

**Terminal A**, the local chain. Each account starts with 10,000 test ETH, and the private keys printed here are what get imported into MetaMask:

```
Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545/

Accounts
========
Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

Account #1: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (10000 ETH)
Private Key: 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
...
```

![Terminal A running npm run chain, showing the funded Hardhat test accounts and their private keys](asset/Deploy_HardhatNode.png)

*Figure 6.4.1: `npm run chain`. Leave this terminal running; closing it wipes all chain state.*

**Terminal B**, the deploy. The script prints exactly what it did and what to do next:

```
Deploying with account: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Balance: 10000.0 ETH

Fetching live ETH/MYR price from CoinGecko…
Initial ETH price: RM 18,432

CryptoLoan deployed to: 0x5FbDB2315678afecb367f032d93F642f64180aa3
MockMYR    deployed to: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
ETH price set to:       RM 18,432

Contract config written to: .../web/src/lib/contractConfig.ts

Next steps:
  1. Add Hardhat network to MetaMask (RPC: http://127.0.0.1:8545, Chain ID: 31337)
  2. Import an account using a private key from 'npm run chain' output
  3. Open http://localhost:3000 and click 'Connect Wallet'
```

![Terminal B running npm run deploy:local, showing both deployed contract addresses and the generated config path](asset/Deploy_DeployLocal.png)

*Figure 6.4.2: `npm run deploy:local`. Both addresses are deterministic on a fresh node, which is exactly what the guard in [6.5](#65-what-deployts-actually-does) protects.*

Available scripts, from [`blockchain/package.json`](../../blockchain/package.json):

| Script                                      | Command                                                 |
| ------------------------------------------- | ------------------------------------------------------- |
| `npm run compile`                           | `hardhat compile`                                       |
| `npm run chain` / `npm run node`            | `hardhat node`                                          |
| `npm run deploy` / `npm run deploy:local`   | `hardhat run scripts/deploy.ts --network localhost`     |
| `npm run deploy:ico`                        | `hardhat run scripts/deploy-ico.ts --network localhost` |
| `npm run deploy:sepolia` / `deploy:mainnet` | same script, `--network sepolia` / `mainnet`            |
| `npm run verify:sepolia` / `verify:mainnet` | `hardhat run scripts/verify.ts --network <net>`         |
| `npm test`                                  | `hardhat test`                                          |
| `npm run gas-report`                        | `REPORT_GAS=true hardhat test`                          |
| `npm run clean`                             | `hardhat clean`                                         |

The full sequence, end to end:

```mermaid
flowchart TD
    subgraph T1["Terminal A"]
        A["npm run chain"] --> B["Hardhat Node :8545<br/>(deployer nonce = 0)"]
    end

    subgraph T2["Terminal B"]
        A2["npm run deploy:local"] --> C["deploy.ts"]
    end

    C -- "getTransactionCount(deployer)" --> B
    B -- "nonce" --> C
    C -- "nonce ≠ 0, no FORCE_DEPLOY" --> X["⛔ Abort:<br/>restart the node first"]
    C -- "GET ethereum/myr price" --> D["CoinGecko API"]
    D -- "live price" --> C
    D -. "timeout / error,<br/>fall back RM 18,000" .-> C
    C -- "deploy(ethPrice)" --> E["CryptoLoan.sol"]
    E -- "new MockMYR()<br/>(inside constructor)" --> F["MockMYR.sol"]
    F -- "token address" --> E
    E -- "loan + myr address" --> C
    C -- "write addresses + ABIs" --> G["📄 contractConfig.ts"]

    subgraph T3["Terminal C"]
        H["npm run dev"] --> I["Next.js app"]
    end

    I -- "import" --> G
```

*Figure 6.4.3: the deployment flow. The three terminals run in parallel; the arrows between them are what makes ordering matter.*

### 6.5 What `deploy.ts` Actually Does

[`scripts/deploy.ts`](../../blockchain/scripts/deploy.ts) is not a bare `factory.deploy()` call. It encodes three deliberate safety/convenience behaviours.

**1. Fresh-node guard.** `CryptoLoan`'s address is derived from `(deployer, nonce)`, and its constructor deploys `MockMYR` internally, so *that* address is derived from `(CryptoLoan's address, its own nonce 0)`. Deploying onto a node whose deployer nonce isn't `0` would silently place both contracts at new addresses, and since MetaMask keys imported ERC-20 tokens by address, every redeploy would leave a stale duplicate "MYR" token behind in the wallet. The script therefore refuses to run unless the deployer's nonce is `0`, with an explicit override:

```ts
const nonce = await ethers.provider.getTransactionCount(deployer.address);
if (nonce !== 0 && process.env.FORCE_DEPLOY !== "1") {
  console.error(/* ...restart the node, or set FORCE_DEPLOY=1... */);
  process.exitCode = 1;
  return;
}
```

What that refusal looks like in practice:

```
✗ Node is not fresh — deployer nonce is 7, expected 0.
  Redeploying now would place MockMYR at a NEW address and add another
  duplicate "MYR" token in MetaMask.

  Fix: restart the Hardhat node, then deploy as the first action:
    1) Stop the running node, then:  npm run chain
    2) In a second terminal:         npm run deploy:local

  This keeps the MYR address constant. To deploy anyway (and accept a new
  token address), re-run with FORCE_DEPLOY=1.
```

![The deploy script aborting with the not-fresh-node error and its suggested fix](asset/Deploy_NonceGuard.png)

*Figure 6.5.1: the fresh-node guard. This is a refusal to proceed, not a crash; the exit code is `1` and nothing was deployed.*

**2. Live price seeding.** Before deploying, it fetches the current ETH/MYR rate from CoinGecko so the contract doesn't start on a stale hardcoded number, falling back to `RM 18,000` if the API call fails or times out (5s):

```ts
const res = await fetch(
  "https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=myr",
  { signal: AbortSignal.timeout(5000) }
);
```

The fallback is visible in the output when it fires, so a demo never silently runs on a made-up price:

```
Could not fetch live price: The operation was aborted due to timeout
Falling back to hardcoded price: RM 18,000
```

**3. Deploy, then generate the frontend config.** `CryptoLoan` is deployed with that price as its constructor argument (its constructor deploys `MockMYR` as part of the same transaction). The script then reads the compiled ABIs out of `artifacts/` and writes a single auto-generated file the Next.js app imports directly, with no manual address copy-pasting between the two projects:

```ts
const CryptoLoan = await ethers.getContractFactory("CryptoLoan");
const loan = await CryptoLoan.deploy(ethPrice);
await loan.waitForDeployment();

const loanAddress = await loan.getAddress();
const myrAddress  = await loan.myr();
// ...writes web/src/lib/contractConfig.ts with CONTRACT_ADDRESSES, CRYPTO_LOAN_ABI, MOCK_MYR_ABI
```

The generated file opens with a warning header, then the addresses and both full ABIs:

```ts
// !! Auto-generated by blockchain/scripts/deploy.ts — do not edit manually !!
// Re-run: npm run deploy:local

export const HARDHAT_CHAIN_ID = 31337;
export const HARDHAT_RPC_URL  = "http://127.0.0.1:8545";
export const ETH_PRICE_MYR    = 18432;

export const CONTRACT_ADDRESSES = {
  CryptoLoan: "0x5FbDB2315678afecb367f032d93F642f64180aa3",
  MockMYR:    "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512",
} as const;

export const CRYPTO_LOAN_ABI = [ /* ... */ ] as const;
export const MOCK_MYR_ABI    = [ /* ... */ ] as const;
```

![The generated contractConfig.ts, showing the do-not-edit header and the two contract addresses](asset/Deploy_ContractConfig.png)

*Figure 6.5.2: `web/src/lib/contractConfig.ts`. This is the only file linking the two npm projects, and it is regenerated on every deploy.*

This is the exact file `WalletContext.tsx` imports to build its contract instances (see [9.1](#91-contract-architecture)). Any restart of the Hardhat node resets its state, so **the contracts must be redeployed** (`npm run deploy:local`) after every restart; this is called out again in Section 11, Troubleshooting.

### 6.6 Testnet / Mainnet Deployment & Verification

The same `deploy.ts` script targets Sepolia or Mainnet by swapping the `--network` flag, using `OWNER_PRIVATE_KEY` and `ALCHEMY_API_KEY` from `.env`:

```bash
npm run deploy:sepolia   # hardhat run scripts/deploy.ts --network sepolia
npm run deploy:mainnet   # hardhat run scripts/deploy.ts --network mainnet
```

[`scripts/verify.ts`](../../blockchain/scripts/verify.ts) then submits both contracts to Etherscan. It doesn't take addresses as arguments. It re-parses them straight out of the `contractConfig.ts` that the deploy step just generated, so there's no risk of verifying the wrong address:

```ts
const content = fs.readFileSync(configPath, "utf8");
const loanMatch = content.match(/CryptoLoan:\s*"(0x[0-9a-fA-F]+)"/);
const myrMatch  = content.match(/MockMYR:\s*"(0x[0-9a-fA-F]+)"/);
const priceMatch = content.match(/ETH_PRICE_MYR\s*=\s*(\d+)/);

await run("verify:verify", {
  address: loanAddress,
  constructorArguments: [Number(ethPrice)],
  contract: "contracts/CryptoLoan.sol:CryptoLoan",
});
```

```bash
npm run verify:sepolia   # or verify:mainnet
```

```
Verifying CryptoLoan at: 0x5FbDB2315678afecb367f032d93F642f64180aa3
Successfully verified contract CryptoLoan on the block explorer.
CryptoLoan verified

Verifying MockMYR at: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
Successfully verified contract MockMYR on the block explorer.
MockMYR verified

All contracts verified on Etherscan.
```

![The verified CryptoLoan contract on Sepolia Etherscan, showing the green verified checkmark and readable source](asset/Deploy_EtherscanVerify.png)

*Figure 6.6: a verified contract on Etherscan. Verification publishes the source, so anyone can read the rules their money is subject to rather than trusting the bytecode.*

### 6.7 Post-Deploy Operations

Two operational scripts are meant to be run *after* a deployment, not as part of it.

**Pushing a fresh price.** [`scripts/set-price.ts`](../../blockchain/scripts/set-price.ts) pushes a new ETH/MYR price on-chain manually. It's the same underlying call the app's own price-sync job makes automatically via the owner key (see [9.6](#96-admin--oracle-functions) and Section 7.3's price self-healing note in the full document). It resolves the deployed `CryptoLoan` address from `contractConfig.ts` if `LOAN_ADDR` isn't set, then either uses `PRICE=<number>` or fetches a live CoinGecko quote:

```bash
PRICE=19500 npx hardhat run scripts/set-price.ts --network localhost
# or, to pull a live quote automatically:
npx hardhat run scripts/set-price.ts --network localhost
```

```
Current on-chain price: RM 18432
Setting price to: RM 19,500
✓ ETH price updated to RM 19,500
```

![The set-price script reporting the old and new on-chain ETH price](asset/Deploy_SetPrice.png)

*Figure 6.7.1: `set-price.ts`. Recall from Section 9 that `setEthPrice()` enforces a ±20% max move per call (`MAX_PRICE_CHANGE`), so a very stale on-chain price may need more than one call to reach the current market rate.*

**Fast-forwarding the chain clock.** Loans are fixed-term (30/90/180/365 days) with a 7-day grace period after the due date (§9.2), so demonstrating maturity or overdue liquidation would otherwise mean waiting months. [`scripts/time-travel.ts`](../../blockchain/scripts/time-travel.ts) advances the **local** Hardhat chain's clock instead, via `evm_increaseTime` + `evm_mine`:

```ts
const seconds = Math.round(days * 24 * 3600);
await ethers.provider.send("evm_increaseTime", [seconds]);
await ethers.provider.send("evm_mine", []);
```

```bash
npx hardhat run scripts/time-travel.ts --network localhost           # +31 days (default)
DAYS=98 npx hardhat run scripts/time-travel.ts --network localhost   # 90d term + 7d grace + 1d
```

```
Chain clock advanced by 98 day(s).
  before: 2026-08-07T01:12:44.000Z
  after:  2026-11-13T01:12:45.000Z
```

![The time-travel script advancing the local chain clock by 98 days](asset/Deploy_TimeTravel.png)

*Figure 6.7.2: `time-travel.ts` with `DAYS=98`, enough to push a 90-day loan past its due date and its 7-day grace period.*

Once the clock passes a loan's `dueDate + GRACE_PERIOD`, that loan reports as liquidatable and `liquidate()` accepts it on the overdue path (§9.8). The script's own header notes the caveat worth repeating in a demo: this pushes `block.timestamp` *ahead* of wall-clock time, and the web UI's client-side interest mirror still uses wall-clock, so on-chain figures read ahead of the UI's live estimates until real time catches up. That is expected on a demo chain, not a bug.

### 6.8 Optional: ICO Demo Deployment

The `/ico` page is a separate, self-contained token-sale demo, unrelated to the lending protocol. [`scripts/deploy-ico.ts`](../../blockchain/scripts/deploy-ico.ts) deploys `RinggitToken` (mints its full 100,000-token supply to the deployer) and `ICO` (priced so that 1 `MYRC` token costs the ETH-equivalent of RM 1, derived from the same live CoinGecko rate), then has the deployer `approve()` the `ICO` contract to sell the entire supply on its behalf:

```bash
npm run deploy:ico
```

```
Deploying ICO with account: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Rate: 1 ETH = RM 18,432 → 1 MYR = 0.000054253472222222 ETH
RinggitToken (MYR) deployed to: 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0
ICO            deployed to: 0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9
Price: 1 MYR = RM 1 = 0.000054253472222222 ETH
Owner approved ICO for: 100000.0 MYR

ICO config written to: .../web/src/lib/icoConfig.ts
```

![The ICO deploy script output, showing both deployed addresses and the owner approval](asset/Deploy_ICO.png)

*Figure 6.8: `npm run deploy:ico`. Note the approval step: the ICO contract never holds the supply, only permission to move it.*

This writes its own generated config, `web/src/lib/icoConfig.ts` (`ICO_ADDRESSES`, `ICO_PRICE_WEI`, ABIs), independent of `contractConfig.ts`. It is not required for the core deposit/borrow/repay flow.

### 6.9 Common Deployment Errors

Every message below is a real guard from the deploy tooling or the contract itself, not a hypothetical, so each is worth recognising on sight rather than re-diagnosing from scratch.

| Error                                                                                         | Where it appears                                                      | Fix                                                                                                                                                                           |
| --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Node is not fresh — deployer nonce is X, expected 0`                                         | `npm run deploy:local`, before any contract is touched (§6.5)         | Restart the Hardhat node (`npm run chain`) and deploy as the very first action against it, or re-run with `FORCE_DEPLOY=1` to accept a new `MockMYR` address                  |
| `contractConfig.ts not found — run deploy first`                                              | `npm run verify:sepolia` / `verify:mainnet`                           | Deploy to that network first (`npm run deploy:sepolia` / `deploy:mainnet`) so `contractConfig.ts` exists for `verify.ts` to read addresses from                               |
| `Could not parse contract addresses from contractConfig.ts`                                   | `verify.ts`                                                           | The generated file is missing or malformed; redeploy to regenerate it                                                                                                         |
| `Could not find CryptoLoan address in contractConfig.ts`                                      | `set-price.ts`, when `LOAN_ADDR` isn't set                            | Deploy first, or pass `LOAN_ADDR=0x...` explicitly                                                                                                                            |
| `Could not determine new price. Set PRICE=<number> env var or ensure CoinGecko is reachable.` | `set-price.ts`                                                        | Set `PRICE=<number>` manually, or check network access to CoinGecko                                                                                                           |
| `Invalid price`                                                                               | `CryptoLoan` constructor / `setEthPrice()` (contract-level `require`) | The price argument was `0`; pass a positive integer                                                                                                                           |
| `Price move too large`                                                                        | `setEthPrice()` (contract-level `require`, `MAX_PRICE_CHANGE`)        | A single call moved the price by more than 20%; step toward the target in smaller increments, the way `price-sync.ts` does automatically ([9.6](#96-admin--oracle-functions)) |
| `Rate exceeds cap`                                                                            | `setBaseRate()` (contract-level `require`, `MAX_BASE_RATE_BPS`)       | Requested `rateBps` exceeds 1500 (15%)                                                                                                                                        |
| `Cap below outstanding debt`                                                                  | `setSupplyCap()` (contract-level `require`)                           | The new `supplyCap` is under `totalBorrowed`; that would pin utilisation at 100% and every new borrow at the rate ceiling, with no way to lend out of it (§9.6)               |
| `DAYS must be a positive number, got "X"`                                                     | `time-travel.ts`                                                      | Pass a positive `DAYS=<number>`; the script defaults to 31 when unset                                                                                                         |

---
