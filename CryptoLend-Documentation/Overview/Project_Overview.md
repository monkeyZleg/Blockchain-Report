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