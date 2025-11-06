# Internet Computer Token & Identity Standards Summary

A consolidated overview of Internet Computer token, signer, and identity standards.  
These specifications define how digital assets, wallets, and canisters interact securely and consistently across the ecosystem.

---

## 1. Tokens
Token standards define how digital assets are represented, transferred, and managed on the Internet Computer.  
They ensure interoperability between ledgers, wallets, and dapps while maintaining extensibility for features like approvals, metadata, and indexing.

---

### 1.1 Fungible Tokens
Fungible tokens represent interchangeable, divisible assets (like ICP, ckBTC, or other community tokens).  
These standards define the canonical ledger interface and extensions for approvals, allowances, and fee mechanisms.

<details>
<summary>View Fungible Token Standards</summary>

| Standard                                 | Summary                                                                                                                                                 |
|------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| **ICRC-1: Token Standard**               | The core standard for fungible tokens. Defines the ledger interface for balances, transfers, accounts, and metadata such as name, symbol, and decimals. |
| **ICRC-2: Approve & Transfer From**      | Adds ERC-20-style approval flows, allowing authorized accounts to move tokens on behalf of others.                                                      |
| **ICRC-103: List Outstanding Approvals** | Enables querying all active approvals for a given account.                                                                                              |
| **ICRC-107: Fee Collection**             | Specifies a standardized mechanism for ledgers to collect transfer fees.                                                                                |

</details>

---

### 1.2 Non-Fungible Tokens (NFTs)
Non-fungible tokens represent unique, indivisible assets (e.g., collectibles, game items, certificates).  
These standards define ownership, transferability, metadata structure, and permission systems.

<details>
<summary>View NFT Standards</summary>

| Standard                         | Summary                                                                                                                                |
|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| **ICRC-7: Minimal NFT Standard** | Establishes a minimal NFT interface (similar to ERC-721), covering ownership, transfers, and enumeration.                              |
| **ICRC-37: NFT Approvals**       | Extends ICRC-7 to enable granting transfer permissions to third parties.                                                               |
| **ICRC-97: NFT Metadata**        | Defines a consistent structure for NFT metadata — including name, description, and media — ensuring compatibility across marketplaces. |

</details>

---

### 1.3 Block Log
The block log standard defines how ledgers record, expose, and retrieve transactions in a consistent format.  
This supports explorers, analytics tools, and indexing canisters.

<details>
<summary>View Block Log Standards</summary>

| Standard              | Summary                                                                                                      |
|-----------------------|--------------------------------------------------------------------------------------------------------------|
| **ICRC-3: Block Log** | Standardizes how ledgers store and expose block and transaction data for external indexing and verification. |

</details>

---

### 1.4 Indexing
Indexing standards define how clients discover and query ledger index canisters that maintain token history and balance tracking.

<details>
<summary>View Indexing Standards</summary>

| Standard                             | Summary                                                                                                            |
|--------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| **ICRC-106: Legacy Index Discovery** | Allows modern clients to detect and use legacy index canisters for older ledgers, ensuring backward compatibility. |

</details>

---

### 1.5 Privileged Operations
These standards define administrative actions such as minting, freezing, and pausing ledgers.  
They ensure privileged operations are transparent, auditable, and structured in a uniform way.

<details>
<summary>View Privileged Operation Standards</summary>

#### Block Formats
| Standard                                         | Summary                                                                |
|--------------------------------------------------|------------------------------------------------------------------------|
| **ICRC-122: Mint & Burn Blocks**                 | Specifies block structures for token minting and burning operations.   |
| **ICRC-123: Freeze & Unfreeze Blocks**           | Defines how blocks represent account freezes and unfreezes.            |
| **ICRC-124: Pause, Unpause & Deactivate Blocks** | Details block formats for pausing, unpausing, or deactivating ledgers. |

#### Canister Interfaces
| Standard                                      | Summary                                                       |
|-----------------------------------------------|---------------------------------------------------------------|
| **ICRC-152: Mint & Burn API**                 | API for authorized token minting and burning.                 |
| **ICRC-153: Freeze & Unfreeze API**           | API methods for freezing or unfreezing token accounts.        |
| **ICRC-154: Pause, Unpause & Deactivate API** | API for controlling ledger operational states (pause/resume). |

</details>

---

## 2. Signer and Wallet Standards
Signer standards describe how wallets and dapps communicate securely.  
They define the request/response formats (via JSON-RPC), message transports, and consent interfaces that make wallet-based authentication and transaction signing safe and interoperable.

---

### 2.1 JSON-RPC
Defines the logical communication layer between wallets and dapps — how they connect, exchange requests, and perform signed canister calls.

<details>
<summary>View JSON-RPC Standards</summary>

| Standard                           | Summary                                                                                           |
|------------------------------------|---------------------------------------------------------------------------------------------------|
| **ICRC-25: Signer Interaction**    | Core interface for wallet–dapp communication. Defines connect, authorize, and sign message flows. |
| **ICRC-27: Accounts**              | Describes how wallets represent and expose user accounts and keys.                                |
| **ICRC-34: Delegation**            | Standardizes delegated signing (a wallet granting another agent permission to act).               |
| **ICRC-49: Call Canister**         | Allows dapps to request a wallet to call a canister on behalf of a user.                          |
| **ICRC-95: Derivation Origin**     | Defines consistent derivation paths for identities across different wallets.                      |
| **ICRC-112: Batch Call Canister**  | Enables multiple canister calls to be grouped and signed together.                                |
| **ICRC-146: Cross-Chain JSON-RPC** | Extends the standard for cross-chain wallet interaction and signing.                              |

</details>

---

### 2.2 Message Transports
Defines how wallet–dapp messages are transmitted securely across environments, such as browsers or extensions.

<details>
<summary>View Message Transport Standards</summary>

| Standard                                             | Summary                                                                                            |
|------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| **ICRC-29: Browser Post Message Transport**          | Uses the browser `postMessage` API to establish a secure channel between dapp and wallet contexts. |
| **ICRC-94: Browser Extension Discovery & Transport** | Describes discovery and communication for browser extension wallets.                               |

</details>

---

### 2.3 Canister Interfaces
These specify how wallets handle user consent, validation, and trusted origins when dapps interact with canisters.

<details>
<summary>View Canister Interface Standards</summary>

| Standard                                    | Summary                                                                               |
|---------------------------------------------|---------------------------------------------------------------------------------------|
| **ICRC-21: Canister Call Consent Messages** | Defines the format and flow for consent prompts when a dapp requests a canister call. |
| **ICRC-28: Trusted Origins**                | Specifies how canisters declare and validate which web origins are trusted.           |
| **ICRC-114: Validate Batch Call**           | Defines verification methods for multiple batched canister calls.                     |

</details>

---

### 2.4 Other / Deprecated Standards
<details>
<summary>View Deprecated Standards</summary>

| Standard                    | Summary                                                                                                       |
|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| **ICRC-32: Sign Challenge** | Early draft for cryptographic identity verification via signed challenges. Superseded by newer RPC standards. |

</details>

---

## 3. Chain-Agnostic Standards
These standards link the Internet Computer to broader multi-chain ecosystems by defining universal ways to identify chains and accounts.

<details>
<summary>View Chain-Agnostic Standards</summary>

| Standard                           | Summary                                                                                |
|------------------------------------|----------------------------------------------------------------------------------------|
| **CAIP-2: Blockchain Identifiers** | Defines unique blockchain identifiers (`<namespace>:<reference>`, e.g. `icp:mainnet`). |
| **CAIP-10: Account Addresses**     | Defines a universal format for account addresses across chains for wallets and dapps.  |

</details>

---

## 4. Examples and Usage

| Implementation                           | Standards Used                              | Description                                                                                                   |
|------------------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| **ICRC Ledger (ICRC-1 Ledger Canister)** | ICRC-1, ICRC-2, ICRC-3, ICRC-107            | Implements the main fungible token ledger (used for ICP and ckBTC) with transfers, approvals, and block logs. |
| **ICRC-7 NFT Ledger**                    | ICRC-7, ICRC-37, ICRC-97                    | Implements NFTs with metadata and approval features; used by marketplaces and art projects.                   |
| **OISY Wallet**                          | ICRC-25, ICRC-27, ICRC-49, ICRC-29, ICRC-28 | Example wallet implementing signer and JSON-RPC standards for secure dapp connections.                        |
| **Stoic / Plug / Bitfinity Wallets**     | ICRC-25, ICRC-27, ICRC-49, ICRC-94          | Browser extension wallets implementing discovery and transport standards for interoperability.                |
| **ICRC Index Canisters**                 | ICRC-3, ICRC-106                            | Provide indexed transaction history and balance tracking for explorers and analytics tools.                   |

---

## 5. Further Reading

- Token Standards: [dfinity/wg-token-standards](https://github.com/dfinity/wg-token-standards)
- Identity & Authentication Standards: [dfinity/wg-identity-authentication](https://github.com/dfinity/wg-identity-authentication)
- Forum context: [Token Standards WG Merge Announcement](https://forum.dfinity.org/t/token-standards-working-group-merging-ledger-and-token-and-nft-working-groups/39900)

---
