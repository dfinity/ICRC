# ICRC-123: Account Freezing & Unfreezing

## Account Representation
ICRC-1 Accounts are represented as an `Array` containing two `Blob` values:
- The first `Blob` is the `owner` principal.
- The second `Blob` is the `subaccount`, which is optional. If no subaccount is specified, this field MUST be an empty `Blob`.

## Block Types
- **Freeze Account**: `123freeze`
- **Unfreeze Account**: `123unfreeze`

## Block Schema
### 123freeze Block
Each `123freeze` block MUST include the following fields:

| Field           | Type (ICRC-3 `Value`)  | Description |
|----------------|----------------------|-------------|
| `btype`        | `Text`               | MUST be `123freeze` |
| `account`      | `Array(vec { Blob, Blob })` | The account being frozen. The first `Blob` is the owner principal, and the second `Blob` is the subaccount. |
| `authorizer`   | `Blob`                | The principal who authorized the freeze. |
| `metadata`     | `Map(Text, Blob)`     | Optional metadata for additional details. |

### 123unfreeze Block
Each `123unfreeze` block MUST include the following fields:

| Field           | Type (ICRC-3 `Value`)  | Description |
|----------------|----------------------|-------------|
| `btype`        | `Text`               | MUST be `123unfreeze` |
| `account`      | `Array(vec { Blob, Blob })` | The account being unfrozen. |
| `authorizer`   | `Blob`                | The principal who authorized the unfreeze. |
| `metadata`     | `Map(Text, Blob)`     | Optional metadata for additional details. |

### Interesting Aspects
- Accounts are represented using an **array of two blobs**: the first blob is the owner principal, and the second blob is the subaccount.
- The `authorizer` field records the principal who authorized the operation.

## Expected Semantics
### Freeze
- When `123freeze` is recorded, the specified `account` MUST be prevented from initiating or receiving transfers.
- Transactions involving a frozen account MUST return an error.
- The `123freeze` block MUST be permanently recorded in the ledger.

### Unfreeze
- When `123unfreeze` is recorded, the specified `account` MUST regain full transfer functionality.
- The `123unfreeze` block MUST be permanently recorded in the ledger.

## Example Blocks
### 123freeze Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123freeze" }};
    record { "account"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    record { "authorizer"; variant { Blob = blob "\b1\a2\c3\d4\e5\f6" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

### 123unfreeze Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123unfreeze" }};
    record { "account"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    record { "authorizer"; variant { Blob = blob "\b1\a2\c3\d4\e5\f6" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

## Compliance Reporting
Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "123freeze" }};
        record { "url"; variant { Text = "https://example.com/icrc-123" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreeze" }};
        record { "url"; variant { Text = "https://example.com/icrc-123" }};
    }};
}
```
