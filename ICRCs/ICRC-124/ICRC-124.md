## Motivation

ICRC-124 provides essential administrative controls for token ledgers on the Internet Computer to:

1. **Emergency Response**: Enable temporary suspension during security incidents or suspected fraud
2. **Regulatory Compliance**: Support mechanisms for meeting legal requirements
3. **Lifecycle Management**: Allow for transparent and standardized token retirement
4. **Auditability**: Create clear records of administrative actions and their authorization
5. **Operational Stability**: Provide standardized recovery paths from exceptional states


## Overview of Block Types

- **Pause Ledger**: `124pause`
- **Unpause Ledger**: `124unpause`
- **Deactivate Ledger**: `124deactivate`

## Common Structure

Each block introduced by this standard MUST include a `tx` field containing a map that encodes the specific administrative action that was submitted to the ledger and which resulted in the block being created. These blocks follow the ICRC-3 structure and include:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | One of: `"124pause"`, `"124unpause"`, or `"124deactivate"` |
| `ts`     | `Nat`                  | Yes      | Timestamp (in nanoseconds) of block creation |
| `phash`  | `Blob`                 | Yes      | Hash of the previous block |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes the pause, unpause, or deactivate transaction |

### `tx` Field Schema

All three block types share the same schema:

| Field        | Type (ICRC-3 `Value`) | Required | Description |
|--------------|------------------------|----------|-------------|
| `authorizer` | `Blob`                 | Yes      | Principal that authorized the action |
| `metadata`   | `Map(Text, Value)`     | Optional | Additional context (e.g., reason, governance proposal) |

## Semantics
- When a `124pause` block is recorded, the ledger MUST reject all new transactions **except** a `124unpause` transaction.
- When a `124unpause` block is recorded, the ledger MUST resume accepting transactions (unless in terminal state).
- When a `124deactivate` block is recorded, the ledger MUST transition to a **terminal state** where:
  - All new state-modifying transactions are permanently rejected
  - Historical transaction data remains available for querying
  - The token's transaction history is preserved as an immutable record

Once a `124deactivate` block is recorded, the ledger's state becomes **immutable** for transfer operations while maintaining read access to historical data.


## Compliance Reporting

Ledgers implementing this standard MUST return the following entries from `icrc3_supported_block_types`:

```motoko
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "124pause" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "124unpause" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "124deactivate" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124.md" }};
    }};
}
