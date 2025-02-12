# ICRC-107: Fee Collection

## 1. Introduction & Motivation

Fee collection in ICRC-based ledgers (e.g., ckBTC) is inconsistently implemented. While transfer transactions often incur fees, approve transactions typically do not. Currently, there is no standardized way to:

- Define fee collection rules—who receives fees, which operations are charged, and whether fees are burned.
- Record fee collection settings directly on-chain in ledger blocks.
- Provide consistent semantics for wallets, explorers, and other integrations to interpret fee structures.

ICRC-107 extends [ICRC-3](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-3.md), adding semantics for fee collection while ensuring full compatibility with the existing block format. 

### ICRC-107 Proposal


ICRC-107 introduces a standardized mechanism for on-chain fee collection, ensuring clarity and interoperability across different ICRC-based ledgers. It defines:

- A fee collection configuration specifying the collector account (`fee_col`) and the operations (`col_ops`) subject to fees.
- A backward-compatible extension to ICRC-3 blocks, allowing fee collection settings to be recorded and modified within ledger history.
- Clear rules governing fee distribution: If `fee_col` is set, fees are transferred to the collector for the operations specified via `col_ops`; otherwise, they are burned.

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

### 4.1 `set_fee_collection`

Updates fee collection settings. Only callable by the ledger controller.

```
set_fee_collection: (opt Map<Value>, opt Vec<Text>) -> ();
```

### 4.2 `get_fee_collection`

Returns the current fee collection settings and the block index of the last update.

```
get_fee_collection: () -> (opt Map<Value>, Vec<Text>, Nat) query;
```

---

## 5. Interaction with Future Standards

Any new standard that introduces additional block types **MUST** specify how those block types interact with the fee collection mechanism defined in **ICRC-107**. Specifically:

1. **Fee Payer Determination**  
   - Define how the fee amount is determined and who is responsible for paying the fee.

2. **Fee Applicability**  
   - Clarify which new block types, if any, are subject to fee collection.

3. **Fee Collection Rules**  
   - Define entries for `fee_col` corresponding to the new block types.

---





## 6. Summary

- **On-Chain Configuration**: Fee collection settings (`fee_col`, `col_ops`) are recorded and modified within ledger blocks.
- **Fee Collection & Burning**: If `fee_col` is set and a block type  appears in `fee_ops` then fees for that block type are collected by `fee_col`. Otherwise, the fee is burned.
- **Governance**: Only the **ledger controller** can update fee collection settings.
- **Backward-Compatible**: Works seamlessly with ICRC-1, ICRC-2, and ICRC-3 block definitions.


---
