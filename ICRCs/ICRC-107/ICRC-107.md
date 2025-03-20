# ICRC-107: Fee Collection

## 1. Introduction & Motivation

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

---

## 2. Fee Collection

### 2.1 Overview

Each block requires a fee, which a designated party must pay based on the operation type.

The fee collection configuration controls how the ledger processes fees. This configuration consists of a single global fee collector account (`icrc107_fee_col`). When a block updates this setting, the change takes effect immediately.

- If `icrc107_fee_col` is set to a ledger account, that account collects all subsequent fees.
- If `icrc107_fee_col` is set to the empty account (see below), the ledger burns all subsequent fees.
- An `icrc107_fee_col` block configures the fee collection for all subsequent blocks, until superseded by another `icrc107_fee_col` block. 
- Until `icrc107_fee_col` is set fees are burned, unless legacy `fee_col` logic applies (see Section 5).
- A **fee collector configuration block** records these settings on-chain, ensuring transparent fee collection.

Once `icrc107_fee_col` is set, it overrides any legacy fee collection logic that may be in place (See Section 5).

Fee burning is explicitly recorded on-chain by setting `icrc107_fee_col = variant { Array = vec {} }`. This ensures unambiguous representation across implementations.


### 2.2 ICRC-107 Block Schema

ICRC-107 introduces a new block type that follows the ICRC-3 specification to configure `icrc107_fee_col`.

In addition to the ICRC-3 specific requirements, a fee collector configuration block:

- **MUST** contain a field `btype` with value `"107feecol"`.
- **MUST** contain a field `icrc107_fee_col` of type `Array`, specifying the owner and, optionally, a subaccount, following the ICRC-3 standard.
- To explicitly burn fees, `icrc107_fee_col` **MUST** be set to `variant { Array = vec {}}`.

#### Example Block

Below is an example of a valid **fee collector configuration block** that sets the fee collector to a specific account:

```
variant { Map = vec {
    // Block type: indicates this is a fee collector configuration block
    record { "btype"; variant { Text = "107feecol" }};
    // Fee collector account: specifies the account that will collect fees
    record { "icrc107_fee_col"; variant { Array = vec {
        variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"}; // Owner principal
        variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Subaccount
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

A block that explicitly sets fee burning by removing the fee collector (i.e., all fees are burned from this point onward):

```
variant { Map = vec {
    record { "btype"; variant { Text = "107feecol" }};
    record { "icrc107_fee_col"; variant { Array = vec {
            }}};
    record { "ts"; variant { Nat = 1_741_312_737_184_874_392 : nat } };
    record { "phash";
              variant {
                Blob = blob "\2d\86\7f\34\c7\2d\1e\2d\00\84\10\a4\00\b0\b6\4c\3e\02\96\c9\e8\55\6f\dd\72\68\e8\df\8d\8e\8a\ee"
              }};
}};
```



---

# ICRC-107: Methods for Setting and Getting Fee Collection

## 3. Methods for Setting and Getting Fee Collection

### 3.1 `icrc107_set_fee_collection`

This method allows a ledger controller to update the fee collection settings. It modifies the `icrc107_fee_col` account, which determines where collected fees are sent. The updated settings are recorded in a new block (of type `107feecol`) added to the ledger.

```
icrc107_set_fee_collection: (SetFeeCollectionRequest) -> ();
```

#### Request Type:
```
type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type SetFeeCollectionRequest = record {
    icrc107_fee_col: opt Account;
};
```

This method MUST only be callable by the ledger controller or authorized principals. Unauthorized calls SHOULD result in an error.

- If `icrc107_fee_col` is set to an account, all subsequent fees are collected by that account.
- If `icrc107_fee_col` is not provided (or set to `null`), all subsequent fees are burned.

The `icrc107_set_fee_collection` method MUST return an error in the following cases:

- The caller is not authorized to modify fee collection settings.
- The provided `Account` is invalid (e.g., the minting account on ledgers,  anonymous principal, malformed principal or subaccount)."


### 3.2 `icrc107_get_fee_collection`

This method retrieves the currently active fee collection settings. Unless changed, these settings apply to the next block added to the ledger.

```
icrc107_get_fee_collection: () -> (opt Account) query;
```


This method should return the currently active fee collection settings:

  - If the response is `null`, fees are burned. This corresponds to `icrc107_fee_col = variant { Array = vec {}}` on-chain.
  - If the response is a valid `Account`, fees are collected by that account. This corresponds to `icrc107_fee_col` being set to the ICRC-3 representation of the account on-chain.

This method strictly returns the last explicitly set value of `icrc107_fee_col`. It does not infer defaults, and if no fee collector was ever set, it returns `opt Account = null`.

---


## 4. Reporting Compliance

Compliance with the current standard is reported as follows.

### 4.1 Supported Standards

Ledgers implementing ICRC-107 **MUST** indicate their compliance through the `icrc1_supported_standards` and `icrc10_supported_standards` methods, by including in the output of these methods:

```
variant { Vec = vec {
    record {
        "name"; variant { Text = "ICRC-107" };
        "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107.md" };
    }
}};
```

### 4.2 Supported Block Types

Ledgers implementing ICRC-107 **MUST** report the new block type in response  to `icrc3_supported_block_types`.  The output of the call must include

```
variant { Vec = vec {
    record { "name"; variant { Text = "107feecol" }; "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107.md" } };
}};
```

## 5. Note on Legacy Fee Collection Mechanisms

The Dfinity maintained ICRC ledgers include a fee collection mechanism which, for completeness is described below.

### 5.1 Legacy Behavior (`fee_col`)


- Until `icrc107_fee_col` is set, the ledger follows this legacy behavior, using `fee_col` only for transfers.
- If `fee_col`is not set, all fees are burned.
- If `fee_col` is set in a block, the designated account collects only transfer fees (`1xfer`, `2xfer`). Fees for all other operations (e.g., `2approve`) were always burned in legacy behavior as implemented in Dfinity-maintained ICRC-3 ledgers.


New implementations SHOULD avoid using `fee_col` and instead use `icrc107_fee_col` for all fee collection settings. Legacy behavior is provided for backward compatibility only and MAY be deprecated in future versions of this standard.

### 5.2 Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

To determine who collects the fee in a block:

1. Check for fee collection configuration

   - If a previous block set `icrc107_fee_col`, the ledger uses the behavior specified by this standard.

2. If no `icrc107_fee_col` setting exists (legacy behavior):

   - If the block is of type `2approve` then the fee is burned
   - If the block is a transfer block, i.e. of type `1xfer` or `2xfer`:
      - If `fee_col` is specified in the block the fee is collected by `fee_col`.
      - If `fee_col_block` is specified use the `fee_col` from the referenced block index.

---
