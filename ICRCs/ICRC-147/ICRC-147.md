# ICRC-147: Ledger Management API

## Purpose

The ICRC family of standards defines how ledger state is recorded in blocks (ICRC-3) and extended with privileged operations such as minting and burning (ICRC-122), freezing and unfreezing (ICRC-123), and pausing or deactivating ledgers (ICRC-124).  

**ICRC-147** standardizes the **canister interface methods** for invoking these privileged operations.  
It ensures that ledgers expose a **uniform, interoperable API** for lifecycle management, while still recording the effects of those operations in blocks as required by their respective standards.  

ICRC-147 includes:

- **Write methods** for minting, burning, freezing/unfreezing, and pausing/deactivating.  
- **Query methods** for retrieving ledger state, frozen entities, and configuration.  
- **Pagination** for queries returning collections, to support large ledgers.  

This API is intended for **controllers or authorized operators** of the ledger and MUST be secured appropriately.

---

## API

```candid
// ---- Minting & Burning  ----

type MintArgs = record {
    to: Account;
    amount: nat;
    created_at_time: nat64;
};
type BurnArgs = record {
    from: Account;
    amount: nat;
    created_at_time: nat64;
};

icrc147_mint : (MintArgs) -> (variant { Ok: nat; Err: MintError });
icrc147_burn : (BurnArgs) -> (variant { Ok: nat; Err: BurnError });


// ---- Freezing & Unfreezing ----

type FreezeAccountArgs = record {
    account: Account;
    reason: opt text;
    created_at_time: nat64;
};
type UnfreezeAccountArgs = record {
    account: Account;
    reason: opt text;
    created_at_time: nat64;
};

type FreezePrincipalArgs = record {
    principal: principal;
    reason: opt text;
    created_at_time: nat64;
};
type UnfreezePrincipalArgs = record {
    principal: principal;
    reason: opt text;
    created_at_time: nat64;
};

icrc147_freeze_account     : (FreezeAccountArgs)   -> (variant { Ok: nat; Err: FreezeError });
icrc147_unfreeze_account   : (UnfreezeAccountArgs) -> (variant { Ok: nat; Err: UnfreezeError });
icrc147_freeze_principal   : (FreezePrincipalArgs) -> (variant { Ok: nat; Err: FreezeError });
icrc147_unfreeze_principal : (UnfreezePrincipalArgs) -> (variant { Ok: nat; Err: UnfreezeError });

// Queries with lexicographic pagination
type FrozenAccountsRequest = record {
    start_after : opt Account;   // return accounts strictly greater than this
    max_results : nat;
};
type FrozenAccountsResponse = record {
    accounts : vec Account;
    has_more : bool;
};

icrc147_list_frozen_accounts : (FrozenAccountsRequest) -> (FrozenAccountsResponse) query;


type FrozenPrincipalsRequest = record {
    start_after : opt principal;   // return principals strictly greater than this
    max_results : nat;
};
type FrozenPrincipalsResponse = record {
    principals : vec principal;
    has_more : bool;
};

icrc147_list_frozen_principals : (FrozenPrincipalsRequest) -> (FrozenPrincipalsResponse) query;

// Boolean helpers for quick lookup
icrc147_is_frozen_account   : (Account) -> (bool) query;
icrc147_is_frozen_principal : (principal) -> (bool) query;


// ---- Ledger State Control ----

type PauseArgs = record {
    reason: opt text;
    created_at_time: nat64;
};
type UnpauseArgs = record {
    reason: opt text;
    created_at_time: nat64;
};
type DeactivateArgs = record {
    reason: opt text;
    created_at_time: nat64;
};

icrc147_pause      : (PauseArgs) -> (variant { Ok: nat; Err: PauseError });
icrc147_unpause    : (UnpauseArgs) -> (variant { Ok: nat; Err: UnpauseError });
icrc147_deactivate : (DeactivateArgs) -> (variant { Ok: nat; Err: DeactivateError });

// Queries
icrc147_is_paused      : () -> (bool) query;
icrc147_is_deactivated : () -> (bool) query;
```