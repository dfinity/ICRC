# ICRC-107: Fee Collection

## 1. Introduction & Motivation

The lack of a unified approach has led to inconsistencies in fee collection across ICRC-based ledgers (e.g., ckBTC). For instance, in the ckBTC ledger, fees for transfer operations are collected by a designated fee collector, whereas fees for approve operations are burned—even when some fee collection details are present in approve blocks. However, there is no standardized way to:


- Define fee collection rules—who receives fees, which operations are charged, and whether fees are burned.
- Record fee collection settings directly on-chain in ledger blocks.
- Provide consistent semantics for wallets, explorers, and other integrations to interpret fee structures.

ICRC-107 extends **ICRC-3**, adding semantics for fee collection while ensuring full compatibility with the existing block format. This proposal eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

### ICRC-107 Proposal

ICRC-107 introduces a standardized mechanism for on-chain fee collection, ensuring clarity and interoperability across different ICRC-based ledgers. It defines:

- A fee collection configuration specifying the collector account (`fee_col`) subject to fees.
- A backward-compatible extension to ICRC-3 blocks, allowing fee collection settings to be recorded and modified within ledger history.
- Clear rules governing fee distribution: If `fee_col` is set, fees are transferred from the paying account to the collector; otherwise, they are burned — meaning they are debited from the paying account and removed from the total supply.

By embedding fee collection settings entirely on-chain, ICRC-107 eliminates reliance on off-chain metadata, simplifies integration with wallets and explorers, and ensures full transparency in fee handling.

---

## 2. Fee Collection

### 2.1 Overview

For each block added to the ledger, some party may have to pay a fee. The amount and payer of the fee depend on the specific operation type.

Fee collection settings determine how fees are processed. These settings consist of:

- A fee collector account (`fee_col`). If `fee_col` has never been set in any block, fees are burned by default. Burning occurs by removing the fee from the paying account and reducing the total token supply by that amount.
- A list of operations (`fee_ops`) for which fees are collected. If an operation is not in `fee_ops`, the fee is burned.

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


- **`fee_ops`** (optional, `Vec<Text>`)
  - A list of operation types for which fees **should** be collected.
  - If omitted, defaults to `["1xfer","2xfer"]` (fees are collected only for transfers).
  - If `fee_ops` is explicitly set to an empty list (`[]`), no fees are collected, and all fees are burned.


**Note:** The block format follows the [ICRC-3 standard](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-3.md).

---

## 3. Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

### 3.1 Determining the Fee for ICRC-1 and ICRC-2 Blocks

Blocks follow the ICRC-3 standard. To determine the **final fee amount** for a block:

1. **Check `tx.fee`**  
   - If present, `tx.fee` is the fee for this block.

2. **Else, check the top-level `fee` field**  
   - If `tx.fee` is **not set**, then a top-level field `fee` exists in the block, and that is the fee.


The fee payer is always the source account in both transfer and approve transactions.


## **3.2 Determining the Active Fee Settings for a Block**

To determine the **active fee settings** (who receives the fee and whether it is collected or burned), follow this algorithm:

1. **Find the most recent block (including the current block) that defines `fee_col`**  
   - If `fee_col = []`, all fees in the block are **burned**, and `fee_ops` is ignored.  
   - Otherwise, use this value as the **active fee collector** (`active_fee_col`).  
   - If no `fee_col` is found in any prior block, all fees **are burned by default**.

2. **Determine `fee_ops` from the same block where `fee_col` was set**  
   - If `fee_ops` is set in that block, use it as the **active fee_ops**.  
   - If `fee_ops` is **not set** in that block, **default to `["1xfer", "2xfer"]`**, meaning only transfer operations collect fees.  

3. **Determine whether the fee should be collected or burned**  
   - Identify the **block type**:  
     - If `btype` is present, use it.  
     - Otherwise, infer from `tx.op`.  
   - If the **block type is in `fee_ops`**, fees are **collected** by `active_fee_col`.  
   - Otherwise, fees are **burned**.  



   ## **3.3 Mapping `fee_ops` to Block Types**

   The `fee_ops` field specifies which block types **have their fees collected** instead of burned. For ICRC-1 and ICRC-2 blocks, `fee_ops` maps to block types as follows:

   | **fee_ops Entry** | **Block Types Affected** |
   |-------------------|--------------------------|
   | `"1xfer"`  | Blocks with `btype = "1xfer"` or where `btype` is not set, `tx.op = "xfer"`, and `tx.spender` is **not** set. |
   | `"2xfer"`  | Blocks with `btype = "2xfer"` or where `btype` is not set, `tx.op = "xfer"`, and `tx.spender` **is** set. |
   | `"2approve"` | Blocks with `btype = "2approve"` or where `btype` is not set and `tx.op = "approve"`. |


#### Key Rules:

1. **Determine Block Type**  
   - If `btype` is present, it takes precedence.
   - Otherwise, use `tx.op` to determine the block type.

2. **Check if `fee_ops` applies to the block type**  
   - If the block type corresponds to an entry in `fee_ops`, the fee is collected by `fee_col`.
   - Otherwise, the fee is burned.



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
The fee collections settings will be recorded in the first block that is created after calling this endpoint.  


### 4.2 `icrc107_get_fee_collection`

This method retrieves the currently active fee collection settings. Unless they are changed they would apply to the next block added to the ledger.

```
icrc107_get_fee_collection: () -> (opt Account, Vec<Text>) query;
```

`icrc107_get_fee_collection` returns two values:

* `opt Account` (`fee_col`) – The active fee collector account.
    - If `null`, fees are burned (this corresponds to `fee_col=[]` on-chain).
    - Otherwise, fees are collected by the returned `Account`

* `Vec<Text>` (`fee_ops`) – The list of operation types for which fees are collected.
    - If `fee_ops=[]`, no fees are collected, and all fees are burned. This is equivalent to `fee_col=[]`, meaning that all transactions incur a fee that is removed from the supply.
    - The API **does not apply a default** — it always returns the explicitly set value. The defaulting behavior (`["1xfer", "2xfer"]`) applies only at the block processing level when `fee_ops` is missing from a block.

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
