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


## Common Elements

This standard inherits core conventions from **ICRC-3** (block log format, Value encoding, hashing, certification) and **ICRC-123** (typed block kinds and freeze/unfreeze semantics).

- **Accounts**  
  Encoded as ICRC-3 `Value` `variant { Array = vec { V1 [, V2] } }` where  
  `V1 = variant { Blob = <owner_principal_bytes> }` and optionally  
  `V2 = variant { Blob = <32-byte_subaccount_bytes> }`.  
  If no subaccount is provided, the array contains only the owner principal.

- **Principals**  
  Encoded as `variant { Blob = <principal_bytes> }`.

- **Timestamps**  
  Caller-supplied `created_at_time` is in **nanoseconds since Unix epoch**.  
  Encoded as `Nat` in ICRC-3 `Value` and **MUST** fit in `nat64`.

- **Blocks & Parent Hash**  
  All blocks created by this API use the ICRC-123 block kinds  
  (`btype = "123freeze_account"`, `"123unfreeze_account"`, `"123freeze_principal"`, `"123unfreeze_principal"`).  
  Standard metadata (`ts`, `phash`) follows ICRC-3.


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

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Mark the specified `account` as **frozen** according to ICRC-123 semantics.  
- Append a new block with `btype = "123freeze_account"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and encodings).  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where the `nat` is the index of the created block.  
- On failure: `variant { Err : FreezeAccountError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication (e.g., using `created_at_time`).  
- If a duplicate is detected, **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `InvalidAccount` — malformed or disallowed account (e.g., minting account, malformed principal/subaccount).  
- `AlreadyFrozen` — account already frozen.  
- `Duplicate { duplicate_of }` — semantically identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid block.


#### Canonical `tx` Mapping (normative)

| Field             | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|-------------------|------------------------|-------------------------|
| `op`              | `Text`                 | **Constant** `"153freeze_account"`. |
| `account`         | `Array` (Account)      | From `FreezeAccountArgs.account`, encoded as ICRC-3 Account. |
| `created_at_time` | `Nat`                  | From `FreezeAccountArgs.created_at_time` (ns since Unix epoch; **MUST** fit `nat64`). |
| `caller`          | `Blob`                 | Principal of the caller (raw bytes). |
| `reason`          | `Text` *(optional)*    | From `FreezeAccountArgs.reason` if provided; **omit** if absent. |


**Clarifications**  
- Optional fields **MUST be omitted** from `tx` if not supplied.  




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


## Query & Introspection Methods

This section defines read-only methods for discovering and checking the current
freeze state. These queries do **not** produce blocks and are required for
wallets, explorers, and auditors to efficiently enumerate frozen entities and
perform quick checks.

All results reflect the ledger’s state **at the time the query is executed**. These queries are read-only and do not produce blocks.


### `icrc153_list_frozen_accounts`

Lexicographically paginated listing of currently frozen **accounts**.

#### Arguments
```
type FrozenAccountsRequest = record {
    start_after : opt Account;   // return accounts strictly greater than this
    max_results : nat;
};
```
#### Returns
```
type FrozenAccountsResponse = record {
    accounts : vec Account;
    has_more : bool;
};

icrc153_list_frozen_accounts : (FrozenAccountsRequest) -> (FrozenAccountsResponse) query;
```


#### Semantics

- Returns up to `max_results` frozen accounts in **strictly increasing** lexicographic order.
- If `start_after` is present, the first returned account MUST be **strictly greater** than `start_after`.
- If `start_after` is absent, the listing starts from the smallest account.
- `has_more = true` iff there exist additional frozen accounts after the last one returned.

#### Canonical Ordering

Ordering MUST be defined over the ICRC-1 `Account` tuple `(owner, subaccount)` as follows:

1. Compare `owner` by its raw principal **bytes** (as if `variant { Blob = <bytes> }`), lexicographically.
2. If equal, compare `subaccount`, where:
   - an absent subaccount sorts **before** a present subaccount;
   - for two present subaccounts, compare their 32-byte values lexicographically.

This matches the natural `Value::Array`/`Blob` bytewise ordering implied by ICRC-3.

#### Notes

- `max_results` MAY be any non-negative `nat`. If `max_results = 0`, the method returns an empty `accounts` vector and a correct `has_more`.
- Implementations SHOULD enforce reasonable upper bounds for `max_results` to avoid excessive responses.
- Results are a **point-in-time snapshot**; concurrent freezes/unfreezes may affect subsequent pages.

### `icrc153_list_frozen_principals`

Lexicographically paginated listing of currently frozen **principals**.

#### Arguments
```
type FrozenPrincipalsRequest = record {
    start_after : opt principal;   // return principals strictly greater than this
    max_results : nat;
};
```
#### Returns
```
type FrozenPrincipalsResponse = record {
    principals : vec principal;
    has_more : bool;
};

icrc153_list_frozen_principals : (FrozenPrincipalsRequest) -> (FrozenPrincipalsResponse) query;
```

#### Semantics

- Returns up to `max_results` frozen principals in **strictly increasing** lexicographic order.
- If `start_after` is present, the first returned principal MUST be **strictly greater** than `start_after`.
- If `start_after` is absent, the listing starts from the smallest principal.
- `has_more = true` iff there exist additional frozen principals after the last one returned.

#### Canonical Ordering

Principals MUST be ordered by their raw principal **bytes** (as if `variant { Blob = <bytes> }`), lexicographically, consistent with ICRC-3 `Value` ordering.

#### Notes

- `max_results = 0` returns an empty `principals` vector with a correct `has_more`.
- Implementations SHOULD bound `max_results`.
- Results are a **point-in-time snapshot**; concurrent changes may affect subsequent pages.

## Effective Freeze Model (Clarification)

ICRC-153 adopts a **compositional freeze rule** consistent with ICRC-123:

- An **account** is *effectively frozen* if **either**:
  1) the account itself is frozen, **or**
  2) the account’s `owner` **principal** is frozen.

### Implications for Queries

- `icrc153_list_frozen_accounts`  
  Returns **only accounts frozen directly** (i.e., via account-level freezes).  
  It does **not** expand principal-level freezes into per-account entries.

- `icrc153_list_frozen_principals`  
  Returns **principals** that are frozen. Any account owned by a listed principal is
  *effectively frozen* by composition, even if it does not appear in the account list.

- `icrc153_is_frozen_account(account)` MUST return `true` if the account is directly frozen
  **or** if `is_frozen_principal(account.owner)` is `true`.

- `icrc153_is_frozen_principal(principal)` returns whether the **principal-level** freeze is active.

### Integrator Guidance

To determine whether a given account is currently frozen, integrators MUST either:
- call `icrc153_is_frozen_account(account)`, or
- check both lists and apply the compositional rule:
  1) see if `account` appears in `icrc153_list_frozen_accounts`, or
  2) see if `account.owner` appears in `icrc153_list_frozen_principals`.

This approach avoids large state updates when freezing/unfreezing principals with many accounts,
while providing a clear, deterministic interpretation of the effective freeze state.


## Example calls and the resulting blocks

#### Call
The caller invokes:

```
icrc153_freeze_account({
  account         = [principal "f5288412af11b299313a5b5a7c128311de102333c4adbe669f2ea1a308"];
  created_at_time = 1_753_500_100_000_000_000 : nat64;
  reason          = ?"Fraud investigation hold";
})
```

#### Resulting block

This call rsults in a block with btype="122freeze" and the following contents:

```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123freeze_account" } };
    record { "phash"; variant { Blob = blob "\12\34\56\78\9a\bc\de\f0\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe\11\22\33\44\55\66\77\88" } };
    record { "ts";    variant { Nat64 = 1_753_500_101_000_000_000 : nat64 } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "op";      variant { Text = "153freeze_account" } };
          record { "account"; variant { Array = vec {
            variant { Blob = blob "\15\28\84\12\af\11\b2\99\31\3a\5b\5a\7c\12\83\11\de\10\23\33\c4\ad\be\66\9f\2e\a1\a3\08" }
          } } };
          record { "ts";      variant { Nat64 = 1_753_500_100_000_000_000 : nat64 } };
          record { "caller";  variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason";  variant { Text  = "Fraud investigation hold" } };
        }
      };
    };
  }
};
```

The block records the operation (op = "153freeze_account"), the account being frozen (account), the caller-supplied timestamp (ts), the caller’s principal (caller), and the optional reason. The account is shown as a principal literal in the call, but stored canonically as a Blob in the block.

#### Call
The caller invokes:
```
icrc153_unfreeze_account({
  account         = [principal "f5288412af11b299313a5b5a7c128311de102333c4adbe669f2ea1a308"];
  created_at_time = 1_753_500_100_000_000_000 : nat64;
  reason          = ?"Lift compliance hold";
})
```


