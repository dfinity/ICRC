# ICRC-122: Token Burning & Minting

ICRC-122 introduces new block types for recording token minting and burning events in ICRC-compliant ledgers. These blocks provide a standardized way to document authorized supply modifications, ensuring transparent tracking of token issuance and removal. The `122burn` block records token reductions, while the `122mint` block tracks token increases. 

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`, the owner's principal).
- **Principals** (such as the `caller`) are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes information about the mint or burn operation sufficient to compute balances. Additional provenance data MAY be included but is not required for semantics.


Each `122burn` or `122mint` block consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"122burn"` or `"122mint"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes information about the token mint or burn transaction sufficient to compute balances. See schemas below. |

### `tx` Field Schemas

#### For `122burn`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `from`       | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account from which tokens are burned. |
| `amt`     | `Nat`                                                        | Yes      | The number of tokens to burn. This value **MUST be greater than 0**. |


#### For `122mint`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `to`         | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account to which tokens are minted. |
| `amt`     | `Nat`                                                        | Yes      | The number of tokens to mint. This value **MUST be greater than 0**. |

## Usage

### Comparison with ICRC-1 Mint/Burn  

The `122mint` and `122burn` blocks introduced in ICRC-122 provide a more explicit and structured mechanism for tracking token supply changes compared to how these operations are represented under ICRC-1. A ledger that supports both ICRC-1 and ICRC-122 may therefore produce blocks of both kinds, reflecting distinct ways of invoking supply modifications.  

- **ICRC-1 Mint/Burn Mechanism**:  
  - *Operation & Block Type*: ICRC-1 defines a `minting_account`. Minting is achieved by an `icrc1_transfer` from this account to a recipient, increasing total supply. Burning is an `icrc1_transfer` to the `minting_account`, reducing supply. ICRC-3-compliant ledgers typically encode these as legacy blocks with `op = "mint"` or `op = "burn"`.  
  - *Authorization*: Control of the `minting_account` implies authority to mint. Only the principal able to submit `icrc1_transfer` calls from this account can change supply.  
  - *Contextual Data*: These legacy blocks do not have a dedicated field for the supply change rationale; any context must be encoded in the generic `memo`. They also do not explicitly identify the ultimate caller beyond the fact that the `from` account was the `minting_account`.  

- **ICRC-122 Mint/Burn Blocks**:  
  - *Typed Blocks*: This standard defines distinct block types, `122mint` and `122burn`. They are not special cases of `icrc1_transfer`; they represent supply changes directly.  
  - *Minimal Structure*: The minimal `tx` structure contains only the target account (`to` for mint, `from` for burn) and the amount (`amt`). These fields are sufficient to determine the ledger’s state transition.  
  - *Optional Provenance*: Producers may include additional non-semantic fields in `tx` such as `caller` (the principal that invoked the supply change), `reason` (a human-readable string), or `created_at_time`. These fields aid transparency and auditability but do not affect ledger semantics.  
  - *Auditability*: By separating the minimal state-changing fields from optional provenance, ICRC-122 ensures clear semantics while still supporting enhanced transparency compared to the ICRC-1 mechanism.  

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

The following examples illustrate how `122burn` and `122mint` blocks are encoded on-chain.  
Each operation is shown in two forms:

- **Minimal form**, which includes only the required fields needed to define the state change.  
- **Extended form**, which demonstrates how optional provenance fields such as `caller`, `reason`, or `created_at_time` may be included to enhance transparency and auditability.  

These examples are intended to guide implementers and tool builders in interpreting both the essential and optional elements of ICRC-122 blocks.


### 122burn Example

### 122burn (Minimal Example)  
The following block shows the minimal `122burn` structure with only the required `from` and `amt` fields inside the `tx` map.  



```
variant { Map = vec {
    record { "btype"; variant { Text = "122burn" }};
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }}; 
    record { "phash"; variant { Blob = blob "\6f\7e\d9\13\75\69\91\bc\d0\0d\04\b6\60\7b\82\f9\e9\62\a8\39\d3\02\80\f2\88\e4\d7\0e\23\2d\29\87" }};
    record { "tx"; variant { Map = vec {
        record { "from"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
        }}};
        record { "amt"; variant { Nat = 1_000_000 : nat }};
    }}};
}}
```


### 122burn (Extended Example with Provenance and Deduplication Information)  
This block demonstrates an extended `122burn` including optional provenance fields (`caller`, `reason`, `created_at_time`) alongside the required fields. These additions provide auditability without altering semantics.  



```
variant { Map = vec {
    record { "btype"; variant { Text = "122burn" }};
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }};
    record { "phash"; variant { Blob = blob "\6f\7e\d9\13\75\69\91\bc\d0\0d\04\b6\60\7b\82\f9\e9\62\a8\39\d3\02\80\f2\88\e4\d7\0e\23\2d\29\87" }};
    record { "tx"; variant { Map = vec {
        record { "from"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
        }}};
        record { "amt"; variant { Nat = 1_000_000 : nat }};
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" }};
        record { "reason"; variant { Text = "Token supply adjustment" }};
        record { "created_at_time"; variant { Nat = 1_741_317_146_900_000_000 : nat }};
    }}};
}}
```

### 122mint (Minimal Example)  
The following block shows the minimal `122mint` structure with only the required `to` and `amt` fields inside the `tx` map.  


```
variant { Map = vec {
    record { "btype"; variant { Text = "122mint" }};
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }};
    record { "phash"; variant { Blob = blob "\7a\61\cf\18\a4\9d\ac\20\39\33\78\f1\c2\fd\45\81\ab\55\37\20\1d\fe\77\a3\c6\55\de\01\92\a4\3b\ee" }};
    record { "tx"; variant { Map = vec {
        record { "to"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
        }}};
        record { "amt"; variant { Nat = 2_000_000 : nat }};
    }}};
}}
```

---

### 122mint (Extended Example with Provenance and Deduplication Information)  
This block demonstrates an extended `122mint` including optional provenance fields (`caller`, `reason`, `created_at_time`) alongside the required fields. These additions provide auditability without altering semantics.  


```
variant { Map = vec {
    record { "btype"; variant { Text = "122mint" }};
    record { "ts"; variant { Nat = 1_741_317_147_000_000_000 : nat }};
    record { "phash"; variant { Blob = blob "\7a\61\cf\18\a4\9d\ac\20\39\33\78\f1\c2\fd\45\81\ab\55\37\20\1d\fe\77\a3\c6\55\de\01\92\a4\3b\ee" }};
    record { "tx"; variant { Map = vec {
        record { "to"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
        }}};
        record { "amt"; variant { Nat = 2_000_000 : nat }};
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" }};
        record { "reason"; variant { Text = "Initial distribution" }};
        record { "created_at_time"; variant { Nat = 1_741_317_146_900_000_000 : nat }};
    }}};
}}
```

