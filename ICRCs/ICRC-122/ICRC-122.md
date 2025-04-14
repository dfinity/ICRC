# ICRC-122: Token Burning & Minting (Consolidated)

ICRC-122 introduces new block types for recording token minting and burning events in ICRC-compliant ledgers. These blocks provide a standardized way to document authorized supply modifications, ensuring transparent tracking of token issuance and removal. The `122burn` block records token reductions, while the `122mint` block tracks token increases.

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components. Accounts are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`, the owner's principal). Additionally, each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map that encodes the minimal information about the token mint or burn transaction required for balance calculation and basic context.

**Important Note on Transaction Recoverability:** The `tx` field defined below is intentionally minimal, containing only the data strictly necessary to update account balances, total supply, and provide a basic reason according to the block's semantics. For full auditability and transparency, ledger implementations compliant with ICRC-122 **MUST** ensure that the complete details of the original transaction invocation can be recovered independently. This includes, but is not limited to, the principal that invoked the ledger operation (the authorizer/caller), the specific ledger method called (e.g., `icrc122_burn`), and the full arguments passed to that method. Mechanisms for recovering this data (e.g., via archive queries or specific lookup methods) are implementation-dependent but necessary for compliance. The `tx` field itself is *not* designed to hold this exhaustive information.

Each `122burn` or `122mint` block consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"122burn"` or `"122mint"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes minimal information about the token mint or burn transaction. See schemas below. |

### `tx` Field Schemas

#### For `122burn`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `from`       | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account from which tokens are burned. |
| `amount`     | `Nat`                                                        | Yes      | The number of tokens to burn. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for the burn. |

#### For `122mint`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `to`         | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account to which tokens are minted. |
| `amount`     | `Nat`                                                        | Yes      | The number of tokens to mint. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for the mint. |

¹ Where `V1` is `variant { Blob = <owner_principal> }` and `V2` is `variant { Blob = <subaccount> }`. If no subaccount exists, the `Array` contains only `V1`.

## Usage

### Comparison with ICRC-1 Mint/Burn

The `122mint` and `122burn` blocks introduced in ICRC-122 serve as a standardized method for tracking token supply changes, offering more structure than the implicit mint/burn events inferred from ICRC-1 transfer blocks involving the zero account.

- **ICRC-1**: In the original ICRC-1 standard, mint and burn events were typically represented by transfers to or from a conventionally designated zero account or address. While functional, this lacked explicit block types for supply changes and didn't standardize fields for context (like a reason) or facilitate easy querying of *only* mint/burn events.

- **ICRC-122**: This standard defines explicit `122mint` and `122burn` block types. The minimal `tx` field standardizes the essential data (account, amount) and includes an optional `reason` field for human-readable context directly within the block relevant to the supply change. This allows for clearer auditing, validation, and direct querying of supply modification events recorded on the ledger.

### Purpose and Limitations of the `tx` Field

The `tx` field within `122mint` and `122burn` blocks is designed to be **minimal**. It records only the essential data needed to verify the effect on balances and token supply, along with an optional contextual reason:

- The **amount** of tokens affected.
- The **from** (burn) or **to** (mint) account.
- An optional **reason** for the operation.

This minimalism ensures efficiency while providing core audit data related to the supply change itself.

Crucially, the `tx` field **does not** aim to capture the full details of the transaction invocation that *caused* the block to be created (like the caller/authorizer principal or exact method arguments). As stated in the "Important Note on Transaction Recoverability" earlier, retrieving those comprehensive details for full auditing **must** be supported by the ledger implementation through other means, adhering to the principle of separating minimal block data from potentially larger contextual/invocation data.

## Compliance Reporting

Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```candid
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "122burn" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "122mint" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-122.md" }}; // Placeholder URL
    }};
}
```

## Example Blocks

### 122burn Example

```candid
variant { Map = vec {
    // Block type identifier
    record { "btype"; variant { Text = "122burn" }};

    // Timestamp when the block was added (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }}; // Approx 2025-04-14T14:32:27Z (Using current time)

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

    // Timestamp when the block was added (nanoseconds since epoch)
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }}; // Approx 2025-04-14T14:32:27Z (Using current time)

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
