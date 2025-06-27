# ICRC-107: Fee Collection

## Introduction & Motivation

Different ICRC-based ledgers (e.g., ckBTC) handle fee collection inconsistently. Some ledgers assign a designated fee collector for transfer operations, while others burn fees for approve operations—even when fee collection details appear in approve blocks. This inconsistency leads to challenges in defining fee collection rules, recording fee settings on-chain, and ensuring clear semantics for wallets and explorers. ICRC-107 builds on the ICRC-3 block schema, introducing a new block type (`107feecol`) to configure fee collection settings on-chain. This ensures compatibility with existing ICRC-3 implementations while extending functionality for fee handling.
By embedding fee collection settings on-chain, ICRC-107 provides the following benefits:

- Transparency: Fee collection rules are visible and auditable on the ledger.

- Interoperability: Wallets and explorers can rely on consistent fee handling across ICRC-based ledgers.

- Simplified Integration: Eliminates the need for off-chain metadata to determine fee collection behavior.

ICRC-107 applies to all ICRC-based ledgers ensuring consistent fee collection behavior across transfers and approvals. This standard is backward-compatible with existing ICRC-based ledgers, allowing them to transition gradually to the new fee collection mechanism, while documenting legacy fee collection logic.

### ICRC-107 Proposal

ICRC-107 establishes an on-chain fee collection mechanism that ensures clarity and interoperability across different ICRC-based ledgers. It defines:

- A fee collection configuration that specifies the collector account (`icrc107_fee_col`), applying to all subsequent blocks.
- A new block type for setting `icrc107_fee_col` to designate the fee collector.
- A backward-compatible mechanism where, until `icrc107_fee_col` is set, legacy `fee_col` logic applies.

By embedding fee collection settings entirely on-chain, ICRC-107 ensures transparency and simplifies integration with external tools.


## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`, the owner's principal).
- **Principals** (such as the `caller`) are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.



---

##  Fee Collection

### Overview

Each block requires a fee, which a designated party must pay based on the operation type.

The fee collection configuration controls how the ledger processes fees. This configuration consists of a single global fee collector account (`icrc107_fee_col`). When a block updates this setting, the change takes effect immediately.

- If `icrc107_fee_col` is set to a ledger account, that account collects all subsequent fees.
- If `icrc107_fee_col` is set to the empty account (see below), the ledger burns all subsequent fees.
- An `icrc107_fee_col` block configures the fee collection for all subsequent blocks, until superseded by another `icrc107_fee_col` block.
- Until `icrc107_fee_col` is set fees are burned, unless legacy `fee_col` logic applies (see Section 5).
- A **fee collector configuration block** records these settings on-chain, ensuring transparent fee collection.

Once `icrc107_fee_col` is set, it overrides any legacy fee collection logic that may be in place (See Section 5).

Fee burning is explicitly recorded on-chain by setting `icrc107_fee_col = variant { Array = vec {} }`. This ensures unambiguous representation across implementations.




### ICRC-107 Transaction and Block Schema

Each `107feecol` block consists of the following top-level fields:

| Field             | Type (ICRC-3 `Value`) | Required | Description                                                                                                                                                                                                                            |
|-------------------|------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `btype`           | `Text`                 | Yes      | **MUST** be `"107feecol"`.                                                                                                                                                                                                                 |
| `ts`              | `Nat`                  | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the block was added to the ledger.                                                                                                                                                |
| `phash`           | `Blob`                 | Yes      | Hash of the parent block.                                                                                                                                                                                                              |
| `tx`              | `Map(Text, Value)`     | Yes      | Encodes information about the fee collection configuration transaction. See `tx` Field Schema below.                                                                                                                                   |


### `tx` Field Schema

The `tx` field for a `107feecol` block is a `Map` that contains the following fields:

| Field           | Type (ICRC-3 `Value`) | Required | Description                                                                                                                                           |
|-----------------|------------------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `op`            | `Text`                 | Yes      | **MUST** be `"set107feecol"`. Explicitly identifies the operation and standard within the transaction, enabling self-contained transaction records and unambiguous hashing. |
| `fee_col` | `Array` (Account)      | Yes      | The target fee collector account. If `variant { Array = vec {} }`, it signifies that fees should be burned. See Common Elements for Account representation. |
| `created_at_time` | `Nat`                | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the user created the transaction. This value **MUST** be provided by the caller to enable transaction deduplication as per ICRC-1. |
| `caller` |  `Blob` | Yes | The principal of the user or canister that originated this transaction. |





#### Example Block

Below is an example of a valid **fee collector configuration block** that sets the fee collector to a specific account:

```
variant { Map = vec {
    // Block type: indicates this is a fee collector configuration block
    record { "btype"; variant { Text = "107feecol" }};
    // The user's intended transaction
    record { "tx"; variant { Map = vec {
        record { "op"; variant { Text = "107setfeecol" } }; // Operation name, as per schema
        record { "fee_col"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"}; // Owner principal
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Subaccount
        }} };
        record { "created_at_time"; variant { Nat = 1_750_951_727_000_000_000 : nat } }; // Timestamp for deduplication (June 27, 2025, 15:28:47 UTC)
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" } }; // Example caller principal
    }} };
    // Timestamp: indicates when the block was created
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } }; // Existing block timestamp
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```
If one wanted to set the fee collector to be the default account of principal identified by `"\00\00\00\00\02\00\01\0d\01\01"`, then the "fee_col" field within `tx`  should be set to variant ``{ Array = vec {
    variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"} } }``.



A block that explicitly sets fee burning by removing the fee collector (i.e., all fees are burned from this point onward):

```
variant { Map = vec {
    // Block type: indicates this is a fee collector configuration block
    record { "btype"; variant { Text = "107feecol" }};
    // The user's intended transaction, from which the hash is computed
    record { "tx"; variant { Map = vec {
        record { "op"; variant { Text = "set107feecol" } }; // Operation name, as per schema
        record { "fee_col"; variant { Array = vec {
                // Empty array to signify burning fees
            }}};
        record { "created_at_time"; variant { Nat = 1_750_951_728_000_000_000 : nat } }; // Timestamp for deduplication (incremented from previous example)
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" } }; // Example caller principal
    }} };
    // Timestamp: indicates when the block was created (can be derived from tx.created_at_time or ledger time)
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } }; // Existing block timestamp
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\2d\86\7f\34\c7\2d\1e\2d\00\84\10\a4\00\b0\b6\4c\3e\02\96\c9\e8\55\6f\dd\72\68\e8\df\8d\8e\8a\ee"
              }};
}};
```



---

# ICRC-107: Methods for Setting and Getting Fee Collection

## Methods for Setting and Getting Fee Collection

### `icrc107_set_fee_collection`

This method allows a ledger controller to update the fee collection settings. It modifies the `icrc107_fee_col` account, which determines where collected fees are sent. The updated settings are recorded in a new block (of type `107feecol`) added to the ledger.

```

type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type SetFeeCollectionArgs = record {
    icrc107_fee_col: opt Account;
    created_at_time: opt nat64;
};

type SetFeeCollectionError = variant {
    AccessDenied : text; // The caller is not authorized to modify fee collection settings.
    InvalidAccount : text; // The provided account for fee collection is invalid (e.g., minting account, anonymous principal, malformed).
    Duplicate : record { duplicate_of : nat }; // A duplicate transaction already exists at position `duplicate_of` in the ledger.
    GenericError : record { error_code : nat; message : text };
};

icrc107_set_fee_collection: (SetFeeCollectionRequest) -> (variant { Ok: empty; Err: SetFeeCollectionError });

```

This method MUST only be callable by a controller of the ledger or some other authorized principals. Unauthorized calls SHOULD result in an error.

- The ledger MUST construct the tx field (an ICRC-3 Value of type Map as described in the previous section) from this request, place it in the generated `107feecol` block and append the block to the ledger.
- If `icrc107_fee_col` is set to an account, all subsequent fees are collected by that account.
- If `icrc107_fee_col` is not provided (or set to `null`), all subsequent fees are burned.

The `icrc107_set_fee_collection` method MUST return an error in the following cases:

- The caller is not authorized to modify fee collection settings.
- The provided `Account` is invalid (e.g., the minting account on ledgers,  anonymous principal, malformed principal or subaccount)."
- Transaction deduplication is enabled for the transaction (i.e. `created_at_time` is set) and an identical transaction already exist.


### `icrc107_get_fee_collection`

This method retrieves the currently active fee collection settings. Unless changed, these settings apply to the next block added to the ledger.

```
icrc107_get_fee_collection: () -> (variant { Ok: opt Account; Err: record { error_code : nat; message : text } }) query;
```


This method should return the currently active fee collection settings:

  - If the response is `null`, fees are burned. This corresponds to `icrc107_fee_col = variant { Array = vec {}}` on-chain.
  - If the response is a valid `Account`, fees are collected by that account. This corresponds to `icrc107_fee_col` being set to the ICRC-3 representation of the account on-chain.

This method strictly returns the last explicitly set value of `icrc107_fee_col`. It does not infer defaults, and if no fee collector was ever set, it returns `opt Account = null`.

---


##  Reporting Compliance

Compliance with the current standard is reported as follows.

###  Supported Standards

Ledgers implementing ICRC-107 **MUST** indicate their compliance through the `icrc1_supported_standards` and `icrc10_supported_standards` methods, by including in the output of these methods:

```
variant { Vec = vec {
    record {
        "name"; variant { Text = "ICRC-107" };
        "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107.md" };
    }
}};
```

###  Supported Block Types

Ledgers implementing ICRC-107 **MUST** report the new block type in response  to `icrc3_supported_block_types`.  The output of the call must include

```
variant { Vec = vec {
    record { "name"; variant { Text = "107feecol" }; "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107.md" } };
}};
```

##  Note on Legacy Fee Collection Mechanisms

The Dfinity maintained ICRC ledgers include a fee collection mechanism which, for completeness is described below.

###  Legacy Behavior (`fee_col`)


- Until `icrc107_fee_col` is set, the ledger follows this legacy behavior, using `fee_col` only for transfers.
- If `fee_col`is not set, all fees are burned.
- If `fee_col` is set in a block, the designated account collects only transfer fees (`1xfer`, `2xfer`). Fees for all other operations (e.g., `2approve`) were always burned in legacy behavior as implemented in Dfinity-maintained ICRC-3 ledgers.


New implementations SHOULD avoid using `fee_col` and instead use `icrc107_fee_col` for all fee collection settings. Legacy behavior is provided for backward compatibility only and MAY be deprecated in future versions of this standard.

###  Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

To determine who collects the fee in a block:

1. Check for fee collection configuration

   - If a previous block set `icrc107_fee_col`, the ledger uses the behavior specified by this standard.

2. If no `icrc107_fee_col` setting exists (legacy behavior):

   - If the block is of type `2approve` then the fee is burned
   - If the block is a transfer block, i.e. of type `1xfer` or `2xfer`:
      - If `fee_col` is specified in the block the fee is collected by `fee_col`.
      - If `fee_col_block` is specified use the `fee_col` from the referenced block index.
