# ICRC-122: Token Burning & Minting (Consolidated)

ICRC-122 introduces new block types for recording token minting and burning events in ICRC-compliant ledgers. These blocks provide a standardized way to document authorized supply modifications, ensuring transparent tracking of token issuance and removal. The `122burn` block records token reductions, while the `122mint` block tracks token increases. The transaction details (`tx`) within each block explicitly include the `caller` principal that authorized the operation.

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`, the owner's principal).
- **Principals** (such as the `caller`) are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes information about the token mint or burn transaction, including the `caller` principal, the data required for balance calculation, and basic context.

**Important Note on Transaction Recoverability:** The `tx` field defined below now includes the `caller` principal. However, for full auditability and transparency in complex scenarios, ledger implementations compliant with ICRC-122 **MUST** ensure that any other details of the original transaction invocation not captured in `tx` can be recovered independently. This could include, but is not limited to, the full arguments passed to the ledger method (if more complex than the data in `tx`), or any intermediary calls if the operation was part of a multi-step process. Mechanisms for recovering such extended data (e.g., via archive queries or specific lookup methods) remain implementation-dependent.

Each `122burn` or `122mint` block consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"122burn"` or `"122mint"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes information about the token mint or burn transaction, including the caller. See schemas below. |

### `tx` Field Schemas

#### For `122burn`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method (e.g., `icrc122_burn`) causing this block to be created. |
| `from`       | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account from which tokens are burned. |
| `amount`     | `Nat`                                                        | Yes      | The number of tokens to burn. This value **MUST be greater than 0**. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for the burn. |

#### For `122mint`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method (e.g., `icrc122_mint`) causing this block to be created. |
| `to`         | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account to which tokens are minted. |
| `amount`     | `Nat`                                                        | Yes      | The number of tokens to mint. This value **MUST be greater than 0**. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for the mint. |

¹ Where `V1` is `variant { Blob = <owner_principal> }` and `V2` is `variant { Blob = <subaccount> }`. If no subaccount exists, the `Array` contains only `V1`.

## Usage

### Comparison with ICRC-1 Mint/Burn

The `122mint` and `122burn` blocks introduced in ICRC-122 serve as a standardized method for tracking token supply changes, offering more structure than the implicit mint/burn events inferred from ICRC-1 transfer blocks involving the zero account.

- **ICRC-1**: In the original ICRC-1 standard, mint and burn events were typically represented by transfers to or from a conventionally designated zero account or address. While functional, this lacked explicit block types for supply changes and didn't standardize fields for context (like a reason or the authorizer of the supply change) or facilitate easy querying of *only* mint/burn events.

- **ICRC-122**: This standard defines explicit `122mint` and `122burn` block types. The `tx` field within each block includes the `caller` principal who initiated the supply change, the essential data for the operation (account, amount), and an optional `reason` field for human-readable context. This allows for clearer auditing, validation, and direct querying of supply modification events and their authorizers recorded on the ledger.

### Purpose of the `tx` Field

The `tx` field within `122mint` and `122burn` blocks is designed to provide a comprehensive summary of the transaction directly within the block:
- The `caller` identifies the principal that invoked the ledger operation.
- The `from` (burn) or `to` (mint) account specifies the target of the operation.
- The `amount` of tokens affected.
- An optional `reason` for the operation provides human-readable context.

This structure ensures that key information about the operation and its initiator is directly available in the `tx` field, enhancing transparency. For further details beyond what's in the block, such as the full original arguments passed to the ledger method (if applicable), implementations should provide separate recovery mechanisms as highlighted in the "Important Note on Transaction Recoverability."

## Compliance Reporting

Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```candid
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "122burn" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "122mint" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md)" }}; // Placeholder URL
    }};
}

```

## Example Blocks

### 122burn Example

```candid
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "122burn" }};

    // Timestamp when the block was added (nanoseconds since Unix epoch)
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }}; // Approx 2025-04-14T14:32:27Z 

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\6f\7e\d9\13\75\69\91\bc\d0\0d\04\b6\60\7b\82\f9\e9\62\a8\39\d3\02\80\f2\88\e4\d7\0e\23\2d\29\87"
    }};

    // The minimal burn transaction details
    record { "tx"; variant { Map = vec {
        // The account from which tokens are burned (owner + subaccount)
        record { "from"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // Example owner principal
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Example subaccount
        }}};

        // The amount to burn
        record { "amount"; variant { Nat = 1_000_000 : nat }};

        // Optional reason for the burn
        record { "reason"; variant { Text = "Token supply adjustment" }};
    }}};
}};
```

### 122mint Example

```candid
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "122mint" }};

    // Timestamp when the block was added (nanoseconds since Unix epoch)
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }}; // Approx 2025-04-14T14:32:27Z

    // Hash of the previous block in the ledger chain
    record { "phash"; variant {
        Blob = blob "\7a\61\cf\18\a4\9d\ac\20\39\33\78\f1\c2\fd\45\81\ab\55\37\20\1d\fe\77\a3\c6\55\de\01\92\a4\3b\ee"
    }};

    // The minimal mint transaction details
    record { "tx"; variant { Map = vec {
        // The account to which tokens are minted (owner + subaccount)
        record { "to"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };  // Example owner principal
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }  // Example subaccount
        }}};

        // The amount to mint
        record { "amount"; variant { Nat = 2_000_000 : nat }};

        // Optional reason for the mint
        record { "reason"; variant { Text = "Initial distribution" }};
    }}};
}};
```
