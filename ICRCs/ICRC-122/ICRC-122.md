# ICRC-122: Token Burning & Minting

ICRC-122 introduces new block types for recording token minting and burning events in ICRC-compliant ledgers. These blocks provide a standardized way to document authorized supply modifications, ensuring transparent tracking of token issuance and removal. The `122burn` block records token reductions, while the `122mint` block tracks token increases. By integrating these blocks into the ledger history, implementations enhance auditability and enforceability of token supply constraints.


## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components. Accounts are recorded as an `Array` of two `Value` variants, where the first element is a `variant { Blob = <owner principal> }`, representing the account owner, and the second element is a `variant { Blob = <subaccount> }`, representing the subaccount. If no subaccount is specified, this field MUST be an empty `Blob`. Additionally, each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block. These elements ensure consistency with the ICRC-3 ledger structure and facilitate seamless integration.


## Block Types & Schema

This standard introduces two new block types:

- **Burn Tokens**: `122burn`
- **Mint Tokens**: `122mint`

The specific fields required for the `122burn` and `122mint` block types are as follows:


### 122burn Block
Each `122burn` block MUST include the following fields:

| Field           | Type (ICRC-3 `Value`)  | Description |
|----------------|----------------------|-------------|
| `btype`        | `Text`               | MUST be `122burn` |
| `from`         | `Array(vec { variant { Blob = <owner principal> }, variant { Blob = <subaccount> } })` | The account from which tokens are burned. |
| `amount`       | `Nat`                 | The amount of tokens burned. |
| `authorizer`   | `Blob`                | The principal who authorized the burn. |
| `metadata`     | `Map(Text, Value)`     | Optional metadata for additional details. |

### 122mint Block
Each `122mint` block MUST include the following fields:

| Field           | Type (ICRC-3 `Value`)  | Description |
|----------------|----------------------|-------------|
| `btype`        | `Text`               | MUST be `122mint` |
| `to`           | `Array(vec { variant { Blob = <owner principal> }, variant { Blob = <subaccount> } })` | The account receiving the minted tokens. |
| `amount`       | `Nat`                 | The amount of tokens minted. |
| `authorizer`   | `Blob`                | The principal who authorized the mint. |
| `metadata`     | `Map(Text, Value)`     | Optional metadata for additional details. |

### Interesting Aspects
- Accounts are represented using an **array of two blobs**: the first blob is the owner principal, and the second blob is the subaccount.
- The `amount` field uses the `Nat` type from ICRC-3’s `Value` specification.
- The `authorizer` field records the principal who authorized the operation.

## Expected Semantics
### Burn
- The ledger MUST reduce the supply of tokens accordingly.
- The `from` account balance MUST be reduced by `amount`.
- Transfers MUST NOT be able to use burned tokens.
- The `122burn` block MUST be permanently recorded in the ledger.

### Mint
- The ledger MUST increase the supply of tokens accordingly.
- The `to` account balance MUST be increased by `amount`.
- The `122mint` block MUST be permanently recorded in the ledger.

## Example Blocks
### 122burn Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "122burn" }};
    record { "from"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
        variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
    }} };
    record { "amt"; variant { Nat = 1_000_000 : nat } };
    record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" };};
    record {
                  "phash";
                  variant {
                    Blob = blob "\6f\7e\d9\13\75\69\91\bc\d0\0d\04\b6\60\7b\82\f9\e9\62\a8\39\d3\02\80\f2\88\e4\d7\0e\23\2d\29\87"
                  };    
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

### 122mint Example
```
variant { Map = vec {
    record { "btype"; variant { Text = "122mint" }};
    record { "to"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\02" };
        variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
    }} };
    record { "amount"; variant { Nat = 500_000 : nat } };
    record { "authorizer"; variant { Blob = blob "\00\00\00\00\00\d0\69\56\01\01" };}
    record {
              "phash";
              variant {
                Blob = blob "\ea\3c\e4\5d\3f\2e\1f\89\98\3b\17\55\fe\6a\b8\2d\7d\85\45\69\b1\d4\a3\88\76\4f\79\43\5f\aa\94\4c"
              };
            };
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
}};
```

## Compliance Reporting
Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "122burn" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "122mint" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md" }};
    }};
}
```
