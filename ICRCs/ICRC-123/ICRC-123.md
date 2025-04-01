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

This section defines the semantics of the freeze and unfreeze block types introduced by this standard.

### Account Status

Given the state of the ledger at a particular block height `h`, an account `acc = (owner: Principal, subaccount: Blob)` is considered **RESTRICTED** if and only if the most recent freeze or unfreeze block at or before height `h` that affects `acc` is a freeze.

A block is considered to affect an account if it satisfies one of the following:

- It is a `123freezeaccount` or `123unfreezeaccount` block whose `account` field matches `acc`.
- It is a `123freezeprincipal` or `123unfreezeprincipal` block whose `principal` field matches the `owner` of `acc`.

To determine whether an account is RESTRICTED, ledgers MUST identify the most recent block at or before height `h` that affects the account, and check whether it is of type `123freezeaccount` or `123freezeprincipal`.

This means:

- A freeze of an account (`123freezeaccount`) can be lifted by a later unfreeze of the same account (`123unfreezeaccount`) **or** by a later unfreeze of the owning principal (`123unfreezeprincipal`).
- A freeze of a principal (`123freezeprincipal`) can be lifted by a later unfreeze of that principal (`123unfreezeprincipal`).
- An unfreeze block always overrides any earlier freeze affecting the same account or principal, regardless of whether the freeze was explicit (account-level) or implicit (principal-level).

Otherwise, the account is considered **NON-RESTRICTED**.


### Ledger Enforcement Rules

- A ledger **MUST reject** any transfer transaction (`icrc1_transfer` or `icrc2_trasfer_from`) where the **sender or recipient account is currently RESTRICTED**.
- Freeze and unfreeze blocks do **not** modify or invalidate previous transactions. They apply only to transactions **at or after** the block height at which the freeze/unfreeze block is recorded.
- Freeze and unfreeze blocks MUST be **permanently recorded** and included in the block hash chain.

### Authorization

- Each freeze and unfreeze block includes an `authorizer` field, which records the principal who authorized the action.
- This standard does **not prescribe** how the ledger verifies that the `authorizer` is permitted to freeze or unfreeze. Ledger implementations MAY use governance mechanisms, access control lists, or DAO-based authorization.

### Idempotency and Redundancy

- A ledger MAY reject freeze or unfreeze blocks that would have **no effect** (e.g., freezing an already frozen account), or MAY choose to **record them anyway** for auditability.
- Clients interpreting freeze status MUST follow a **"latest-action-wins" rule**: the most recent freeze or unfreeze block affecting an account or principal determines its effective status.

### Querying Freeze Status

Ledgers implementing this standard SHOULD expose a query interface (e.g., `is_account_frozen(account)`) that returns whether an account is currently restricted. This serves as a convenience layer and does not replace auditing based on block history.




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

## Example Blocks

### 123freezeaccount Example
```
variant { Map = vec {
    // The block type
    record { "btype"; variant { Text = "123freezeaccount" }};
    // The account that is frozen
    record { "account";
        variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    // The principal that has authorized freezing the account
    record { "authorizer";
        variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    // The time (in nanoseconds) when the block was appended to the ledger
    record { "ts";
        variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: hash of the previous block; links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```

### 123unfreezeaccount Example
```
variant { Map = vec {
    // The block type
    record { "btype"; variant { Text = "123unfreezeaccount" }};
    // The account that is unfrozen
    record { "account"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
    }} };
    // The principal that has authorized unfreezing the account
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```

### 123freezeprincipal Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123freezeprincipal" }};
    // The principal whose accounts are all frozen
    record { "principal"; variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }};
    // The principal that has authorized freezing the principal
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    // The time (in nanoseconds) when the block was appended to the ledger
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```

### 123unfreezeprincipal Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "123unfreezeprincipal" }};
    // The principal whose accounts are all unfrozen
    record { "principal"; variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }};
    // The principal that has authorized unfreezing the principal
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    // The time (in nanoseconds) when the block was appended to the ledger
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // The time (in nanoseconds) when the block was appended to the ledger
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```
