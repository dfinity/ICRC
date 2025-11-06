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

## 2. Signer
Signer standards describe how signers (e.g. wallets) and dapps communicate securely.  
They define the request/response formats (via JSON-RPC), message transports, and consent interfaces that make signer-based authentication and transaction signing safe and interoperable.

---

### 2.1 JSON-RPC
Defines the logical communication layer between signers and dapps: how they connect, exchange requests, and perform signed canister calls.

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

## 3. Chain-Agnostic
These standards link the Internet Computer to broader multichain ecosystems by defining universal ways to identify chains and accounts.

<details>
<summary>View Chain-Agnostic Standards</summary>

| Standard                           | Summary                                                                                |
|------------------------------------|----------------------------------------------------------------------------------------|
| **CAIP-2: Blockchain Identifiers** | Defines unique blockchain identifiers (`<namespace>:<reference>`, e.g. `icp:mainnet`). |
| **CAIP-10: Account Addresses**     | Defines a universal format for account addresses across chains for wallets and dapps.  |

</details>

---

## 4. Examples

This section illustrates how Internet Computer standards are applied in production-grade systems.  
Each example demonstrates how multiple ICRC standards interconnect to provide consistent, secure, and extensible functionality.

---

### 4.1 ICRC Ledger — Canonical Fungible Token Implementation

The **ICRC Ledger canister** is the Internet Computer’s canonical implementation for fungible tokens, used for assets such as **ICP**, **ckBTC**, and numerous community tokens.

At its foundation, the ledger implements **ICRC-1**, which defines the basic ledger interface for account handling, transfers, and metadata.  
Building on this, **ICRC-2** introduces delegated transfer capability — enabling workflows like escrow, token marketplaces, and automated treasury management via the `approve` and `transfer_from` methods.

To maintain a verifiable record of all transactions, the ledger uses **ICRC-3**, which specifies the **block log** structure. This allows explorers and third-party services to retrieve transaction data consistently across ledgers.  
Finally, **ICRC-107** ensures that transaction fees are collected and reported in a standardized way, making it easier to account for network or ledger-specific costs.

Together, these standards make the ICRC Ledger a modular, secure, and auditable foundation for all fungible tokens on the Internet Computer.

---

### 4.2 ICRC Index Canisters — Transaction History and Ledger Discovery

**Index canisters** provide efficient read-only access to ledger activity, allowing users and tools to query transactions without directly interacting with the main ledger.

They rely on **ICRC-3** to interpret and expose block log data, enabling lightweight transaction history queries.  
To remain backward-compatible with earlier implementations, **ICRC-106** defines how these index canisters can detect and link to **legacy ledgers**.

In practice, index canisters make it possible for explorers, dashboards, and analytical services to track balances, transfers, and events efficiently across the Internet Computer network.

---

### 4.3 ORIGYN NFT Framework — Implementation of ICRC-7 NFT Standards
**Repository:** [ORIGYN-SA/nft](https://github.com/ORIGYN-SA/nft)

The **ORIGYN NFT framework** is a full-fledged implementation of the Internet Computer’s non-fungible token standards, demonstrating how **ICRC-7**, **ICRC-37**, and **ICRC-97** integrate to create interoperable, metadata-rich digital assets.

At its core, **ICRC-7** defines the minimal NFT interface for ownership, transfer, and enumeration. This ensures that any dapp, wallet, or marketplace can interact with NFTs in a consistent way.  
Building on that, **ICRC-37** adds an approval model, allowing marketplaces or custodial agents to transfer NFTs on behalf of users with explicit consent — a crucial component for trust-based exchanges.  
Finally, **ICRC-97** introduces a standardized metadata schema that includes names, descriptions, media URIs, and traits. This metadata ensures NFTs are portable and renderable across different apps and wallets.

The ORIGYN framework brings these standards together into a cohesive canister-based implementation, used by NFT marketplaces, art projects, and digital identity systems to manage verifiable, tamper-proof assets on-chain.  
It exemplifies how the ICRC standards can serve as a foundation for complex applications while remaining open and composable.

---

### 4.4 OISY Wallet — Web-Based Signer Using JSON-RPC

The **OISY wallet** implements Internet Computer signer standards to securely connect web dapps with user identities.  
It operates entirely through a JSON-RPC communication model that standardizes the way dapps request signatures and perform canister calls.

This interaction is governed by **ICRC-25**, which defines the request and response structure for connecting, authorizing, and signing.  
**ICRC-27** specifies how wallets expose accounts and manage switching between them, while **ICRC-49** extends these interactions to include canister calls performed on behalf of users.

For secure browser communication, **ICRC-29** defines how messages are exchanged using the browser’s `postMessage` API, and **ICRC-28** governs trusted origins — ensuring only approved websites can request user interaction.

By adhering to these standards, OISY provides a web-native, interoperable wallet experience that can interact seamlessly with any compliant Internet Computer dapp.

---

### 4.5 Plug Wallet — Browser Extension Signer

**Plug Wallet** implements the same JSON-RPC and signer architecture as OISY but within a **browser extension environment**, allowing persistent, sandboxed access for dapps.

It leverages **ICRC-25** for RPC-based wallet interaction, **ICRC-27** for account management, and **ICRC-49** for executing signed canister calls.  
Unlike web-based wallets, Plug follows **ICRC-94**, which defines how browser extensions are discovered by webpages and how secure transport channels are established for message passing.

This enables Plug to interoperate seamlessly with the same dapps that use OISY or other compliant wallets, ensuring a unified and secure user experience across browser and web environments.

---