# ICRC-122: Token Burning & Minting

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
| `ts`     | `Nat`                  | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the block was added to the ledger. |
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

The `122mint` and `122burn` blocks introduced in ICRC-122 offer a more explicit and structured method for tracking token supply changes compared to how these operations are handled under the ICRC-1 standard. It is important to note that a ledger implementing both ICRC-1 and ICRC-122 may feature blocks resulting from both mechanisms, reflecting the distinct operations invoked.

- **ICRC-1 Mint/Burn Mechanism**:
    - **Operation & Block Type**: ICRC-1 defines a `minting_account`. Minting is achieved by an `icrc1_transfer` from this `minting_account` to the recipient, increasing total supply. Burning is an `icrc1_transfer` to the `minting_account`, decreasing total supply. ICRC-3 compliant ledgers supporting ICRC-1 mint and burn operations typically record these events using specific block types, often designated as `1mint` and `1burn` respectively (or similar, such as `icrc1_mint` and `icrc1_burn`). These blocks, while distinct from standard transfers, primarily confirm the transfer details associated with the minting or burning action as defined by ICRC-1.
    - **Authorization**: Authorization for minting is tied to the control of the `minting_account`. Only the principal that can initiate `icrc1_transfer` calls from this specific account can mint tokens. This centralizes minting authority to the controller of the minting account.
    - **Contextual Data**: The ICRC-1 `1mint` and `1burn` block structures, as typically derived from the underlying `icrc1_transfer` operation, do not have dedicated fields for a structured `reason` for the supply change beyond what might be placed in the generic `memo` field of the transfer. They also do not explicitly identify an ultimate `caller` or authorizer of the mint/burn operation beyond the `from` field of the transfer (which would be the minting account itself).

- **ICRC-122 Mint/Burn Blocks**:
    - **Explicit Block Types & Dedicated Methods**: This standard defines distinct block types (`122mint`, `122burn`) which result from calls to new, dedicated ledger methods (e.g., `icrc122_mint`, `icrc122_burn`). These are not special cases of `icrc1_transfer`.
    - **Flexible Authorization & Direct Caller Identification**: Authorization to call the `icrc122_mint` or `icrc122_burn` endpoints can be managed by the ledger implementation's governance logic, potentially allowing multiple, distinct whitelisted principals to initiate minting or burning. The `caller` field within the `tx` map of each `122mint` or `122burn` block directly records the principal that invoked this specific supply change operation.
    - **Enhanced Context**: The `tx` map includes an optional `reason` field, providing a standardized way to include human-readable context specifically for the supply change.
    - **Auditability**: The combination of dedicated block types, distinct methods, direct recording of the `caller`, and the optional `reason` field significantly enhances the auditability and transparency of token supply modifications compared to the ICRC-1 mechanism.


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
