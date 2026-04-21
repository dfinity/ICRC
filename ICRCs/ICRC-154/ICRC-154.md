# `ICRC-154`: Privileged Pause, Unpause & Deactivate API

| Status |
|:------:|
| Draft  |

## Introduction

For operational safety, incident response, and regulatory compliance, ledgers often need a way to **pause** (temporarily disable critical state transitions) or **deactivate** (enter a terminal safe state) the system. Without a standard interface and canonical on-chain recording, integrators cannot reliably determine the current operational state or attribute transitions to authorized actors.

ICRC-154 standardizes three privileged methods and their canonical block mappings (using **ICRC-124** block kinds):

- `icrc154_pause` — temporarily halt state-changing operations.
- `icrc154_unpause` — resume normal operation after a pause.
- `icrc154_deactivate` — permanently deactivate the ledger (terminal state).
- `icrc154_is_paused`, `icrc154_is_deactivated` — read-only status queries.

## Dependencies

This standard does not introduce new block kinds.

- **ICRC-3** — Block log format, hashing, certification, and canonical `tx` mapping rules.
- **ICRC-124** — Defines the typed block kinds that ICRC-154 uses:
  - `btype = "124pause"`
  - `btype = "124unpause"`
  - `btype = "124deactivate"`

A ledger implementing ICRC-154 MUST:
- Emit the appropriate **ICRC-124** block on each successful call.
- Populate `tx.mthd` with namespaced values **introduced by this standard**:
  `"154pause"`, `"154unpause"`, `"154deactivate"`.


## Common Elements

This standard follows the conventions defined in ICRC-3 and used across
ICRC-122, 123, and 124.

- **Principals**  
  Represented as `variant { Blob = <principal_bytes> }`.

- **Timestamps**  
  The `ts` value in the transaction (`tx`) corresponds to the
  caller-supplied `created_at_time` argument.  
  Timestamps are measured in **nanoseconds since the Unix epoch** and
  encoded as `Nat` (MUST fit into `nat64`).

- **Parent Hash**  
  Each block includes `phash : Blob`, the hash of its parent block, unless it
  is the genesis block (where `phash` is omitted).

- **Caller Tracking**  
  Each block includes `tx.caller`, encoded as
  `variant { Blob = <caller_principal_bytes> }`, to identify the authorized
  entity performing the privileged operation.



### `icrc154_pause`

Temporarily move the ledger into a **paused** state (semantics per ICRC-124).

#### Arguments

```
type PauseArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type PauseError = variant {
  Unauthorized       : text;          // caller not permitted
  AlreadyPaused      : text;          // ledger is already paused
  AlreadyDeactivated : text;          // cannot pause when deactivated
  TooOld;
  CreatedInFuture    : record { ledger_time : nat64 };
  Duplicate          : record { duplicate_of : nat };
  GenericError       : record { error_code : nat; message : text };
};

icrc154_pause : (PauseArgs) -> (variant { Ok : nat; Err : PauseError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or
  other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Transition the ledger into the **paused** state, following the semantics of
  ICRC-124.  
- Append a new block with `btype = "124pause"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in
  **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts`
  derived from the `created_at_time` argument.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }`, where `nat` is the block index.  
- On failure: `variant { Err : PauseError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication using `created_at_time`.  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and
  **MUST** return  
  `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `AlreadyPaused` — ledger already paused.  
- `AlreadyDeactivated` — cannot pause when deactivated.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure.

**Clarifications**  
- The `tx` field uses **`ts`** for the caller-supplied timestamp
  (`created_at_time`).  
- Optional fields **MUST** be omitted from `tx` if not supplied.  
- Representation-independent hashing (ICRC-3) applies; field presence and value
  determine the hash, not field order.

#### Canonical `tx` Mapping (normative)

| Field   | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|----------|-----------------------|-------------------------|
| `mthd`   | `Text`                | Constant `"154pause"`. |
| `ts`     | `Nat`                 | From `created_at_time` (ns since Unix epoch; MUST fit in `nat64`). |
| `caller` | `Blob`                | Principal of the caller. |
| `reason` | `Text` (optional)     | From `reason` argument; omit if absent. |



### `icrc154_unpause`

Return the ledger to **unpaused** operation (semantics per ICRC-124).

#### Arguments
```
type UnpauseArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type UnpauseError = variant {
  Unauthorized       : text;
  NotPaused          : text;          // ledger is not currently paused
  AlreadyDeactivated : text;          // cannot unpause when deactivated
  TooOld;
  CreatedInFuture    : record { ledger_time : nat64 };
  Duplicate          : record { duplicate_of : nat };
  GenericError       : record { error_code : nat; message : text };
};

icrc154_unpause : (UnpauseArgs) -> (variant { Ok : nat; Err : UnpauseError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or
  other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Transition the ledger from **paused** to **unpaused** (per ICRC-124).  
- Append a new block with `btype = "124unpause"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in
  **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts`
  derived from the `created_at_time` argument.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }` where `nat` is the block index.  
- On failure: `variant { Err : UnpauseError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication (e.g., using `created_at_time`).  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and
  **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `NotPaused` — ledger is not currently paused.  
- `AlreadyDeactivated` — cannot unpause when deactivated.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**  
- The `tx` field uses **`ts`** for the caller-supplied timestamp (`created_at_time`).  
- Optional fields **MUST** be omitted from `tx` if not supplied.  
- Representation-independent hashing (ICRC-3)

#### Canonical `tx` Mapping (normative)

| Field   | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|---------|------------------------|-------------------------|
| `mthd`  | `Text`                 | **Constant** `"154unpause"`. |
| `ts`    | `Nat`                  | From `created_at_time`. |
| `caller`| `Blob`                 | Principal of the caller. |
| `reason`| `Text` *(optional)*    | From `reason` if provided; **omit** if absent. |


### `icrc154_deactivate`

Move the ledger into a **deactivated** state (long-lived/terminal safe state as per ICRC-124).

#### Arguments
```
type DeactivateArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type DeactivateError = variant {
  Unauthorized       : text;
  AlreadyDeactivated : text;          // ledger already deactivated
  TooOld;
  CreatedInFuture    : record { ledger_time : nat64 };
  Duplicate          : record { duplicate_of : nat };
  GenericError       : record { error_code : nat; message : text };
};

icrc154_deactivate : (DeactivateArgs) -> (variant { Ok : nat; Err : DeactivateError });
```

#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or
  other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- Transition the ledger into the **deactivated** state (per ICRC-124).  
- Append a new block with `btype = "124deactivate"`.  
- The block’s `tx` field **MUST** be constructed **exactly** as defined in
  **Canonical `tx` Mapping** (same keys, types, and encodings), with `ts`
  derived from the `created_at_time` argument.  
- On success, return the index of the newly appended block.

**Return value**  
- On success: `variant { Ok : nat }` where `nat` is the block index.  
- On failure: `variant { Err : DeactivateError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication (e.g., using `created_at_time`).  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and
  **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `AlreadyDeactivated` — ledger is already deactivated.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — identical transaction previously accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid block.

**Clarifications**  
- The `tx` field uses **`ts`** for the caller-supplied timestamp (`created_at_time`).  
- Optional fields **MUST** be omitted from `tx` if not supplied.  
- Representation-independent hashing (ICRC-3) applies; field presence and value determine the hash, not field order.

#### Canonical `tx` Mapping (normative)

| Field   | Type (ICRC-3 `Value`) | Source / Encoding Rule |
|---------|------------------------|-------------------------|
| `mthd`  | `Text`                 | **Constant** `"154deactivate"`. |
| `ts`    | `Nat`                  | From `created_at_time`. |
| `caller`| `Blob`                 | Principal of the caller. |
| `reason`| `Text` *(optional)*    | From `reason` if provided; **omit** if absent. |



## Query & Introspection Methods

These read-only methods expose the current operational state. They **do not** produce blocks.

### `icrc154_is_paused`
```
icrc154_is_paused : () -> (bool) query;
```
- Returns `true` iff the ledger is currently **paused** (per ICRC-124).
- Returns `false` otherwise.

### `icrc154_is_deactivated`
```
icrc154_is_deactivated : () -> (bool) query;
```

- Returns `true` iff the ledger is currently **deactivated** (per ICRC-124).
- Returns `false` otherwise.

**Notes:**
- If the ledger is **deactivated**, it is implementation-defined in ICRC-124 whether certain operations are permanently disabled; `icrc154_unpause` MUST return `AlreadyDeactivated` in that state.
- Answers reflect the ledger state **at query time**.

## Notes on Semantics & Scope

- **Authorizations**: Methods are **privileged**; ledgers MUST restrict calls to authorized principals (policy/RBAC is ledger-defined or standardized separately).
- **Deduplication**: `created_at_time` is used for replay protection/deduplication (as in ICRC-1). Duplicate calls MUST return `Duplicate { duplicate_of = <block_index> }` and MUST NOT produce a new block.
- **Operational semantics**: The precise operational effects of pause/unpause/deactivate (which operations are blocked, permanence of deactivation, etc.) are **entirely defined by ICRC-124**. ICRC-154 only defines the interface and canonical `tx` mapping.
- **No fees**: ICRC-154 calls do not involve fees; ledgers MUST NOT include a top-level `fee` field for these blocks.

## Reporting Compliance

### Supported Standards

Ledgers implementing ICRC-154 MUST indicate compliance via `icrc10_supported_standards`, and via `icrc1_supported_standards` if ICRC-1 is also implemented, by including:

```candid
record { name = "ICRC-154"; url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-154/ICRC-154.md" }
```
### Supported Block Types

ICRC-154 extends **ICRC-124** and does not introduce new block kinds.
Accordingly, ledgers implementing ICRC-154 MUST already advertise support for the
relevant ICRC-124 block kinds (`124pause`, `124unpause`, `124deactivate`) as required by ICRC-124.
No additional block types need to be reported.



### Example calls and resulting blocks

#### Call
```
icrc154_pause({
  created_at_time = 1_753_700_000_000_000_000 : nat64;
  reason          = ?"System maintenance window";
})
```


#### Resulting Block```variant {
  Map = vec {
    record { "btype"; variant { Text = "124pause" } };
    record { "phash"; variant { Blob = blob "\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_700_001_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";   variant { Text = "154pause" } };
          record { "ts";     variant { Nat  = 1_753_700_000_000_000_000 : nat } };
          record { "caller"; variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text = "System maintenance window" } };
        }
      };
    };
  }
};
```



### Example: `icrc154_unpause`

**Call with parameters**
```motoko
icrc154_unpause({
  created_at_time = 1_753_700_500_000_000_000 : nat64;
  reason          = ?"Maintenance completed";
})

```

#### Resulting Block
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "124unpause" } };
    record { "phash"; variant { Blob = blob "\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_700_501_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";   variant { Text = "154unpause" } };
          record { "ts";     variant { Nat  = 1_753_700_500_000_000_000 : nat } };
          record { "caller"; variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text = "Maintenance completed" } };
        }
      };
    };
  }
};
```


### Example: `icrc154_deactivate`


```
icrc154_deactivate({
  created_at_time = 1_753_800_000_000_000_000 : nat64;
  reason          = ?"Regulatory directive";
})
```

#### Resulting Block
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "124deactivate" } };
    record { "phash"; variant { Blob = blob "\2d\86\7f\34\c7\2d\1e\2d\00\84\10\a4\00\b0\b6\4c\3e\02\96\c9\e8\55\6f\dd\72\68\e8\df\8d\8e\8a\ee" } }; // illustrative
    record { "ts";    variant { Nat = 1_753_800_001_000_000_000 : nat } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";   variant { Text = "154deactivate" } };
          record { "ts";     variant { Nat  = 1_753_800_000_000_000_000 : nat } };
          record { "caller"; variant { Blob = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text = "Regulatory directive" } };
        }
      };
    };
  }
};
```

