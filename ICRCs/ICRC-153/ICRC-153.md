# ICRC-153: Privileged Freeze & Unfreeze API

| Status |
|:------:|
| Draft  |

## Introduction & Motivation

Operational and regulatory requirements for custodial tokens, stablecoins, and RWA ledgers often include the ability to **freeze** (temporarily disable) or **unfreeze** transfer activity for specific **accounts** or for all accounts belonging to a **principal**. Absent a standard interface, integrators cannot reliably determine whether funds are movable, nor attribute freezes to clear, on-chain actions.

ICRC-153 addresses this by standardizing four privileged methods that append canonical, typed blocks (as defined in **ICRC-123**) and by defining the canonical mapping from method inputs to the `tx` field for those blocks.

**Privileged methods (authorized principals only):**
- `icrc153_freeze_account`, `icrc153_unfreeze_account`
- `icrc153_freeze_principal`, `icrc153_unfreeze_principal`

**Canonical `tx` mapping** with namespaced `op` values (`"153..."`), caller identity, and optional human-readable reason.

Recording uses **ICRC-123** block kinds (this standard does **not** add new block types).

## Overview

ICRC-153 standardizes privileged freeze/unfreeze controls for ICRC ledgers.

Specifically, it defines:

- **APIs** to freeze/unfreeze **accounts** and **principals**, callable only by authorized entities.
- **Canonical `tx` mapping** rules ensuring deterministic block content and easy attribution/auditing.
- **Use of ICRC-123 block kinds** to record these actions:
  - `btype = "123freeze_account"`, `btype = "123unfreeze_account"`
  - `btype = "123freeze_principal"`, `btype = "123unfreeze_principal"`
- **Compliance reporting** through ICRC-10 methods.

This enables wallets, explorers, and auditors to:

- Determine, on-chain, whether an account or a principal is currently frozen.
- Attribute actions to a specific authorized caller and (optionally) a recorded reason.
- Interoperate across ledgers that implement the same API and block semantics.

## Dependencies

This standard does not introduce new block kinds.

- **ICRC-3** — Block log format, hashing, certification, and canonical `tx` mapping rules.
- **ICRC-123** — Defines the typed block kinds that ICRC-153 uses:
  - `btype = "123freeze_account"`
  - `btype = "123unfreeze_account"`
  - `btype = "123freeze_principal"`
  - `btype = "123unfreeze_principal"`

A ledger implementing ICRC-153 MUST:
- Emit the appropriate **ICRC-123** block on each successful call.
- Populate `tx.op` with namespaced values **introduced by this standard**:
  `"153freeze_account"`, `"153unfreeze_account"`, `"153freeze_principal"`, `"153unfreeze_principal"`.

## Methods

### `icrc153_freeze_account`

Freeze a specific **account** (owner + optional subaccount). Once frozen, operations affected by ICRC-123’s semantics MUST be rejected until unfrozen.

#### Arguments

```
type FreezeAccountArgs = record {
  account         : Account;
  created_at_time : nat64;
  reason          : opt text;
};

type FreezeAccountError = variant {
  Unauthorized : text;               // caller not permitted
  InvalidAccount : text;             // malformed or disallowed account
  AlreadyFrozen : text;              // account is already frozen
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_freeze_account : (FreezeAccountArgs) -> (variant { Ok : nat; Err : FreezeAccountError });
```
#### Semantics

- Marks the account as **frozen** according to ICRC-123 semantics.  
- Appends a block of type `123freeze_account`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the freeze semantics defined by ICRC-123.  

#### Return Values

On success:  

- `variant { Ok : nat }` — the created block index.  

On failure:  

- `variant { Err : FreezeAccountError }`.  

#### Canonical `tx` Mapping

A successful call to `icrc153_freeze_account` produces a `123freeze_account` block.  
The `tx` field is derived as follows:

- `op      = "153freeze_account"`  
- `account = FreezeAccountArgs.account`  
- `ts      = FreezeAccountArgs.created_at_time`  
- `caller  = caller_principal (as Blob)`  
- `reason  = FreezeAccountArgs.reason` (if provided)  

Optional fields MUST be omitted if not supplied.  


### `icrc153_unfreeze_account`

Unfreeze a specific account previously frozen.

#### Arguments

```
type UnfreezeAccountArgs = record {
  account         : Account;
  created_at_time : nat64;
  reason          : opt text;
};

type UnfreezeAccountError = variant {
  Unauthorized : text;
  InvalidAccount : text;
  NotFrozen : text;                  // account is not currently frozen
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_unfreeze_account : (UnfreezeAccountArgs) -> (variant { Ok : nat; Err : UnfreezeAccountError });
```

#### Semantics

- Marks account as **unfrozen** according to ICRC-123 semantics.  
- Appends a block of type `123unfreeze_account`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the unfreeze semantics defined by ICRC-123.  

#### Return Values

On success:  

- `variant { Ok : nat }` — the created block index.  

On failure:  

- `variant { Err : UnfreezeAccountError }`.  

#### Canonical `tx` Mapping

A successful call to `icrc153_unfreeze_account` produces a `123unfreeze_account` block.  
The `tx` field is derived as follows:

- `op      = "153unfreeze_account"`  
- `account = UnfreezeAccountArgs.account`  
- `ts      = UnfreezeAccountArgs.created_at_time`  
- `caller  = caller_principal (as Blob)`  
- `reason  = UnfreezeAccountArgs.reason` (if provided)  

Optional fields MUST be omitted if not supplied.


### `icrc153_freeze_principal`

Freeze all accounts belonging to a given principal per ICRC-123 semantics.

#### Arguments

```
type FreezePrincipalArgs = record {
  principal       : principal;
  created_at_time : nat64;
  reason          : opt text;
};

type FreezePrincipalError = variant {
  Unauthorized : text;
  InvalidPrincipal : text;
  AlreadyFrozen : text;              // principal is already frozen at scope defined by ICRC-123
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_freeze_principal : (FreezePrincipalArgs) -> (variant { Ok : nat; Err : FreezePrincipalError });
```

#### Semantics

- Marks the **principal** as frozen (scope per ICRC-123), affecting its accounts.  
- Appends a block of type `123freeze_principal`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the freeze semantics defined by ICRC-123.  

#### Return Values

On success:  

- `variant { Ok : nat }` — the created block index.  

On failure:  

- `variant { Err : FreezePrincipalError }`.  

#### Canonical `tx` Mapping

A successful call to `icrc153_freeze_principal` produces a `123freeze_principal` block.  
The `tx` field is derived as follows:

- `op        = "153freeze_principal"`  
- `principal = caller-supplied principal (as Blob)`  
  (in ICRC-3 Value, principals are encoded as `Blob`)  
- `ts        = FreezePrincipalArgs.created_at_time`  
- `caller    = caller_principal (as Blob)`  
- `reason    = FreezePrincipalArgs.reason` (if provided)  

Optional fields MUST be omitted if not supplied.  


### `icrc153_unfreeze_principal`

Lift the freeze affecting a principal (and the scope of accounts specified by ICRC-123).

#### Arguments

```
type UnfreezePrincipalArgs = record {
  principal       : principal;
  created_at_time : nat64;
  reason          : opt text;
};

type UnfreezePrincipalError = variant {
  Unauthorized : text;
  InvalidPrincipal : text;
  NotFrozen : text;                  // principal is not currently frozen
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_unfreeze_principal : (UnfreezePrincipalArgs) -> (variant { Ok : nat; Err : UnfreezePrincipalError });
```

#### Semantics

- Marks the principal as **unfrozen**, lifting restrictions according to ICRC-123 semantics.  
- Appends a block of type `123unfreeze_principal`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the unfreeze semantics defined by ICRC-123.  

#### Return Values

On success:  

- `variant { Ok : nat }` — the created block index.  

On failure:  

- `variant { Err : UnfreezePrincipalError }`.  

#### Canonical `tx` Mapping

A successful call to `icrc153_unfreeze_principal` produces a `123unfreeze_principal` block.  
The `tx` field is derived as follows:

- `op        = "153unfreeze_principal"`  
- `principal = UnfreezePrincipalArgs.principal` (encoded as Blob per ICRC-3 Value)  
- `ts        = UnfreezePrincipalArgs.created_at_time`  
- `caller    = caller_principal (as Blob)`  
- `reason    = UnfreezePrincipalArgs.reason` (if provided)  

Optional fields MUST be omitted if not supplied.

## Notes on Semantics & Scope

- **Authorizations**: Each method is privileged; ledgers MUST restrict these calls to authorized principals (policy/RBAC is ledger-defined or standardized separately).
- **Deduplication**: `created_at_time` is used for replay protection/deduplication (as in ICRC-1). Duplicate calls MUST return `Duplicate { duplicate_of = <block_index> }` and MUST NOT produce a new block.
- **Effect on operations**: The precise effects of freeze/unfreeze (which operations are blocked, account vs principal precedence, etc.) are entirely defined by ICRC-123. ICRC-153 only defines the interface and canonical `tx` mapping.
- **No fees**: ICRC-153 calls do not involve fees; ledgers MUST NOT include a top-level `fee` field for these blocks.

## Reporting Compliance

### Supported Standards

Ledgers implementing ICRC-153 MUST indicate compliance through the
`icrc1_supported_standards` and `icrc10_supported_standards` methods by
including:

```
variant { Vec = vec {
  record {
    "name"; variant { Text = "ICRC-153" };
    "url";  variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-153.md" };
  }
}};
```

### Supported Block Types

ICRC-153 extends ICRC-123 and does not introduce new block kinds.
Accordingly, ledgers implementing ICRC-153 MUST already advertise support for the
relevant ICRC-123 block kinds (e.g., `123freeze_account`, `123unfreeze_account`,
`123freeze_principal`, `123unfreeze_principal`) as required by ICRC-123.
No additional block types need to be reported.
