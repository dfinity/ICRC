# ICRC-124: Ledger Pausing, Unpausing, and Deactivation

## Status

Draft

## Introduction

This standard defines new block types for recording administrative actions that control the operational state of an ICRC-compliant ledger: pausing (`124pause`), unpausing (`124unpause`), and deactivating (`124deactivate`). These actions allow ledger operations to be temporarily halted (e.g., for maintenance), resumed, or permanently stopped (making the ledger immutable). This standard provides a consistent, auditable way to represent these ledger-wide state transitions within the ledger's block history, ensuring transparency and enabling robust governance mechanisms. The transaction details (`tx`) within each block explicitly include the `caller` principal that authorized the operation.

## Motivation

Ledger lifecycle management may require administrative actions like pausing for upgrades, unpausing after checks, or deactivating at the end of a token's useful life. These significant events must be recorded transparently on-chain. This standard provides explicit block types for these actions, defining a block structure that includes the initiator (`caller`) and essential details for the operation, enhancing on-chain auditability.

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Principals** (such as the `caller`) are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes information about the administrative action, including the `caller` principal and an optional reason.

**Important Note on Transaction Recoverability:** The `tx` field defined below now includes the `caller` principal. However, for full auditability and transparency in complex scenarios, ledger implementations compliant with ICRC-124 **MUST** ensure that any other details of the original transaction invocation not captured in `tx` can be recovered independently. This could include, but is not limited to, the full arguments passed to the ledger method (if more complex than the data in `tx`), or any intermediary calls if the operation was part of a multi-step process. Mechanisms for recovering such extended data (e.g., via archive queries or specific lookup methods) remain implementation-dependent. The rules determining *who* is authorized to invoke these ledger state change operations are an implementation detail of the ledger's governance model.

Each block defined by this standard consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"124pause"`, `"124unpause"`, or `"124deactivate"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes information about the pause/unpause/deactivate operation, including the caller. See schema below. |

### `tx` Field Schema

The `tx` field schema is the same for `124pause`, `124unpause`, and `124deactivate`:

| Field        | Type (ICRC-3 `Value`)                                    | Required | Description |
|--------------|----------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method causing this block. |
| `reason`     | `Text`                                                   | Optional | Human-readable reason for the administrative action. |

## Semantics

The recording of these blocks MUST influence the behavior of the ledger according to the following semantics:

### Pause Ledger (`124pause`)
- When a `124pause` block is recorded, the ledger MUST enter a "paused" state.
- While paused, the ledger MUST reject incoming requests for standard token operations such as `icrc1_transfer`, `icrc2_approve`, and other non-administrative state changes such as those defined in ICRC-122 (e.g., `122mint`, `122burn`).
- However, while paused, the ledger MUST continue to accept specific administrative or management operations necessary for governance or recovery. This includes operations defined in ICRC-123 (e.g., `123freezeaccount`, `123unfreezeaccount`, `123freezeprincipal`, `123unfreezeprincipal`) and, critically, requests that result in recording a `124unpause` block. Ledger implementations MAY also permit requests that result in recording a `124deactivate` block while the ledger is paused, according to their defined governance policies. The exact set of allowed administrative operations during a pause SHOULD be defined by the specific ledger implementation's policy.
- Query calls SHOULD generally remain operational while the ledger is paused.

### Unpause Ledger (`124unpause`)
- When a `124unpause` block is recorded, the ledger MUST exit the "paused" state and resume normal operation, accepting transactions as defined by its implementation and other active states (unless it is in a terminal state).
- An `124unpause` block has no effect if the ledger is already unpaused or if it is in a terminal state due to deactivation.

### Deactivate Ledger (`124deactivate`)
- When a `124deactivate` block is recorded, the ledger MUST transition to a permanent "terminal" or "deactivated" state.
- In this terminal state:
    - All ingress calls attempting to modify the ledger state MUST be permanently rejected. This includes, but is not limited to, transfers, approvals, mints, burns, freezes, unfreezes, pauses, unpauses, and any other state-changing operations defined now or in the future. **No transactions that alter state are permitted.**
    - Query calls retrieving historical data (e.g., transaction history, past balances via `icrc3_get_blocks`) MUST remain available indefinitely to preserve the immutable record.
- The deactivated state is irreversible.

## Compliance Reporting

Ledgers implementing this standard MUST return the following entries (including entries for other supported types like ICRC-1, ICRC-3, etc.) from `icrc3_supported_block_types`, with URLs pointing to the standards defining each block type:

```candid
vec {
    // ... other supported types like ICRC-1, ICRC-3, ICRC-122, ICRC-123 ...
    variant { Record = vec {
        record { "btype"; variant { Text = "124pause" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "124unpause" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "124deactivate" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md)" }}; // Placeholder URL
    }};
}

```

## Example Blocks

### 124pause Example

```candid
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "124pause" }};

    // Timestamp when the block was recorded (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_747_774_560_000_000_000 : nat }}; // Example: 2025-05-19T12:56:00Z

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\de\ad\be\ef\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\10\20\30\40\50\60\70\80\90\a0\b0\c0"
    }};

    // Pause transaction details
    record { "tx"; variant { Map = vec {
        // The principal that invoked the pause_ledger operation
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\03" }}; // Example caller principal (e.g., a governance canister)
        // Optional reason
        record { "reason"; variant { Text = "DAO vote #78: pause for scheduled maintenance." }};
    }}};
}};

```

### 124unpause Example

```candid
variant { Map = vec {
    record { "btype"; variant { Text = "124unpause" }};
    record { "ts"; variant { Nat = 1_747_778_160_000_000_000 : nat }}; // Example: 2025-05-19T13:56:00Z
    record { "phash"; variant {
        Blob = blob "\be\ba\fe\ca\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\0f\10\11\12\13\14\15\16\17\18\19\1a\1b"
    }};
    // Unpause transaction details
    record { "tx"; variant { Map = vec {
        // The principal that invoked the unpause_ledger operation
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\03" }}; // Example caller principal
        // Optional reason
        record { "reason"; variant { Text = "Ledger resumes after maintenance window (DAO vote #79)." }};
    }}};
}};
```

### 124deactivate Example

```candid
variant { Map = vec {
    record { "btype"; variant { Text = "124deactivate" }};
    record { "ts"; variant { Nat = 1_747_864_560_000_000_000 : nat }}; // Example: 2025-05-20T12:56:00Z
    record { "phash"; variant {
        Blob = blob "\c0\ff\ee\00\10\20\30\40\50\60\70\80\90\a0\b0\c0\d0\e0\f0\00\11\22\33\44\55\66\77\88\99\aa\bb\cc"
    }};
    // Deactivate transaction details
    record { "tx"; variant { Map = vec {
        // The principal that invoked the deactivate_ledger operation
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\04" }}; // Example caller (e.g., project multisig or final DAO vote)
        // Optional reason
        record { "reason"; variant { Text = "Token project sunset. Ledger permanently archived as per SNS DAO proposal #314." }};
    }}};
}};

```
