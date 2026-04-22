# ICRC-153: Privileged Freeze & Unfreeze API

| Status |
|:------:|
| Draft  |

## Introduction

Operational and regulatory requirements for custodial tokens, stablecoins, and RWA ledgers often require the ability to freeze or unfreeze transfer activity for specific accounts or principals. Absent a standard interface, integrators cannot reliably determine whether funds are movable, nor attribute freezes to clear, on-chain actions.

ICRC-153 standardizes this by defining four privileged methods that append canonical, typed blocks to the ledger (as defined in **ICRC-123**):

- **`icrc153_freeze_account`** / **`icrc153_unfreeze_account`** — freeze or unfreeze a specific account.
- **`icrc153_freeze_principal`** / **`icrc153_unfreeze_principal`** — freeze or unfreeze all accounts belonging to a principal.

Two query methods (`icrc153_list_frozen_accounts`, `icrc153_list_frozen_principals`) and two status checks (`icrc153_is_frozen_account`, `icrc153_is_frozen_principal`) allow wallets, explorers, and auditors to determine freeze state on-chain and attribute actions to specific authorized callers.

Recording uses **ICRC-123** block kinds; this standard does not add new block types.

## Dependencies

This standard does not introduce new block kinds.

- **ICRC-3** — Block log format, hashing, certification, and canonical `tx` mapping rules.
- **ICRC-123** — Defines the typed block kinds that ICRC-153 uses:
  - `btype = "123freezeaccount"`
  - `btype = "123unfreezeaccount"`
  - `btype = "123freezeprincipal"`
  - `btype = "123unfreezeprincipal"`

A ledger implementing ICRC-153 MUST:
- Emit the appropriate **ICRC-123** block on each successful call.
- Populate `tx.mthd` with namespaced values **introduced by this standard**:
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
  The caller-supplied timestamp is recorded as `ts` in the block's `tx` map, in **nanoseconds since Unix epoch**, encoded as `Nat` in ICRC-3 `Value` and **MUST** fit in `nat64`.

- **Blocks & Parent Hash**  
  All blocks created by this API use the ICRC-123 block kinds  
  (`btype = "123freezeaccount"`, `"123unfreezeaccount"`, `"123freezeprincipal"`, `"123unfreezeprincipal"`).  
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
  AlreadyFrozen   : text;              // account is already frozen
  TooOld;
  CreatedInFuture  : record { ledger_time : nat64 };
  Duplicate        : record { duplicate_of : nat };
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
- Append a new block with `btype = "123freezeaccount"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and encodings).  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where the `nat` is the index of the created block.  
- On failure: `variant { Err : FreezeAccountError }`.

**Deduplication & idempotency**  
- `created_at_time` is required (not optional) because deduplication is always mandatory for privileged operations — accidental duplicate freeze/unfreeze actions must be prevented regardless of caller intent.  
- The ledger **MUST** perform deduplication using `created_at_time` and any implementation-defined inputs.  
- If a duplicate is detected, **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `InvalidAccount` — malformed or disallowed account (e.g., minting account, malformed principal/subaccount).  
- `AlreadyFrozen` — account already frozen.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**  
- Optional fields **MUST be omitted** from `tx` if not supplied.  


#### Canonical `tx` Mapping (normative)

| Field             | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|-------------------|------------------------|-------------------------|
| `mthd`            | `Text`                 | **Constant** `"153freeze_account"`. |
| `account`         | `Array` (Account)      | From `FreezeAccountArgs.account`, encoded as ICRC-3 Account. |
| `ts`              | `Nat`                  | From `FreezeAccountArgs.created_at_time` (ns since Unix epoch; **MUST** fit `nat64`). |
| `caller`          | `Blob`                 | Principal of the caller (raw bytes). |
| `reason`          | `Text` *(optional)*    | From `FreezeAccountArgs.reason` if provided; **omit** if absent. |





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
  Unauthorized    : text;
  InvalidAccount  : text;
  TooOld;
  CreatedInFuture : record { ledger_time : nat64 };
  Duplicate       : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_unfreeze_account : (UnfreezeAccountArgs) -> (variant { Ok : nat; Err : UnfreezeAccountError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Mark the specified `account` as **unfrozen** according to ICRC-123 semantics.  
- Append a new block with `btype = "123unfreezeaccount"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts` derived from `created_at_time`.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where the `nat` is the index of the created block.  
- On failure: `variant { Err : UnfreezeAccountError }`.

**Deduplication & idempotency**  
- `created_at_time` is required (not optional) because deduplication is always mandatory for privileged operations — accidental duplicate freeze/unfreeze actions must be prevented regardless of caller intent.  
- The ledger **MUST** perform deduplication using `created_at_time` and any implementation-defined inputs.  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**
- `Unauthorized` — caller not permitted.
- `InvalidAccount` — malformed or disallowed account.
- `TooOld` — `created_at_time` is before the ledger's deduplication window.
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.
- `Duplicate { duplicate_of }` — identical transaction previously accepted.
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**
- The call MUST succeed even if the account is not currently frozen at account level. Under ICRC-123's latest-action-wins rule, recording an unfreeze block can lift freeze blocks at other levels (e.g., account-level freezes lifted by a principal unfreeze).
- The `tx` field uses **`ts`** for the caller-supplied timestamp (`created_at_time`).
- Optional fields **MUST be omitted** from `tx` if not supplied.
- Representation-independent hashing (ICRC-3) applies; field presence and value determine hash, not field order.

#### Canonical `tx` Mapping (normative)

| Field | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|---|---|---|
| `mthd` | `Text` | **Constant** `"153unfreeze_account"`. |
| `account` | `Array` (Account) | From `UnfreezeAccountArgs.account`. |
| `ts` | `Nat` | From `UnfreezeAccountArgs.created_at_time`. |
| `caller` | `Blob` | Principal of the caller. |
| `reason` | `Text` *(optional)* | From `UnfreezeAccountArgs.reason` if provided; **omit** if absent. |


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
  AlreadyFrozen    : text;              // principal is already frozen at scope defined by ICRC-123
  TooOld;
  CreatedInFuture  : record { ledger_time : nat64 };
  Duplicate        : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_freeze_principal : (FreezePrincipalArgs) -> (variant { Ok : nat; Err : FreezePrincipalError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Mark the specified `principal` as **frozen** (scope and effect per ICRC-123), impacting its accounts via composition.  
- Append a new block with `btype = "123freezeprincipal"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts` derived from `created_at_time`.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where the `nat` is the index of the created block.  
- On failure: `variant { Err : FreezePrincipalError }`.

**Deduplication & idempotency**  
- `created_at_time` is required (not optional) because deduplication is always mandatory for privileged operations — accidental duplicate freeze/unfreeze actions must be prevented regardless of caller intent.  
- The ledger **MUST** perform deduplication using `created_at_time` and any implementation-defined inputs.  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `InvalidPrincipal` — malformed/invalid principal bytes.  
- `AlreadyFrozen` — principal already frozen at the scope defined by ICRC-123.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**  
- The `tx` field uses **`ts`** for the caller-supplied timestamp (`created_at_time`).  
- Optional fields **MUST be omitted** from `tx` if not supplied.  
- Representation-independent hashing (ICRC-3) applies; field presence and value determine hash, not field order.

#### Canonical `tx` Mapping (normative)

| Field | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|---|---|---|
| `mthd` | `Text` | **Constant** `"153freeze_principal"`. |
| `principal` | `Blob` | From `FreezePrincipalArgs.principal` (principal bytes). |
| `ts` | `Nat` | From `FreezePrincipalArgs.created_at_time`. |
| `caller` | `Blob` | Principal of the caller. |
| `reason` | `Text` *(optional)* | From `FreezePrincipalArgs.reason` if provided; **omit** if absent. |




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
  Unauthorized     : text;
  InvalidPrincipal : text;
  TooOld;
  CreatedInFuture  : record { ledger_time : nat64 };
  Duplicate        : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc153_unfreeze_principal : (UnfreezePrincipalArgs) -> (variant { Ok : nat; Err : UnfreezePrincipalError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Mark the specified `principal` as **unfrozen** (lifting restrictions per ICRC-123).  
- Append a new block with `btype = "123unfreezeprincipal"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts` derived from `created_at_time`.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where the `nat` is the index of the created block.  
- On failure: `variant { Err : UnfreezePrincipalError }`.

**Deduplication & idempotency**  
- `created_at_time` is required (not optional) because deduplication is always mandatory for privileged operations — accidental duplicate freeze/unfreeze actions must be prevented regardless of caller intent.  
- The ledger **MUST** perform deduplication using `created_at_time` and any implementation-defined inputs.  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**
- `Unauthorized` — caller not permitted.
- `InvalidPrincipal` — malformed/invalid principal bytes.
- `TooOld` — `created_at_time` is before the ledger's deduplication window.
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.
- `Duplicate { duplicate_of }` — identical transaction previously accepted.
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**
- The call MUST succeed even if the principal is not currently frozen at principal level. Under ICRC-123's latest-action-wins rule, recording an unfreeze block can lift freeze blocks at other levels (e.g., account-level freezes lifted by a principal unfreeze).
- The `tx` field uses **`ts`** for the caller-supplied timestamp (`created_at_time`).
- Optional fields **MUST be omitted** from `tx` if not supplied.
- Representation-independent hashing (ICRC-3) applies; field presence and value determine hash, not field order.

#### Canonical `tx` Mapping (normative)

| Field | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|---|---|---|
| `mthd` | `Text` | **Constant** `"153unfreeze_principal"`. |
| `principal` | `Blob` | From `UnfreezePrincipalArgs.principal`. |
| `ts` | `Nat` | From `UnfreezePrincipalArgs.created_at_time`. |
| `caller` | `Blob` | Principal of the caller. |
| `reason` | `Text` *(optional)* | From `UnfreezePrincipalArgs.reason` if provided; **omit** if absent. |


## Notes on Semantics & Scope

- **Authorizations**: Each method is privileged; ledgers MUST restrict these calls to authorized principals (policy/RBAC is ledger-defined or standardized separately).
- **Deduplication**: `created_at_time` is used for replay protection/deduplication (as in ICRC-1). Duplicate calls MUST return `Duplicate { duplicate_of = <block_index> }` and MUST NOT produce a new block.
- **Effect on operations**: The precise effects of freeze/unfreeze (which operations are blocked, account vs principal precedence, etc.) are entirely defined by ICRC-123. ICRC-153 only defines the interface and canonical `tx` mapping.
- **No fees**: ICRC-153 calls do not involve fees; ledgers MUST NOT include a top-level `fee` field for these blocks.

## Compliance Reporting

### Supported Standards

Ledgers implementing ICRC-153 MUST indicate compliance through the
`icrc10_supported_standards` method by including in its output:

```candid
vec {
    record {
        name = "ICRC-153";
        url  = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-153/ICRC-153.md";
    }
}
```

Ledgers that also implement ICRC-1 MUST additionally include this entry in
the output of `icrc1_supported_standards`.

### Supported Block Types

ICRC-153 extends ICRC-123 and does not introduce new block kinds.
Accordingly, ledgers implementing ICRC-153 MUST already advertise support for the
relevant ICRC-123 block kinds (e.g., `123freezeaccount`, `123unfreezeaccount`,
`123freezeprincipal`, `123unfreezeprincipal`) as required by ICRC-123.
No additional block types need to be reported.


## Query & Introspection Methods

This section defines read-only methods for discovering and checking the current
freeze state. These queries do **not** produce blocks and are required for
wallets, explorers, and auditors to efficiently enumerate frozen entities and
perform quick checks.

All results reflect the ledger’s state **at the time the query is executed**. These queries are read-only and do not produce blocks.


### `icrc153_list_frozen_accounts`

Lexicographically paginated listing of accounts that are **currently effectively frozen at the account level** — i.e., an account `acc` is listed iff both of the following hold:

1. The most recent `123freezeaccount`/`123unfreezeaccount` block targeting `acc` is a freeze, and
2. No `123unfreezeprincipal` block on `acc.owner` is more recent than the account-level freeze identified in (1).

Every account returned by this method **MUST** satisfy `icrc153_is_frozen_account(acc) = true`.

> ⚠️ **Not exhaustive of effectively frozen accounts.** Accounts that are effectively frozen *solely* because their owner principal is frozen (via `123freezeprincipal`, with no account-level freeze) are **not** listed here. To enumerate principal-level freezes, use `icrc153_list_frozen_principals`. To check whether a specific account is effectively frozen, use `icrc153_is_frozen_account`. See **Effective Freeze Model** below.

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

### `icrc153_is_frozen_account`

Returns whether a given account is currently effectively frozen.

```
icrc153_is_frozen_account : (Account) -> (bool) query;
```

Returns `true` iff the account is *effectively frozen* under ICRC-123's latest-action-wins rule,
considering both account-level and principal-level actions.

### `icrc153_is_frozen_principal`

Returns whether a given principal is currently frozen at the principal level.

```
icrc153_is_frozen_principal : (principal) -> (bool) query;
```

Returns `true` iff the most recent **principal-level** action for the given principal was a freeze.
Does not consider account-level actions.

## Effective Freeze Model (Clarification)

ICRC-153 adopts the **latest-action-wins** rule defined in ICRC-123:

- An **account** `acc = (owner, subaccount)` is *effectively frozen* iff the most
  recent freeze/unfreeze block that **affects** `acc` is a freeze block.
- A block **affects** `acc` if:
  1) it is a `123freezeaccount` or `123unfreezeaccount` block where `tx.account` matches `acc`, **or**
  2) it is a `123freezeprincipal` or `123unfreezeprincipal` block where `tx.principal` equals `owner`.
- If no block affects `acc`, the account is **not frozen**.

### Key implications

- Unfreezing a **principal** lifts all earlier account-level freezes for that
  principal’s accounts (since the principal unfreeze is more recent).
- Freezing an **account** after unfreezing its principal re-freezes only that
  specific account.
- The effective status of any account is always determined by comparing the
  block index of the latest account-level action (if any) with the latest
  principal-level action (if any); whichever is more recent wins.

### Implications for Queries

- `icrc153_list_frozen_accounts`
  Returns accounts that are effectively frozen **by an account-level freeze**:
  the most recent `123freezeaccount`/`123unfreezeaccount` block targeting the
  account is a freeze, **and** no later `123unfreezeprincipal` on the owner
  has superseded it. It does **not** expand principal-level freezes into
  per-account entries (so accounts frozen only via a principal-level freeze
  are excluded), but every returned account is guaranteed to satisfy
  `icrc153_is_frozen_account = true`.

- `icrc153_list_frozen_principals`
  Returns principals whose most recent **principal-level** action was a freeze.

- `icrc153_is_frozen_account(account)` MUST return `true` iff the account is
  *effectively frozen* under the latest-action-wins rule (considering both
  account-level and principal-level actions).

- `icrc153_is_frozen_principal(principal)` returns whether the most recent
  **principal-level** action was a freeze.

### Integrator Guidance

To determine whether a given account is currently frozen, integrators SHOULD
call `icrc153_is_frozen_account(account)`, which applies the full
latest-action-wins rule.

To enumerate every effectively frozen account, integrators MUST take the
union of `icrc153_list_frozen_accounts` with the accounts owned by each
principal returned by `icrc153_list_frozen_principals`. Both listing
endpoints are individually sound (each entry is effectively frozen), but
principal-level freezes are not expanded into per-account entries.


## Example: Freeze and Unfreeze — method calls and resulting blocks

> **Encoding note:**  
> In the example method calls below, principals and accounts are shown in
> human-readable form (e.g., `principal "abcd..."` or `[principal "abcd..."]`)
> following Candid notation.  
>  
> In the resulting blocks, the same values appear in their canonical
> **ICRC-3 `Value` representation**, where identifiers are encoded as  
> `variant { Blob = <raw principal bytes> }` or  
> `variant { Array = vec { variant { Blob = <owner bytes> } [, variant { Blob = <subaccount bytes> }] } }`.  
>  
> These two forms represent the **same identity** — the Candid form is used for
> readability in examples, while the `Blob` form is the canonical encoding used
> on-chain for deterministic hashing and certification.



#### Call
The caller invokes:

```
icrc153_freeze_account({
  account         = [principal "ryjl3-tyaaa-aaaaa-aaaba-cai"];
  created_at_time = 1_753_500_100_000_000_000 : nat64;
  reason          = ?"Fraud investigation hold";
})
```

#### Resulting block

This call results in a block with `btype = "123freezeaccount"` and the following contents:

```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123freezeaccount" } };
    record { "phash"; variant { Blob = blob "\12\34\56\78\9a\bc\de\f0\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe\11\22\33\44\55\66\77\88" } };
    record { "ts";    variant { Nat = 1_753_500_101_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";    variant { Text = "153freeze_account" } };
          record { "account"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\00\00\00\02\01\01" }
          } } };
          record { "ts";      variant { Nat = 1_753_500_100_000_000_000 : nat } };
          record { "caller";  variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason";  variant { Text  = "Fraud investigation hold" } };
        }
      };
    };
  }
};
```

The block records the method (`mthd = "153freeze_account"`), the account being frozen (`account`), the caller-supplied timestamp (`ts`), the caller’s principal (`caller`), and the optional `reason`. The account is shown as a principal literal in the call, but stored canonically as a Blob in the block.

#### Call
The caller invokes:
```
icrc153_unfreeze_account({
  account         = [principal "ryjl3-tyaaa-aaaaa-aaaba-cai"];
  created_at_time = 1_753_500_200_000_000_000 : nat64;
  reason          = ?"Lift compliance hold";
})
```

#### Resulting block
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123unfreezeaccount" } };
    record { "phash"; variant { Blob = blob "\9f\aa\10\44\20\19\77\35\c2\9e\00\41\aa\cd\12\ef\04\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_500_201_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";    variant { Text = "153unfreeze_account" } };
          record { "account"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\00\00\00\02\01\01" }
          } } };
          record { "ts";      variant { Nat  = 1_753_500_200_000_000_000 : nat } };
          record { "caller";  variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason";  variant { Text = "Lift compliance hold" } };
        }
      };
    };
  }
};
```


#### Call
The caller invokes:
```
icrc153_freeze_principal({
  principal       = principal "uf6dk-hyaaa-aaaaq-qaaaq-cai";
  created_at_time = 1_753_600_000_000_000_000 : nat64;
  reason          = ?"KYC review";
})
```

#### Resulting block
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123freezeprincipal" } };
    record { "phash"; variant { Blob = blob "\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_600_001_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";      variant { Text = "153freeze_principal" } };
          record { "principal"; variant { Blob = blob "\00\00\00\00\02\10\00\01\01\01" } };
          record { "ts";        variant { Nat  = 1_753_600_000_000_000_000 : nat } };
          record { "caller";    variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason";    variant { Text = "KYC review" } };
        }
      };
    };
  }
};
```

#### Call
The caller invokes:

```
icrc153_unfreeze_principal({
  principal       = principal "uf6dk-hyaaa-aaaaq-qaaaq-cai";
  created_at_time = 1_753_600_500_000_000_000 : nat64;
  reason          = ?"KYC cleared";
})
```


#### Resulting block
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123unfreezeprincipal" } };
    record { "phash"; variant { Blob = blob "\fe\dc\ba\98\76\54\32\10\ef\cd\ab\89\67\45\23\01\99\88\77\66\55\44\33\22\11\00\ff\ee\dd\cc\bb\aa" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_600_501_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";      variant { Text = "153unfreeze_principal" } };
          record { "principal"; variant { Blob = blob "\00\00\00\00\02\10\00\01\01\01" } };
          record { "ts";        variant { Nat  = 1_753_600_500_000_000_000 : nat } };
          record { "caller";    variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason";    variant { Text = "KYC cleared" } };
        }
      };
    };
  }
};
```
