| Status |
|:------:|
| Draft  |

# ICRC-107: Fee Collection

## 1. Introduction

Many ICRC-based ledgers (such as **ckBTC**) already implement partial fee collection, often charging fees for **transfer** transactions but not for **approve** transactions. However, there is currently no standardized way to:

1. Indicate and configure **who** should collect fees (or if fees are burned).
2. Record fee collection information on existing ledgers.
3. Provide clear semantics for wallets, explorers, and other integrations to **interpret** fee information.

**ICRC-107** aims to address this gap by defining:
- A set of **metadata entries** that indicate how fees are collected.
- A **minimal interface** for setting and enabling fee collection.
- **A backward-compatible standard** for **populating and understanding fee-collection fields** in ICRC-3 blocks corresponding to ICRC-1 and ICRC-2 transactions.
- Guidance on how to **interpret fee collection fields**, as recorded in blocks, in a consistent way.

### Non-Requirements
This standard does **not** address or require:
- Unsetting or removing a previously configured fee collector.
- Having multiple fee collectors (e.g., distinct collectors for transfers vs. approvals).
- Generalizing the mechanism for future block types beyond transfers/approvals.
- Prescribing detailed behavior for ledger upgrades that enable fee collection.

By focusing on **backward-compatible** enhancements and clear metadata, this standard allows ledgers to adopt fee collection with minimal disruption, while giving external tools a reliable way to parse and display fee-related data.

---

## 2. Metadata

A ledger implementing **ICRC-107** **MUST** include the following entry in the output of the `icrc1_supported_standards` method (indented for display):

    record {
      name = "ICRC-107";
      url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-107"
    }

Additionally, the ledger **MUST** provide (at minimum) these metadata entries, retrievable via the `icrc1_metadata` method:

1. **`icrc107_fee_collector` (text)**  
   The textual representation of the principal (or account) designated as the fee collector.

2. **`icrc107_fee_collection_transfers` (bool)**  
   Indicates whether fees are collected on **transfer** transactions (`true`) or not (`false`).

3. **`icrc107_fee_collection_approvals` (bool)**  
   Indicates whether fees are collected on **approve** transactions (`true`) or not (`false`).

If a ledger has not set a fee collector, it may return an empty string or omit the metadata key for `icrc107_fee_collector`. Similarly, if fees are never collected for transfers or approvals, the corresponding booleans should be `false` or omitted.

---

## 3. Fee Collection Management Interface

A ledger **implementing ICRC-107** **SHOULD** expose the following methods to programmatically configure and enable fee collection.


    // Sets the fee collector principal/account.
    // If not set, or set to an empty string, fees are effectively burned.
    icrc107_set_fee_collector: (text) -> ();

    // Enables fee collection for transfer transactions.
    // Returns the currently set fee collector if successful, or an error if none is set.
    icrc107_turn_on_fee_collector_transfers: () -> (variant {
        Ok : text;          // The fee collector principal
        Err : text;         // Error message
    });

    // Enables fee collection for approve transactions.
    // Returns the currently set fee collector if successful, or an error if none is set.
    icrc107_turn_on_fee_collector_approvals: () -> (variant {
        Ok : text;          // The fee collector principal
        Err : text;         // Error message
    });

### Method Semantics

- **`icrc107_set_fee_collector(fee_collector: text)`**  
  Updates the ledger’s metadata (`icrc107_fee_collector`) to the specified principal/account.  
  If this method is never called, fees are assumed to be burned by default.

- **`icrc107_turn_on_fee_collector_transfers()`**  
  Sets `icrc107_fee_collection_transfers` to `true`.  
  If a valid fee collector has been set, returns `Ok(fee_collector)`. Otherwise, returns `Err` describing why it cannot be enabled.

- **`icrc107_turn_on_fee_collector_approvals()`**  
  Sets `icrc107_fee_collection_approvals` to `true`.  
  If a valid fee collector has been set, returns `Ok(fee_collector)`. Otherwise, returns `Err`.

These endpoints allow ledger administrators (or canister controllers) to enable fee collection for different transaction types once the fee collector account is specified.

---

## 4. Fee Information in Blocks 

This section defines how **ICRC-107** determines whether a fee is collected or burned **exclusively** from information **contained in the blocks themselves**. No reliance on external metadata (e.g., `icrc107_fee_collector`) is assumed.

### 4.1 Determining the Fee

When reading or processing a block that may carry a fee, the ledger or external tools **SHOULD** apply the following logic to identify the final fee amount:

1. **Check for an Explicit Transaction Fee (`tx.fee`)**  
   - If the transaction itself contains a `fee` field (e.g., `tx.fee`), that value **SHOULD** take precedence as the fee to be applied.

2. **Fallback to `effective_fee`**  
   - If no `tx.fee` is provided, the ledger or tool **SHOULD** refer to the block’s `effective_fee` field (or an equivalent ledger-defined field) to determine the fee.

3. **No Explicit Fee**  
   - If neither `tx.fee` nor `effective_fee` is available, then the ledger MAY default the fee to zero or burn any implied fee as per its internal policy.  
   - An **ICRC-107**-compliant ledger interprets the absence of a specified fee as no fee collected, or a fee of zero.

### 4.2 Fee Collection Logic

After determining the fee amount, **ICRC-107** specifies how to decide **whether** a fee is burned or collected, and if collected, **who** receives it, **based solely on block contents**:

1. **Check for Fee Collector Fields**  
   - If the block contains a direct account field (e.g., `fee_col`) or a reference field (e.g., `fee_col_block`, `app_fee_col_block`), the fee is be collected by the referenced account.
   - If **neither** a direct account field nor a reference field is present, the fee is burned.

2. **Resolving a Reference to a Previous Block**  
   - If a reference field is used (`fee_col_block` or `app_fee_col_block`), the ledger or tool **MUST** locate that earlier block and extract the account specified there (e.g., `fee_col`).
   - If the referenced block does not contain a valid collector the fee is burned.


### 4.3 Recording Fee Collection in Blocks for ICRC-1 and ICRC-2 Transactions

The block schema for ICRC-1 and ICRC-2 transactions are specified in the ICRC-3 standard. This section explains how to extend those schemas to include backwards compatible fee collection information.

#### 4.3.1 Transfer Blocks

Transfer blocks in ICRC-based ledgers can be identified by one of the following:
- **`btype = "1xfer"`** (ICRC-1 style),
- **`btype = "2xfer"`** (ICRC-2 style),
- or a **transfer transaction** (`tx.op = "xfer"`).

To **record and interpret** fee-collection information in these blocks:

- **`fee_col` (Account)**  
  - A direct indicator of which account receives the collected fee.
- **`fee_col_block` (nat)**  
  - A pointer to an earlier block where `fee_col` is explicitly set.

**How to Determine Who Collects the Fee**  
1. If `fee_col` is present in the block, that account is the collector.  
2. If `fee_col` is absent but `fee_col_block` is present, fetch the referenced block; the `fee_col` in that block is the collector.  
3. If neither is present, **no** collector can be inferred, and the fee is burned.

#### 4.3.2 Approve Blocks

Approve blocks in ICRC-based ledgers can be identified by:
- **`btype = "2approve"`** (ICRC-2 style),
- or a **transaction object** (`tx.op = "approve"`).

To **record and interpret** fee-collection information in these blocks:

- **`fee_col` (Account)**  
  - A direct indicator of which account receives the collected fee for the approval operation.
- **`app_fee_col_block` (nat)**  
  - A pointer to an earlier block where `fee_col` is explicitly set for approves.

**How to Determine Who Collects the Fee**  
1. If `fee_col` is present in the block, that account is the collector.  
2. If `fee_col` is absent but `app_fee_col_block` is present, fetch the referenced block; the `fee_col` in that block is the collector.  
3. If neither is present, **no** collector can be inferred, and the fee is burned.

---

By strictly focusing on **block-level fields**:

- **Ledgers** remain free to implement or omit references in blocks while still following a consistent approach for collecting or burning fees.
- **External tools** (wallets, explorers) can determine the collector without requiring out-of-band metadata or additional method calls. If no collector info is embedded (directly or via a reference), the fee is considered burned.
- This specification thereby ensures **ICRC-1**, **ICRC-2**, and **ICRC-3** ledgers can remain consistent with **ICRC-107** using only the fields present in each block.
