# ICRC-107: Fee Management

## Introduction & Motivation

Different ICRC-based ledgers (e.g., ckBTC) handle fee collection inconsistently.  
Some assign a designated fee collector for transfer operations, while others burn fees for approve operations—even when fee collection details appear in approve blocks.

This inconsistency creates challenges:

- Fee collection rules are not uniformly defined or visible on-chain.  
- Wallets and explorers cannot reliably predict where fees go.  
- Integration requires off-chain metadata to interpret fee behavior.  

**ICRC-107** standardizes fee collection across all ICRC-based ledgers by introducing:

- A **global fee collector configuration** stored in the ledger state.  
- A new block type (`107feecol`) for setting the fee collector on-chain.  
- A consistent fee application model integrated into the ICRC-3 evaluation model.  

This ensures **transparency**, **interoperability**, and **simplified integration**.  

ICRC-107 applies to all ICRC-based ledgers ensuring consistent fee collection behavior across transfers and approvals. This standard is backward-compatible with existing ICRC-based ledgers, allowing them to transition gradually to the new fee collection mechanism, while documenting legacy fee collection logic.

## Overview

ICRC-107 introduces a global on-chain fee collection setting recorded in the ledger via a new block type `btype = 107feecol`. This setting determines whether the effective fee of fee-charging blocks is credited to a designated account or burned. All changes are recorded on-chain for transparent, interoperable processing by wallets and explorers.

Specifically, ICRC-107 defines:

- A fee collection configuration that specifies the collector account (`icrc107_fee_col`), applying to all subsequent blocks.  
- A new block type (`btype = 107feecol`) for setting `icrc107_fee_col` to designate the fee collector.  
- A backward-compatible mechanism where, until `icrc107_fee_col` is set, legacy `fee_col` logic applies.  

By embedding fee collection settings entirely on-chain, ICRC-107 ensures **transparency**, **interoperability**, and **simplified integration** for wallets, explorers, and other tools.  

### Effective Fee Application 

For any block that charges a fee, handle the fee using this **checklist in order**:

1) **Find the current 107feecol setting for this block**  
   Look **backwards** from this block’s index to the most recent `107feecol` block (if any).  
   - If you find one with `tx.icrc107_fee_col = <Account>`, the current setting is **Account**.  
   - If you find one with `tx.icrc107_fee_col = []`, the current setting is **Burn**.  
   - If you find **none**, there is **no 107feecol setting** for this block.

2) **Apply the setting**  
   - **Account found:** credit the **effective fee** to that account.  
   - **Burn found (`[]`):** burn the effective fee.  
   - **No 107feecol setting found:** apply **legacy `fee_col` logic** (Section 5).  
     - If legacy config designates a collector, credit that account.  
     - If no legacy collector is configured, burn the fee.

> **Important:** “No 107feecol setting found” is **not** the same as `[]`.  
> `[]` means “we explicitly burn from now on.”  
> “No setting found” means “use legacy rules.”

3) **Non-retroactivity**  
   A `107feecol` block affects only blocks at its index **and after**. Earlier blocks keep whatever applied at their own indices (legacy or prior `107feecol`).


## Common Elements

This standard follows the conventions set by ICRC-3, inheriting key structural components.

- **Accounts**  
  Represented using the ICRC-3 `Value` type as a  
  `variant { Array = vec { V1 [, V2] } }` where:  
  - `V1` = `variant { Blob = <owner_principal_bytes> }` (the account owner),  
  - `V2` = `variant { Blob = <32-byte_subaccount_bytes> }` (optional).  
  If no subaccount is specified, the `Array` MUST contain only the owner’s principal.

- **Principals**  
  Represented as `variant { Blob = <principal_bytes> }`.

- **Timestamps**  
  Both `ts` and `tx.created_at_time` are expressed in **nanoseconds since the Unix epoch**.  
  They are encoded as `Nat` in ICRC-3 `Value`, but MUST fit into a `nat64`.  
  - `ts` is set by the ledger when the block is created.  
  - `created_at_time` is provided by the caller for deduplication (per ICRC-1).

- **Parent Hash**  
  Each block includes `phash : Blob`, the hash of its parent block,  
  unless it is the genesis block (where `phash` is omitted).



##  Fee Management






### ICRC-107 Transaction and Block Schema

Each `107feecol` block consists of the following top-level fields:

| Field             | Type (ICRC-3 `Value`) | Required | Description                                                                                                                                                                                                                            |
|-------------------|------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `btype`           | `Text`                 | Yes      | **MUST** be `"107feecol"`.                                                                                                                                                                                                                 |
| `ts`              | `Nat`                  | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the block was added to the ledger.                                                                                                                                                |
| `phash`           | `Blob`                 | Yes      | Hash of the parent block.                                                                                                                                                                                                              |
| `tx`              | `Map(Text, Value)`     | Yes      | Encodes information about the fee collection configuration transaction. See `tx` Field Schema below.                                                                                                                                   |

### Minimal `tx` Schema

The minimal fields required to interpret a `107feecol` block:

| Field             | Type (ICRC-3 `Value`) | Required | Description |
|-------------------|------------------------|----------|-------------|
 | `icrc107_fee_col` | `Array` (Account/Empty)| Yes      | Target fee collector account, or empty (`[]`) to burn fees. |


## Semantics of `107feecol` Blocks

### Core State Transition

A `107feecol` block updates the ledger’s global fee collector configuration:

- If `tx.icrc107_fee_col = []` → all subsequent fees are **burned**.  
- If `tx.icrc107_fee_col` encodes an account → all subsequent fees are **credited to that account**.  

This configuration applies **to all subsequent blocks** until replaced by another `107feecol`.

---

### Fee Application under ICRC-107

For every block that charges a fee:

1. Execute the **pre-fee state transition** (per ICRC-3 evaluation model).  
2. Compute the **effective fee** (as defined in ICRC-3).  
3. Debit the **effective fee** from the block’s **fee payer**.  
4. Determine how the effective fee is handled:

   - **If at least one `107feecol` block exists:**  
     • If the most recent `107feecol` has `tx.icrc107_fee_col = []` → burn the effective fee.  
     • If the most recent `107feecol` has `tx.icrc107_fee_col = <Account>` → credit the effective fee to that account.  

   - **If no `107feecol` block exists yet (legacy mode):**  
     • Apply the legacy rules described in the *Legacy Fee Collection* section.  
     • In brief: if `fee_col` is unset, burn all fees; if `fee_col` is set, only transfer fees are credited to that account, and all other fees are burned.  

This ensures that fee handling is always well-defined, both before and after the introduction of ICRC-107.

---

**Notes**

- Invariants from ICRC-3 still apply (e.g., `effective fee ≤ amt` for mints; sufficient balance for `amt + effective fee`; allowance reductions include `amt + effective fee` where applicable).  
- ICRC-107 does **not** define who the fee payer is — that comes from the semantics of each block type (e.g., ICRC-1/2 legacy rules in ICRC-3).





# ICRC-107: Methods for Setting and Getting Fee Collector Information


### `icrc107_set_fee_collector`

This method allows a ledger controller to update the fee collector. It modifies the `icrc107_fee_col` account, which determines where collected fees are sent. The updated settings are recorded in a new block (of type `107feecol`) added to the ledger.

```

type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type SetFeeCollectorArgs = record {
    icrc107_fee_col: opt Account;
    created_at_time: nat64;
};

type SetFeeCollectorError = variant {
    AccessDenied : text; // The caller is not authorized to modify fee collector.
    InvalidAccount : text; // The provided account for fee collection is invalid (e.g., minting account, anonymous principal, malformed).
    Duplicate : record { duplicate_of : nat }; // A duplicate transaction already exists at position `duplicate_of` in the ledger.
    GenericError : record { error_code : nat; message : text };
};

icrc107_set_fee_collector: (SetFeeCollectorArgs) -> (variant { Ok: nat; Err: SetFeeCollectorError });

```

### `icrc107_set_fee_collector` — Semantics (clarified link to `tx`)

This method MUST only be callable by a controller of the ledger or other authorized principals. Unauthorized calls SHOULD result in an error.

- On success, the ledger MUST append a new block with `btype = "107feecol"`.  
  **The block’s `tx` is a top-level field of this `107feecol` block** and **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** below (same keys, types, and encoding). The ledger MUST embed this `tx` map into the newly appended block.
- The value in `tx.icrc107_fee_col` becomes the active fee-collection setting **from that block’s index onward** (non-retroactive).  
  - If `icrc107_fee_col` is provided (`Account`), all subsequent effective fees are **credited** to that account until changed by a later `107feecol`.  
  - If `icrc107_fee_col` is not provided (`null`), it MUST be encoded as `[]` in `tx`; all subsequent effective fees are **burned** until changed by a later `107feecol`.
- If multiple `107feecol` blocks exist, the **last one at or before** a block’s index determines the setting for that block.
- The ledger MUST perform deduplication (e.g., using `created_at_time` and implementation-defined inputs). On duplicate detection, it MUST NOT append a new block and MUST return `Err(Duplicate { duplicate_of = <index> })`.

The `icrc107_set_fee_collector` method MUST return an error in the following cases:

- The caller is not authorized to modify the fee collector (`AccessDenied`).  
- The provided `Account` is invalid (e.g., minting account, anonymous principal, malformed principal or subaccount) (`InvalidAccount`).  
- A semantically identical transaction was already accepted (per deduplication rules), resulting in `SetFeeCollectorError::Duplicate` (with `duplicate_of`).  
- Any other failure that prevents appending a valid `107feecol` block (`GenericError`).

---

### Canonical `tx` Mapping (normative and referenced above)

**Scope:** This mapping defines the **exact** content and encoding of the **top-level** `tx` field that MUST be embedded in every `107feecol` block created by `icrc107_set_fee_collector` (or by the ledger itself in the system-generated case).

| Field             | Type (ICRC-3 `Value`)   | Source / Encoding Rule                                                                 |
|-------------------|-------------------------|-----------------------------------------------------------------------------------------|
| `icrc107_fee_col` | `Array` (Account/Empty) | From the `icrc107_fee_col` argument. If `null`, **encode as** `Array = []`. If present, encode the Account. |
| `created_at_time` | `Nat`                   | From the `created_at_time` argument (ns since Unix epoch; **MUST** fit in `nat64`).     |
| `caller`          | `Blob`                  | Principal of the caller.                                                                |

> **Conformance note:** Any `107feecol` block produced by this method **MUST** carry a top-level `tx` whose map entries **exactly match** the table above (no additional required keys; optional keys, if any, MUST follow this spec). Consumers MUST interpret fee behavior from this `tx` content.

#### System-Generated `107feecol` Blocks

For blocks created by the ledger itself (e.g., during an upgrade/migration):

- MUST include top-level `tx.icrc107_fee_col` (encoded as above).  
- MAY omit `tx.created_at_time` and `tx.caller`.  
- If `tx.caller` is included, it SHOULD be the ledger canister principal.



### `icrc107_get_fee_collector`

This method retrieves the currently active fee collector account. Unless changed, these settings apply to the next block added to the ledger.

```
icrc107_get_fee_collector: () -> (variant { Ok: opt Account; Err: record { error_code : nat; message : text } }) query;
```


This method should return the currently active fee collector account:

  - If the response is `null`, fees are burned. This corresponds to `icrc107_fee_col = variant { Array = vec {}}` on-chain.
  - If the response is a valid `Account`, fees are collected by that account. This corresponds to `icrc107_fee_col` being set to the ICRC-3 representation of the account on-chain.

This method strictly returns the last explicitly set value of `icrc107_fee_col`. It does not infer defaults, and if no fee collector was ever set, it returns `opt Account = null`.







#### Example Block

Below is an example of a valid **fee collector configuration block** that sets the fee collector to a specific account:

```
variant { Map = vec {
    // Block type: indicates this is a fee collector configuration block
    record { "btype"; variant { Text = "107feecol" }};
    // The user's intended transaction
    record { "tx"; variant { Map = vec {
        record { "op"; variant { Text = "107_set_fee_col" } }; // Operation name, as per schema
        record { "icrc107_fee_col"; variant { Array = vec {
            variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"}; // Owner principal
            variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Subaccount
        }} };
        record { "created_at_time"; variant { Nat = 1_750_951_727_000_000_000 : nat } }; // Timestamp for deduplication (June 27, 2025, 15:28:47 UTC)
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" } }; // Caller principal
    }} };
    // Timestamp: indicates when the block was created
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51"
              }};
}};
```
If one wanted to set the fee collector to be the default account of principal
identified by `"\00\00\00\00\02\00\01\0d\01\01"`, then the
`"icrc107_fee_col"` field within `tx` should be set to:

variant { Array = vec { variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"} } }




A block that explicitly sets fee burning by removing the fee collector (i.e., all fees are burned from this point onward):

```
variant { Map = vec {
    // Block type: indicates this is a fee collector configuration block
    record { "btype"; variant { Text = "107feecol" }};
    // The user's intended transaction, from which the hash is computed
    record { "tx"; variant { Map = vec {
        record { "op"; variant { Text = "107_set_fee_col" } }; // Operation name, as per schema
        record { "icrc107_fee_col"; variant { Array = vec {
                // Empty array to signify burning fees
            }}};
        record { "created_at_time"; variant { Nat = 1_750_951_728_000_000_000 : nat } }; // Timestamp for deduplication
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" } }; // Caller principal
    }} };
    // Timestamp: indicates when the block was created
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    // Parent hash: links this block to the previous block in the chain
    record { "phash";
              variant {
                Blob = blob "\2d\86\7f\34\c7\2d\1e\2d\00\84\10\a4\00\b0\b6\4c\3e\02\96\c9\e8\55\6f\dd\72\68\e8\df\8d\8e\8a\ee"
              }};
}};
```



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

##  Legacy Fee Collection Mechanism

The Dfinity maintained ICRC ledgers include a fee collection mechanism which, for completeness is described below.

###  Legacy Behavior (`fee_col`)


Until the first block of type `107feecol` the ledger follows the following legacy behavior.

- If `fee_col`is not set, all fees are burned.
- If `fee_col` is set in a block, the designated account collects only transfer fees (`1xfer`, `2xfer`). Fees for all other operations (e.g., `2approve`) were always burned in legacy behavior as implemented in Dfinity-maintained ICRC-3 ledgers.


New implementations SHOULD avoid using `fee_col` and instead use `icrc107_fee_col` for all fee collection settings. Legacy behavior is provided for backward compatibility only and MAY be deprecated in future versions of this standard.

###  Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

To determine who collects the fee in a block:

1. Check for fee collector configuration

   - If a previous block set `icrc107_fee_col`, the ledger uses the behavior specified by this standard.

2. If no `icrc107_fee_col` setting exists (legacy behavior):

   - If the block is of type `2approve` then the fee is burned
   - If the block is a transfer block, i.e. of type `1xfer` or `2xfer`:
      - If `fee_col` is specified in the block the fee is collected by `fee_col`.
      - If `fee_col_block` is specified use the `fee_col` from the referenced block index.
      - If netiher `fee_col` nor `fee_col_block` are specified, then the fee is burned.
