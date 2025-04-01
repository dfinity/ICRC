# ICRC-122: Token Burning & Minting

ICRC-122 introduces new block types for recording token minting and burning events in ICRC-compliant ledgers. These blocks provide a standardized way to document authorized supply modifications, ensuring transparent tracking of token issuance and removal. The `122burn` block records token reductions, while the `122mint` block tracks token increases.


## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components. Accounts are recorded as an `Array` of two `Value` variants, where the first element is a `variant { Blob = <owner principal> }`, representing the account owner, and the second element is a `variant { Blob = <subaccount> }`, representing the subaccount. If no subaccount is specified, this field MUST be an empty `Blob`. Additionally, each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.


## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map that encodes the token mint or burn transaction submitted to the ledger that caused this block to be created, similarly to how transaction blocks defined in ICRC-1 and ICRC-2 include the submitted transaction.

This enables canister clients, indexers, and auditors to reconstruct the exact instruction that led to the block being appended to the ledger.

Each `122burn` or `122mint` block consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"122burn"` or `"122mint"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes the token mint or burn transaction submitted to the ledger that caused this block to be created. |

### `tx` Field Schemas

#### For `122burn`

| Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `from`       | `Array(vec { Blob [, Blob] })` | Yes      | The account from which tokens are burned. |
| `amount`     | `Nat`                         | Yes      | The number of tokens to burn. |
| `authorizer` | `Blob`                        | Yes      | The principal who authorized the burn. |
| `metadata`   | `Map(Text, Value)`            | Optional | Additional metadata. |

#### For `122mint`

| Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `to`         | `Array(vec { Blob [, Blob] })` | Yes      | The account to which tokens are minted. |
| `amount`     | `Nat`                         | Yes      | The number of tokens to mint. |
| `authorizer` | `Blob`                        | Yes      | The principal who authorized the mint. |
| `metadata`   | `Map(Text, Value)`            | Optional | Additional metadata. |


## Interesting Aspects

- Accounts are represented using an **array of one or two blobs**: the first blob is the owner principal, and the second blob is the subaccount (if present). If no subaccount is specified, only the owner is included.
- The `amount` field uses the `Nat` type from ICRC-3’s `Value` specification.
- The `authorizer` field in the `tx` object identifies the principal who authorized the operation.
- Each `122burn` or `122mint` block contains a `tx` field that encodes the mint or burn instruction that caused the block to be appended to the ledger.

## Semantics

### Burn

When a block of type `122burn` is recorded:

- The ledger MUST reduce the total supply of tokens by the amount specified in `tx.amount`.
- The balance of the `tx.from` account MUST be decreased by `tx.amount`.
- Tokens burned in this way MUST NOT be available for future transfers.
- The `122burn` block MUST be permanently recorded in the ledger.


### Mint

When a block of type `122mint` is recorded:

- The ledger MUST increase the total supply of tokens by the amount specified in `tx.amount`.
- The balance of the `tx.to` account MUST be increased by `tx.amount`.
- The `122mint` block MUST be permanently recorded in the ledger.

## Example Blocks
### 122burn Example
```
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "122burn" }};

    // Timestamp when the block was appended (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat }};

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\6f\7e\d9\13\75\69\91\bc\d0\0d\04\b6\60\7b\82\f9\e9\62\a8\39\d3\02\80\f2\88\e4\d7\0e\23\2d\29\87"
    }};

    // The burn transaction
    record { "tx"; variant { Map = vec {
        // The account from which tokens are burned
        record { "from"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
        }}};

        // The amount to burn
        record { "amount"; variant { Nat = 1_000_000 : nat }};

        // The principal that authorized the burn
        record { "authorizer"; variant { Blob = blob "\94\85\a4\06\ba\33\de\19\f8\ad\b1\ee\3d\07\9e\63\1d\7f\59\43\57\bc\dd\98\56\63\83\96\02" }};
    }}};
}};

```

### 122mint Example
```
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "122mint" }};

    // Timestamp when the block was appended (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat }};

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\7a\61\cf\18\a4\9d\ac\20\39\33\78\f1\c2\fd\45\81\ab\55\37\20\1d\fe\77\a3\c6\55\de\01\92\a4\3b\ee"
    }};

    // The mint transaction
    record { "tx"; variant { Map = vec {
        // The account to which tokens are minted (principal + subaccount)
        record { "to"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };  // owner principal
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }  // subaccount
        }}};

        // The amount to mint
        record { "amount"; variant { Nat = 2_000_000 : nat }};

        // The principal that authorized the mint
        record { "authorizer"; variant { Blob = blob "\aa\bc\34\12\cd\78\ef\90\01\23\45\67\89\ab\cd\ef\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\0f" }};
    }}};
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
