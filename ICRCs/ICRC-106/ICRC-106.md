| Status |
|:------:|
|Draft|

# ICRC-106: Standard for Associating Index Canisters with ICRC-1 Tokens

## 1. Introduction

Wallet applications and token management tools often need to retrieve both token metadata and transaction history for a given principal. However, identifying an associated index canister for ICRC-1 tokens is currently unstandardized, leading to inconsistencies in wallet integrations.

**ICRC-106** introduces a standard approach for:
1. Indicating the presence of an index canister for ICRC-1 tokens through ledger metadata.
2. Defining a minimal interface for the index canister to facilitate querying transaction history in a consistent manner.

This standard aims to improve interoperability, simplify wallet integrations, and enable token-related applications to reliably access transaction histories.

---

## 2. Metadata

To indicate the presence of an index canister, ledgers implementing ICRC-106 MUST include the following metadata entry:

- **`icrc106:index_principal`**: Text entry representing the principal of the index canister associated with the ledger. This metadata entry enables programmatic discovery of the index canister’s principal by wallet applications and other clients.

### Optional Method

In addition, ledgers MAY provide an endpoint to retrieve the principal of the index canister:

```candid
icrc106_get_index_principal : () -> (opt principal) query
