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


## 2. Metadata

A ledger implementing ICRC-106 MUST include the following entry in the output of the `icrc1_supported_standards` method:

```candid
record { name = "ICRC-106"; url = "https://github.com/dfinity/ICRC-1/standards/ICRC-106" }
```

Additionally, the ledger MUST provide the following metadata entry retrievable via the `icrc1_metadata` method:

- `icrc106:index_principal` (text): The textual representation of the principal of the associated index canister.

These metadata entries allow clients to discover and interact with the index canister associated with a ledger and can be retrieved using method `icrc1_metadata` defined by the ICRC-1 standard.


Compliant ledgers MAY also implement the following optional endpoint for programmatically retrieving the index principal:

```candid
icrc106_get_index_principal: () -> (principal) query;
```

## 3. Index Canister Interface

The index canister associated with the ledger is expected to implement the following minimal Candid interface:

```candid
type Tokens = nat;

type BlockIndex = nat;

type SubAccount = blob;

type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type GetAccountTransactionsArgs = record {
    account: Account;
    start: opt nat; // The block index of the last transaction seen by the client.
    max_results: nat; // Maximum number of transactions to fetch.
};

type Transaction = record {
    burn: opt record { from: Account; amount: nat; };
    transfer: opt record { from: Account; to: Account; amount: nat; };
    approve: opt record { from: Account; spender: Account; amount: nat; };
    timestamp: nat64; // Timestamp of the transaction.
};

type TransactionWithId = record {
    id: nat; // Block index of the transaction.
    transaction: Transaction;
};

type GetTransactionsResult = variant {
    Ok: record {
        balance: Tokens; // Current balance of the account.
        transactions: vec TransactionWithId; // List of transactions.
    };
    Err: record { message: text; };
};

type ListSubaccountsArgs = record {
    owner: principal;
    start: opt SubAccount;
};

type Status = record {
    num_blocks_synced : BlockIndex;
};

service : {
    get_account_transactions: (GetAccountTransactionsArgs) -> (GetTransactionsResult) query;
    icrc1_balance_of: (Account) -> (Tokens) query;
    ledger_id: () -> (principal) query;
    list_subaccounts : (ListSubaccountsArgs) -> (vec SubAccount) query;
    status : () -> (Status) query;
}
```

This interface defines the minimal set of methods that index canisters must support for integration.

## 4. Implementation Considerations

Implementers should ensure that:
- The `icrc106:index_principal` metadata entry accurately reflects the principal of the associated index canister.
- The `icrc106:index_canister_interface` metadata entry points to a location where the Candid interface definition of the index canister is accessible.
- Any changes to the index canister interface should maintain backward compatibility.

By adhering to ICRC-106, ledger canisters provide a standardized mechanism for clients to discover and interact with their associated index canisters, improving integration and user experience within the Internet Computer ecosystem.
