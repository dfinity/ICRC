# ICRC-107: Fee Collection

## 1. Introduction & Motivation

The absence of a unified approach has resulted in inconsistencies in fee collection across ICRC-based ledgers (e.g., ckBTC). In the ckBTC ledger, the fee collector collects fees for transfer operations, while fees for approve operations are burned, demonstrating the need for a unified standard. However, there is no standardized way to:

- Define fee collection rules—who receives fees, which operations are charged, and whether fees are burned.
- Record fee collection settings directly on-chain in ledger blocks.
- Provide consistent semantics for wallets, explorers, and other integrations to interpret fee structures.

ICRC-107 extends **ICRC-3**, adding semantics for fee collection while ensuring full compatibility with the existing block format. This proposal eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

### ICRC-107 Proposal

ICRC-107 introduces a standardized mechanism for on-chain fee collection, ensuring clarity and interoperability across different ICRC-based ledgers. It defines:

- A fee collection configuration specifying the collector account (`fee_col`) subject to fees.
- A backward-compatible extension to ICRC-3 blocks, allowing fee collection settings to be recorded and modified within ledger history.
- Clear rules governing fee distribution: If `fee_col` is set, fees are transferred from the paying account to the collector; otherwise, they are burned, —meaning they are debited from the paying account and removed from the total supply.

By embedding fee collection settings entirely on-chain, ICRC-107 eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

---

## 2. Fee Collection

### 2.1 Overview

For each block added to the ledger, some party may have to pay a fee. The amount and payer of the fee depend on the specific operation type.

Fee collection settings determine how fees are processed. These settings include:


- A fee collector account (`fee_col`). If `fee_col` is not set, all fees are burned.  Burning occurs by removing the fee from the paying account and reducing the total token supply by that amount.
- A list of operations (`fee_ops`) for which fees are collected. If an operation is not in `fee_ops`, the fee is burned.
- A flag `fee_burn` explicitly determines that all fees are burned. If set to "true", fees are always burned, regardless of `fee_col`.   

Changes to fee collection settings are recorded on-chain and take effect immediately in the block where they appear.

### 2.2 Fee Collection Settings

**Backwards Compatibility:**
To ensure compatibility with existing ICRC-3 implementations, `fee_col` **MUST** be recorded using the existing array format:

```
fee_col;
variant {
  Array = vec {
    variant { Blob = principal_blob };
    variant { Blob = subaccount_blob };
  }
}
```

- The **first Blob** represents the **principal** of the fee collector; if this is the anonymous principal (0x04) then fees for all operations are burned.
- The **second Blob** represents the **subaccount**, or a zeroed-out Blob if no subaccount is used.

This ensures that existing tools and explorers relying on the current `fee_col` format remain functional.

A block **MAY** include the following fields to define or update fee collection settings:

- **`fee_col`** (optional, `Map<Value>`)
  - `principal`: `Blob` — The principal of the fee collector.
  - `subaccount`: `Blob` (optional) — The subaccount identifier, if applicable.

- **`fee_burn`** (optional, `Text`)
  - If `"true"`, fees are burned (removed from supply), irrespective of `fee_col` settings.
  - If `"false"`, fees are transferred to `fee_col`.
  - Defaults to `"true"` if `fee_col` is unset.

- **`fee_ops`** (optional, `Vec<Text>`)

  - A list of operation types for which fees **should** be collected.
  - If omitted, defaults to `["1xfer","2xfer"]` (fees are collected only for transfers).

Blocks where `fee_ops` is specified, or `fee_burn` is set to "false" MUST also specify `fee_col`.  

**Note:** The block format follows the [ICRC-3 standard](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-3.md).

---

## 3. Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

### 3.1 How to Calculate the Fee for ICRC-1 and ICRC-2 Blocks

Blocks follow the ICRC-3 standard. To determine the **final fee amount** for a block:

1. **Check `tx.fee`**  
   - If present, `tx.fee` is the fee for this block.

2. **Else, check the top-level `fee` field**  
   - If `tx.fee` is **not set**, then a top-level field `fee` exists in the block, and that is the fee.


The paying account is the source account for both transfer and approve transactions.

### 3.2 How to Determine the Fee Collector for a Block

Find the most recent block where fee collection settings are present.  
* If no such block exists, the fees in the current block are burned.
* If the most recent block with fee settings has `fee_burn` = "true", then fees in the current block are burned.
* Otherwise, the active fee collector is the account specified by `fee_col`, and fees are collected for the operations listed in `fee_ops` (see the next section).


### 3.3 How `fee_ops` Determines Fee Collection for a Block

The `fee_ops` field defines which block types incur fees that are collected instead of burned. For ICRC-1 and ICRC-2 blocks, `fee_ops` entries map to block types as follows:

| **fee_ops Entry** | **Block Types Affected** |
|-------------------|--------------------------|
| "1xfer"    | Blocks with `btype = "1xfer"` and blocks where `btype` is not set and `tx.op = "xfer"` and there is no `tx.spender` field |
| "2xfer"    | Blocks with `btype = "2xfer"` and blocks where `btype` is not set and `tx.op = "xfer"` and the field `tx.spender` is set. |
| "2approve"     | Blocks with `btype = "2approve"` and blocks where `btype` is not set and `tx.op = "approve"`. |

#### Key Rules:

1. **Determine Block Type**  
   - If `btype` is present, it takes precedence.
   - Otherwise, use `tx.op` to determine the block type.

2. **Check if `fee_ops` applies to the block type**  
   - If the block type corresponds to an entry in `fee_ops`, the fee is collected by `fee_col`.
   - Otherwise, the fee is burned.

If `fee_col` is set but `fee_ops` is not, `fee_ops` defaults to `["1xfer, 2xfer"]`, i.e. fees are only collected for transfer operations.


---

## 4. Minimal API for Fee Collection Settings

### 4.1 `icrc107_set_fee_collection`

This method allows a ledger controller to update the fee collection settings. It modifies the `fee_col` account, which determines where collected fees are sent, and updates the `fee_ops` list, which specifies the transaction types for which fees should be collected. The updated settings are recorded in the next block added to the ledger.

```
icrc107_set_fee_collection: (SetFeeCollectionRequest) -> ();
```

with

```
type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type SetFeeCollectionRequest = record {
     fee_col: opt Account;
     fee_ops: Vec<Text>
}
```

If `fee_col` is not provided then fee collection reverts to fee burning.


### 4.2 `icrc107_get_fee_collection`

This method retrieves the currently active fee collection settings. Unless they are changed they would apply to the next block added to the ledger.

```
icrc107_get_fee_collection: () -> (opt Account, Vec<Text>) query;
```

`icrc107_get_fee_collection` returns two values:

* `opt Account` (`fee_col`) – The active fee collector account. If `null`, fees are burned (`fee_burn` = "true").
* `Vec<Text>` (`fee_ops`) – The list of operation types for which fees are collected. If `fee_col` is set but `fee_ops` is empty, no fees are collected and all fees are burned.

---

## 5. Interaction with Future Standards

Any new standard that introduces additional block types **MUST** specify how those block types interact with the fee collection mechanism defined in **ICRC-107**. Specifically:

1. **Fee Applicability**  
   - Clarify which new block types, if any, are subject to fee collection.

2. **Fee Payer Determination**  
   - Define how the fee amount is determined and who is responsible for paying the fee.

3. **Fee Collection Rules**  
   - Define new entry types for `fee_col` corresponding to the new block types.
   - Specify how `fee_ops` should be applied to determine whether fees for the new block types are collected or burned.

By ensuring that any future standards explicitly define their interaction with fee collection, ICRC-107 remains a robust and extensible framework.


---
## 6. Reporting Compliance with ICRC-Supported Standards

Ledgers implementing ICRC-107 **MUST** indicate their compliance through the `icrc1_supported_standards` and `icrc10_supported_standards` methods, as defined in ICRC-1 and ICRC-10 by including the following record in the output of these methods:

```
record {
    name = "ICRC-107";
    url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107.md"
}
```

---

## 7. Summary

ICRC-107 provides a self-contained, transparent, and interoperable framework for managing transaction fees across ICRC-based ledgers.
