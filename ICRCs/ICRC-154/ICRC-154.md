# `ICRC-154`: Privileged Pause, Unpause & Deactivate API

| Status |
|:------:|
| Draft  |

## Introduction & Motivation

For operational safety, incident response, and regulatory compliance, ledgers often need a way to **pause** (temporarily disable critical state transitions) or **deactivate** (enter a long-lived/terminal safe state) the system. Without a standard interface and canonical on-chain recording, integrators cannot reliably determine the current operational state or attribute transitions to authorized actors.

**ICRC-154** standardizes three privileged methods—`pause`, `unpause`, and `deactivate`—and defines the canonical mapping from method inputs to the `tx` field of the corresponding typed blocks (as defined in **ICRC-124**). It also includes lightweight query methods for current status.

- Privileged methods (authorized principals only):
  - `icrc154_pause`, `icrc154_unpause`, `icrc154_deactivate`
- Canonical `tx` mapping with namespaced `op` values (`"154..."`), caller identity, and optional human-readable `reason`.
- Recording uses **ICRC-124** block kinds (this standard **does not** add new block types).
- Read-only queries: `icrc154_is_paused`, `icrc154_is_deactivated`.

## Overview

ICRC-154 standardizes privileged **pause/unpause/deactivate** controls for ICRC ledgers.

Specifically, it defines:

- **APIs** to pause, unpause, and deactivate the ledger (privileged only).
- **Canonical `tx` mapping** rules ensuring deterministic, auditable block content.
- **Use of ICRC-124 block kinds** to record these actions:
  - `btype = "124pause"`, `btype = "124unpause"`, `btype = "124deactivate"`
- **Status queries** for quick integration.

This enables wallets, explorers, and auditors to:

- Determine, on-chain, whether the ledger is currently **paused** or **deactivated**.
- Attribute actions to a specific authorized caller with an optional `reason`.
- Interoperate across ledgers that implement the same API and block semantics.

## Dependencies

This standard does not introduce new block kinds.

- **ICRC-3** — Block log format, hashing, certification, and canonical `tx` mapping rules.
- **ICRC-124** — Defines the typed block kinds that ICRC-154 uses:
  - `btype = "124pause"`
  - `btype = "124unpause"`
  - `btype = "124deactivate"`

A ledger implementing ICRC-154 MUST:
- Emit the appropriate **ICRC-124** block on each successful call.
- Populate `tx.op` with namespaced values **introduced by this standard**:
  `"154pause"`, `"154unpause"`, `"154deactivate"`.

### `icrc154_pause`

Temporarily move the ledger into a **paused** state (semantics per ICRC-124).

#### Arguments

```
type PauseArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type PauseError = variant {
  Unauthorized : text;                // caller not permitted
  AlreadyPaused : text;               // ledger is already paused
  AlreadyDeactivated : text;          // cannot pause when deactivated
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc154_pause : (PauseArgs) -> (variant { Ok : nat; Err : PauseError });
```

#### Semantics
- Transitions the ledger into **paused** state (per ICRC-124’s operational rules).
- Appends a block of type `124pause`.
- On success, returns the **index of the created block**.
- On failure, returns an appropriate error.
- Semantics are consistent with the pause semantics defined by ICRC-124.

#### Return Values
- Success: `variant { Ok : nat }` — created block index.
- Failure: `variant { Err : PauseError }`.

#### Canonical `tx` Mapping
A successful call to `icrc154_pause` produces a `124pause` block. The `tx` field:

- `op     = "154pause"`
- `ts     = PauseArgs.created_at_time`
- `caller = caller_principal (as Blob)`
- `reason = PauseArgs.reason` (if provided)

Optional fields MUST be omitted if not supplied.

### `icrc154_unpause`

Return the ledger to **unpaused** operation (semantics per ICRC-124).

#### Arguments
```
type UnpauseArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type UnpauseError = variant {
  Unauthorized : text;
  NotPaused : text;                   // ledger is not currently paused
  AlreadyDeactivated : text;          // cannot unpause when deactivated
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc154_unpause : (UnpauseArgs) -> (variant { Ok : nat; Err : UnpauseError });
```

#### Semantics
- Transitions the ledger from **paused** to **unpaused** (per ICRC-124).
- Appends a block of type `124unpause`.
- On success, returns the **index of the created block**.
- On failure, returns an appropriate error.
- Semantics are consistent with the unpause semantics defined by ICRC-124.

#### Return Values
- Success: `variant { Ok : nat }` — created block index.
- Failure: `variant { Err : UnpauseError }`.

#### Canonical `tx` Mapping
A successful call to `icrc154_unpause` produces a `124unpause` block. The `tx` field:

- `op     = "154unpause"`
- `ts     = UnpauseArgs.created_at_time`
- `caller = caller_principal (as Blob)`
- `reason = UnpauseArgs.reason` (if provided)

Optional fields MUST be omitted if not supplied.

### `icrc154_deactivate`

Move the ledger into a **deactivated** state (long-lived/terminal safe state as per ICRC-124).

#### Arguments
```
type DeactivateArgs = record {
  reason          : opt text;
  created_at_time : nat64;
};

type DeactivateError = variant {
  Unauthorized : text;
  AlreadyDeactivated : text;          // ledger already deactivated
  Duplicate : record { duplicate_of : nat };
  GenericError : record { error_code : nat; message : text };
};

icrc154_deactivate : (DeactivateArgs) -> (variant { Ok : nat; Err : DeactivateError });
```

#### Semantics
- Transitions the ledger into **deactivated** state (per ICRC-124).
- Appends a block of type `124deactivate`.
- On success, returns the **index of the created block**.
- On failure, returns an appropriate error.
- Semantics are consistent with the deactivate semantics defined by ICRC-124.

#### Return Values
- Success: `variant { Ok : nat }` — created block index.
- Failure: `variant { Err : DeactivateError }`.

#### Canonical `tx` Mapping
A successful call to `icrc154_deactivate` produces a `124deactivate` block. The `tx` field:

- `op     = "154deactivate"`
- `ts     = DeactivateArgs.created_at_time`
- `caller = caller_principal (as Blob)`
- `reason = DeactivateArgs.reason` (if provided)

Optional fields MUST be omitted if not supplied.

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

Ledgers implementing ICRC-154 MUST indicate compliance through the
`icrc1_supported_standards` and `icrc10_supported_standards` methods by including:
```
variant { Vec = vec {
  record {
    "name"; variant { Text = "ICRC-154" };
    "url";  variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-154.md" };
  }
}};
```
### Supported Block Types

ICRC-154 extends **ICRC-124** and does not introduce new block kinds.
Accordingly, ledgers implementing ICRC-154 MUST already advertise support for the
relevant ICRC-124 block kinds (`124pause`, `124unpause`, `124deactivate`) as required by ICRC-124.
No additional block types need to be reported.
