# ICRC-107: Fee Collection

## 1. Introduction & Motivation

Fee collection in ICRC-based ledgers (e.g., ckBTC) is inconsistently implemented. While transfer transactions often incur fees, approve transactions typically do not. However, there is no standardized way to:

- Define fee collection rules—who receives fees, which operations are charged, and whether fees are burned.
- Record fee collection settings directly on-chain in ledger blocks.
- Provide consistent semantics for wallets, explorers, and other integrations to interpret fee structures.

ICRC-107 extends **ICRC-3**, adding semantics for fee collection while ensuring full compatibility with the existing block format. This proposal eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

### ICRC-107 Proposal

ICRC-107 introduces a standardized mechanism for on-chain fee collection, ensuring clarity and interoperability across different ICRC-based ledgers. It defines:

- A fee collection configuration specifying the collector account (`fee_col`) subject to fees.
- A backward-compatible extension to ICRC-3 blocks, allowing fee collection settings to be recorded and modified within ledger history.
- Clear rules governing fee distribution: If `fee_col` is set, fees are transferred to the collector; otherwise, they are burned.

By embedding fee collection settings entirely on-chain, ICRC-107 eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

---

## 2. Fee Collection

### 2.1 Overview

For each block added to the ledger, some party must pay a fee. The amount and payer of the fee depend on the specific transaction type.

Fee collection settings determine how fees are processed. These settings include:

- A fee collector account (`fee_col`). If `fee_col` is not set, all fees are burned.
- A list of operations (`col_ops`) for which fees are collected. If an operation is not in `col_ops`, the fee is burned.

Changes to fee collection settings are recorded on-chain, ensuring a transparent history of modifications.

### 2.2 Fee Collection Settings

**Backwards Compatibility:**
To ensure compatibility with existing ICRC-3 implementations, `fee_col` **MUST** continue to be recorded using the existing array format:

```
fee_col;
variant {
  Array = vec {
    variant { Blob = principal_blob };
    variant { Blob = subaccount_blob };
  }
}
```

- The **first Blob** represents the **principal** of the fee collector.
- The **second Blob** represents the **subaccount**, or a zeroed-out Blob if no subaccount is used.

This ensures that existing tools and explorers relying on the current `fee_col` format remain functional.

A block **MAY** include the following fields to define or update fee collection settings:

- **`fee_col`** (optional, `Map<Value>`)

  - `principal`: `Blob` — The principal of the fee collector.
  - `subaccount`: `Blob` (optional) — The subaccount identifier, if applicable.

- **`col_ops`** (optional, `Vec<Text>`)

  - A list of operation types for which fees **should** be collected.
  - If omitted, defaults to `"transfer"` (fees are collected only for transfers).

- **`prev_fee_col_info`** (optional, `Nat`)

  - The block index at which `fee_col` or `col_ops` was last updated.
  - Enables quick lookup of previous fee collection settings.

**Note:** The block format follows the [ICRC-3 standard](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-3.md).

---

## 3. Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

### 3.1 How to Calculate the Fee for ICRC-1 and ICRC-2 Blocks

Blocks follow the ICRC-3 standard. To determine the **final fee amount** for a block:

1. **Check `tx.fee`**  
   - If present, `tx.fee` is the fee for this block.

2. **Else, check the top-level `fee` field**  
   - If `tx.fee` is **not set**, and a top-level field `fee` exists in the block, then that is the fee.

3. **Else, fallback to `0`**  
   - If **neither** `tx.fee` nor `fee` is present, the default fee is `0`.

The paying account is the source account for both transfer and approve transactions.

### 3.2 How `col_ops` Determines Fee Collection

The `col_ops` field defines which block types incur fees that are collected instead of burned. For ICRC-1 and ICRC-2 blocks, `col_ops` entries map to block types as follows:

| **col_ops Entry** | **Block Types Affected** |
|-------------------|--------------------------|
| **"transfer"**    | Blocks with `btype = "1xfer"` or `btype = "2xfer"`. If `btype` is not set, use `tx.op = "xfer"` or `"2xfer"`. |
| **"approve"**     | Blocks with `btype = "2approve"`. If `btype` is not set, use `tx.op = "approve"`. |

#### Key Rules:

1. **Determine Block Type**  
   - If `btype` is present, it takes precedence.
   - Otherwise, use `tx.op`.

2. **Check if `col_ops` applies to the block type**  
   - If the block type corresponds to an entry in `col_ops`, the fee is collected by `fee_col`.
   - Otherwise, the fee is burned.

If no `fee_col` is set, all fees are burned by default.

---

## 4. Minimal API for Fee Collection Settings

### 4.1 `icrc107_set_fee_collection`

This method allows the ledger controller to update the fee collection settings. It modifies the fee_col account, which determines where collected fees are sent, and updates the col_ops list, which specifies the transaction types for which fees should be collected. The updated settings are recorded in the next block added to the ledger.

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
     fee_col: Account;
     fee_ops: Vec<Text>
}
```


### 4.2 `icrc107_get_fee_collection`

This method retrieves the currently active fee collection settings, including the fee_col account (if set), the list of transaction types (col_ops) for which fees are collected, and the block index of the last recorded update. This allows external systems, such as wallets and explorers, to determine how fee collection is configured.
```
icrc107_get_fee_collection: () -> (opt Account, Vec<Text>, Nat) query;
```

**Note:** The block returned reflects the last block where a change in fee collection information was recorded. However, this block may not contain the most recent settings (as returned in the first part of the response) if no block was created between setting fee collection information and retrieving it. The latest fee collection settings will be recorded in the first block that will be added to the ledger.

---

## 5. Interaction with Future Standards

Any new standard that introduces additional block types **MUST** specify how those block types interact with the fee collection mechanism defined in **ICRC-107**. Specifically:

1. **Fee Applicability**  
   - Clarify which new block types, if any, are subject to fee collection.

2. **Fee Payer Determination**  
   - Define how the fee amount is determined and who is responsible for paying the fee.

3. **Fee Collection Rules**  
   - Define new entry types for `fee_col` corresponding to the new block types.
   - Specify how `fee_col` should be applied to determine whether fees for the new block types are collected or burned.

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
