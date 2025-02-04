
# ICRC-107: Fee Collection (Purely Block-Based)

## 1. Introduction & Motivation

Many ICRC-based ledgers (such as **ckBTC**) already implement partial fee collection, often charging fees for **transfer** transactions but not for **approve** transactions. However, there is currently no standardized way to:

1. Indicate and configure **who** should collect fees (or if fees are burned).
2. Record fee collection information directly in ledger blocks.
3. Provide clear semantics for wallets, explorers, and other integrations to **interpret** fee information in a consistent way.

**ICRC-107** aims to address this gap by defining:
- A mechanism to specify **fee collection details** (collector account and applicable operations) directly in blocks.
- A **backward-compatible standard** for **recording and interpreting fee-collection fields** in ICRC-3 blocks corresponding to ICRC-1 and ICRC-2 transactions.
- Clear rules on how fee collection information evolves over time and how subsequent blocks rely on prior updates.

This design ensures that **all** fee-related information is stored entirely **on-chain** in the block history, without reliance on external metadata or specialized block types. It provides a fully self-contained, transparent standard for fee collection that simplifies integration with wallets, explorers, and other tools.

---

## 2. Fee Collection Mechanism

### 2.1 Fields in Blocks

A block **MAY** include the following fields to define or update fee collection settings:

- **`fee_col`**: An **account** (ICRC account format) to which fees are paid.  
  **Format**:
  ```candid
  record {
      principal: principal;
      subaccount: opt blob;
  }
  ```
- **`fee_ops`**: An **array of block types** (strings) for which fees **should** be collected. This standard specifies two types of blocks for which fee collection can occur: "transfer" and "approve".  

When a block includes either or both fields, it **overrides** the previously known settings from that point onward.

#### 2.1.1 Defaults
- If `fee_ops` is **omitted**, it defaults to `["transfer"]`.  
- If `fee_col` is **omitted**, the previously known collector remains in effect (or none if it was never set).

---

## 3. Fee Computation

### 3.1 Determining the Fee Amount

The fee amount in any block can be derived via:
- An explicit `tx.fee` field in the block’s transaction.
- A fallback `effective_fee` or other ledger-defined field if `tx.fee` is absent.
- Zero if neither is present.

### 3.2 Collected or Burned

After determining the fee amount, check the **current** configuration (from the most recent update):
1. **Operation Present**: If the block’s operation (e.g., `"transfer"`, `"approve"`, etc.) is in the active list (`fee_ops`), the fee is collected by the active `fee_col`.
2. **Operation Absent**: If the operation is **not** in the active list, the fee is burned.
3. **No Collector**: If no collector has ever been set, fees are burned by default.

---

## 4. Determining the Current Fee Configuration

At any block height `h`, to figure out **who** collects fees and for **which** operations:
1. **Traverse blocks** backward (or maintain an in-memory state) until you find the most recent block `< h` that included `fee_col` or `fee_ops`.
2. The fee-collector account from that block is **active**, and the operation list from that block is **active** (defaulting to `["transfer"]` if omitted).
3. If no such block is found, no collector is active and **all fees are burned**.

---

## 5. Examples

### 5.1 Setting a Collector (Defaults to Transfers Only)

```json
{
  "block_index": 100,
  "timestamp": 1690000000,
  "kind": "transaction",
  "fee_col": {
    "principal": "aaaaa-aa",
    "subaccount": null
  },
  "transaction": {
    "operation": "transfer",
    "from": {
      "principal": "bbbbb-bb",
      "subaccount": null
    },
    "to": {
      "principal": "ccccc-cc",
      "subaccount": null
    },
    "amount": 5000,
    "fee": 10
  }
}
```

- Future blocks that perform a **transfer** pay fees to `aaaaa-aa`.
- **Approve** or other operations are not in the default set, so their fees are burned.

### 5.2 Block with Approve but No Update

```json
{
  "block_index": 101,
  "timestamp": 1690000500,
  "kind": "transaction",
  "transaction": {
    "operation": "approve",
    "from": {
      "principal": "bbbbb-bb",
      "subaccount": null
    },
    "spender": {
      "principal": "ccccc-cc",
      "subaccount": null
    },
    "amount": 3000
  }
}
```

- No new fee settings. The **previous** block #100 had `["transfer"]` as the fee-bearing operation.
- This is an `"approve"` operation, **not** in `["transfer"]`, so the fee (if any) is burned.

### 5.3 Updating Collector and Adding Approves

```json
{
  "block_index": 150,
  "timestamp": 1690001000,
  "kind": "transaction",
  "fee_col": {
    "principal": "zzzzz-zz",
    "subaccount": null
  },
  "fee_ops": ["transfer", "approve"],
  "transaction": {
    "operation": "approve",
    "from": {
      "principal": "bbbbb-bb",
      "subaccount": null
    },
    "spender": {
      "principal": "ccccc-cc",
      "subaccount": null
    },
    "amount": 6000,
    "fee": 5
  }
}
```

- At block #150, the collector is changed to `zzzzz-zz`, **and** we explicitly list `["transfer", "approve"]`.
- From block #150 onward, **both** transfers and approves incur fees, credited to `zzzzz-zz`.
- The **approve** operation in this same block has a fee of 5 tokens.

---

## 6. API for Managing Fee Collection

### 6.1 Methods

#### 6.1.1 `set_fee_collection`

Sets or updates the fee collector account and/or the set of fee-bearing operations.

**Candid Definition:**
```candid
set_fee_collection: (opt Account, opt vec text) -> ();
```

#### 6.1.2 `get_fee_collection`

Retrieves the current fee collector account and the list of fee-bearing operations.

**Candid Definition:**
```candid
get_fee_collection: () -> (opt Account, vec text) query;
```

---

## 7. Interaction with New Standards

Any new standard that introduces **additional block types** **MUST** specify how those block types interact with the fee collection mechanism defined in **ICRC-107**.

---

## 8. Summary

- **Purely On-Chain**: Fee collection settings are managed entirely within the block history.
- **Flexible Defaults**: Default fee-bearing operations are `["transfer"]` if not explicitly specified.
- **Transparent Evolution**: The chain’s history defines how fee collection evolves over time, ensuring full traceability.
