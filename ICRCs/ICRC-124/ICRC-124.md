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


## Example blocks

### 124pause Example

```
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "124pause" }};

    // Timestamp when the block was appended (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat }};

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\de\ad\be\ef\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\10\20\30\40\50\60\70\80\90\a0\b0\c0"
    }};

    // Pause transaction payload
    record { "tx"; variant { Map = vec {
        // Principal that authorized the pause
        record { "authorizer"; variant { Blob = blob "\ab\cd\01\23\45\67\89\ef\01\23\45\67\89\ab\cd\ef\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\01" }};

        // Optional metadata
        record { "metadata"; variant { Map = vec {
            record { "reason"; variant { Text = "DAO vote: pause due to upgrade prep" }}
        }}}
    }}};
}};
```

### 124unpause Example

```
variant { Map = vec {
    record { "btype"; variant { Text = "124unpause" }};
    record { "ts"; variant { Nat = 1_741_312_737_200_000_000 : nat }};
    record { "phash"; variant {
        Blob = blob "\be\ba\fe\ca\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\0f\10\11\12\13\14\15\16\17\18\19\1a\1b"
    }};
    record { "tx"; variant { Map = vec {
        record { "authorizer"; variant { Blob = blob "\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\00\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\01" }};
        record { "metadata"; variant { Map = vec {
            record { "note"; variant { Text = "Ledger resumes after maintenance window" }}
        }}}
    }}};
}};
```

### 124deactivate Example

```
variant { Map = vec {
    record { "btype"; variant { Text = "124deactivate" }};
    record { "ts"; variant { Nat = 1_741_312_737_250_000_000 : nat }};
    record { "phash"; variant {
        Blob = blob "\c0\ff\ee\00\10\20\30\40\50\60\70\80\90\a0\b0\c0\d0\e0\f0\00\11\22\33\44\55\66\77\88\99\aa\bb\cc"
    }};
    record { "tx"; variant { Map = vec {
        record { "authorizer"; variant { Blob = blob "\de\ad\be\ef\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\10\20\30\40\50\60\70\80\90\a0\b0\01" }};
        record { "metadata"; variant { Map = vec {
            record { "finalized_by"; variant { Text = "SNS DAO proposal 314" }},
            record { "note"; variant { Text = "Ledger permanently frozen to preserve historical state" }}
        }}}
    }}};
}};

```
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
