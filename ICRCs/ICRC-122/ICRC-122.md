
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
| `btype`  | `Text`                 | Yes      | MUST be one of: "122burn" or "122mint". |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes the token mint or burn transaction submitted to the ledger that caused this block to be created. |

### `tx` Field Schemas

#### For `122burn`

| Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `from`       | `Array(vec { Blob [, Blob] })` | Yes      | The account from which tokens are burned. |
| `amount`     | `Nat`                         | Yes      | The number of tokens to burn. |
| `reason`     | `Text`                        | Yes      | Human-readable reason for the burn. |

#### For `122mint`

| Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `to`         | `Array(vec { Blob [, Blob] })` | Yes      | The account to which tokens are minted. |
| `amount`     | `Nat`                         | Yes      | The number of tokens to mint. |
| `reason`     | `Text`                        | Yes      | Human-readable reason for the mint. |

## Usage

### Purpose of the New Mint and Burn Blocks

The `122mint` and `122burn` blocks introduced in ICRC-122 serve as an updated method for tracking token supply changes compared to the mint and burn blocks from ICRC-1.

- **ICRC-1**: In the original ICRC-1 standard, mint and burn events were recorded in a much more basic form, typically involving only the amount of tokens and the affected account. However, this method lacked detailed information on the transaction context and authorization.

- **ICRC-122**: With ICRC-122, each mint and burn event is recorded with a more comprehensive structure, ensuring full transparency. The `tx` field in particular captures all the necessary transaction details, such as the involved accounts and the reason for the transaction. This allows for better auditing, validation, and traceability of token supply modifications. The new format ensures that the system is extensible, allowing for future use cases where additional information (e.g., authorization principals) may need to be recorded.

### Purpose of the `tx` Field

The `tx` field in the block schema is designed to fully capture the transaction that resulted in the minting or burning of tokens. The key purpose of this field is to preserve enough information to reconstruct the original transaction. This means that, if necessary, external parties or auditors should be able to verify the context of the mint or burn by looking at the `tx` data.

For example:
- If the transaction includes an additional principal (such as an authority or administrative account) that authorized the minting or burning, this principal needs to be included in the `tx` record.
- The flexibility of the `tx` field allows the system to accommodate future changes or new requirements, such as additional metadata or other necessary details about the transaction.

The `tx` field ensures that the ledger remains transparent and fully auditable while also leaving room for future extensibility.

### Minimal Format

The current format for both `122mint` and `122burn` blocks is intentionally minimal. It records only the essential data needed to track the effect on balances and token supply:
- The **amount** of tokens affected (either minted or burned).
- The **from** or **to** account, depending on whether tokens are being burned or minted.
- The **reason** for the burn or mint, providing context to the event.

This minimalism ensures that the ledger is efficient while still recording sufficient information for auditing and verifying the impact of each transaction.


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

        // The reason for the burn
        record { "reason"; variant { Text = "Token supply adjustment" }};
    }}};
}};
```

### 123mint example

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

        // The reason for the mint
        record { "reason"; variant { Text = "Initial distribution" }};
    }}};
}};
```
