# ICRC-123: Account and Principal Freezing & Unfreezing

ICRC-123 introduces new block types for recording account and unfreezing events in ICRC-compliant ledgers. These blocks provide a standardized way to document administrative actions restricting or restoring account activity. The `123freezeaccount` block records an account freeze event, while the `123unfreezeaccount` block documents the removal of such restrictions. Additionally, the `123freezeprincipal` block records the freezing of all accounts belonging to a principal, and the `123unfreezeprincipal` block records the unfreezing of all accounts belonging to a principal.

## Common Elements

This standard follows the conventions set by ICRC-3, inheriting key structural components. Accounts are recorded as an `Array` of two `Value` variants, where:

- The first element is a `variant { Blob = <owner principal> }`, representing the account owner.
- The second element is a `variant { Blob = <subaccount> }`, representing the subaccount. If no subaccount is specified, this field MUST be an empty `Blob`.

Additionally, each block includes:

- `phash`: a `Blob` representing the hash of the parent block.
- `ts`: a `Nat` representing the timestamp of the block.

## Block Types & Schema

This standard introduces four new block types:

- **Freeze Account**: `123freezeaccount`
- **Unfreeze Account**: `123unfreezeaccount`
- **Freeze Principal**: `123freezeprincipal`
- **Unfreeze Principal**: `123unfreezeprincipal`

## Block Types & Schema

This standard introduces four new block types:

- **Freeze Account**: `123freezeaccount`
- **Unfreeze Account**: `123unfreezeaccount`
- **Freeze Principal**: `123freezeprincipal`
- **Unfreeze Principal**: `123unfreezeprincipal`

### 123freezeaccount Block
Each `123freezeaccount` block MUST include the following fields:

| Field        | Type (ICRC-3 `Value`) | Description                                      |
| ------------ | --------------------- | ------------------------------------------------ |
| `btype`      | `Text`                | MUST be `123freezeaccount`                      |
| `account`    | `Array(vec { Blob, Blob })` | The account being frozen.                        |
| `authorizer` | `Blob`                | The principal who authorized the freeze.       |
| `metadata`   | `Map(Text, Value)`    | Optional metadata for additional details.      |

### 123unfreezeaccount Block
Each `123unfreezeaccount` block MUST include the following fields:

| Field        | Type (ICRC-3 `Value`) | Description                                      |
| ------------ | --------------------- | ------------------------------------------------ |
| `btype`      | `Text`                | MUST be `123unfreezeaccount`                    |
| `account`    | `Array(vec { Blob, Blob })` | The account being unfrozen.                      |
| `authorizer` | `Blob`                | The principal who authorized the unfreeze.     |
| `metadata`   | `Map(Text, Value)`    | Optional metadata for additional details.      |

### 123freezeprincipal Block
Each `123freezeprincipal` block MUST include the following fields:

| Field        | Type (ICRC-3 `Value`) | Description                                      |
| ------------ | --------------------- | ------------------------------------------------ |
| `btype`      | `Text`                | MUST be `123freezeprincipal`                    |
| `principal`  | `Blob`                | The principal whose accounts are being frozen.  |
| `authorizer` | `Blob`                | The principal who authorized the freeze.       |
| `metadata`   | `Map(Text, Value)`    | Optional metadata for additional details.      |

### 123unfreezeprincipal Block
Each `123unfreezeprincipal` block MUST include the following fields:

| Field        | Type (ICRC-3 `Value`) | Description                                      |
| ------------ | --------------------- | ------------------------------------------------ |
| `btype`      | `Text`                | MUST be `123unfreezeprincipal`                  |
| `principal`  | `Blob`                | The principal whose accounts are being unfrozen. |
| `authorizer` | `Blob`                | The principal who authorized the unfreeze.       |
| `metadata`   | `Map(Text, Value)`    | Optional metadata for additional details.        |



## Semantics

- When a `123freezeaccount` block is recorded, the ledger MUST ensure that the specified `account` is prevented from initiating or receiving transfers.
- When a `123freezeprincipal` block is recorded, **all accounts (including subaccounts) belonging to the specified principal MUST be frozen**.
- Any attempt to transact with a frozen account MUST return an error.
- When a `123unfreezeaccount` block is recorded, the ledger MUST restore the ability of the specified `account` to initiate and receive transactions.
- When a `123unfreezeprincipal` block is recorded, **all accounts (including subaccounts) belonging to the specified principal MUST be unfrozen**.
- All freeze and unfreeze blocks MUST be permanently recorded in the ledger.
- Querying an account's status SHOULD indicate whether it is currently frozen.

## Example Blocks

### 123freezeaccount Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123freezeaccount" }};
    record { "account"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

### 123unfreezeaccount Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123unfreezeaccount" }};
    record { "account"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

### 123freezeprincipal Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123freezeprincipal" }};
    record { "principal"; variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }};
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

### 123unfreezeprincipal Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123unfreezeprincipal" }};
    record { "principal"; variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }};
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

## Compliance Reporting

Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
}
```
