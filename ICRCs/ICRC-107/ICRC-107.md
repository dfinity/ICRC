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

ICRC-107 introduces a global on-chain fee collection setting recorded in the ledger via a new block type `btype = "107feecol"`. This setting determines whether the effective fee of fee-charging blocks is credited to a designated account or burned. All changes are recorded on-chain for transparent, interoperable processing by wallets and explorers.


Specifically, ICRC-107 defines:

- A fee collection configuration that specifies the collector account (`fee_collector`), applying to all subsequent blocks.  
- A new block type (`btype = "107feecol"`) for setting `fee_collector` to designate the fee collector.  
- A backward-compatible mechanism where, until the new mechanism is employed, the legacy collection logic applies.  

By embedding fee collection settings entirely on-chain, ICRC-107 ensures **transparency**, **interoperability**, and **simplified integration** for wallets, explorers, and other tools.  

### Effective Fee Application 

For any block that charges a fee, handle the fee using the following  checklist, **in order**:

1. **Find the current `107feecol` setting for this block**

   Look **backwards** from this block’s index to the most recent block with `btype = "107feecol"`.

   - If you find **none**, then **legacy collection logic applies** (see *Legacy Fee Collection Mechanism*).
   - If you find one, inspect its `tx`:
     - If `tx` contains a `fee_collector` field, the current setting is **that Account**.
     - If `tx` does **not** contain a `fee_collector` field, the current setting is **“burn all fees”**.

2. **Apply the setting**

   - **Account found (`fee_collector` present):**  
     Credit the **effective fee** to that account.
   - **Explicit burn (`fee_collector` absent in the latest `107feecol`):**  
     Burn the effective fee.
   - **No `107feecol` block exists yet:**  
     Apply **legacy `fee_col` logic (see Legacy Fee Collection Mechanism)**.  
     In brief, for DFINITY-maintained ledgers: if `fee_col` is unset, burn all fees; if `fee_col` is set, only transfer fees are credited to that account, and all other fees are burned.

> **Important:**  
> - “No block with `btype = "107feecol"` is found” is **not** the same as having a `107feecol` block whose `tx` omits `fee_collector`.  
> - **No `107feecol`** → legacy rules.  
> - **Latest `107feecol` present, but `fee_collector` field missing in `tx`** → “fees are explicitly burned from now on.”

3. **Non-retroactivity**

   A `107feecol` block affects only blocks at its index **and after**.  
   Earlier blocks keep whatever applied at their own indices (legacy or earlier `107feecol` settings).


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

| Field   | Type (ICRC-3 `Value`) | Required | Description                                                                 |
|--------|------------------------|----------|-----------------------------------------------------------------------------|
| `btype`| `Text`                 | Yes      | **MUST** be `"107feecol"`.                                                 |
| `ts`   | `Nat`                  | Yes      | Timestamp (in nanoseconds since the Unix epoch) when the block was added.  |
| `phash`| `Blob`                 | Yes      | Hash of the parent block.                                                  |
| `tx`   | `Map(Text, Value)`     | Yes      | Encodes information about the fee collection configuration transaction.    |

### Minimal `tx` Schema

The minimal fields required to interpret a `107feecol` block:

| Field           | Type (ICRC-3 `Value`) | Required | Description |
|----------------|------------------------|----------|-------------|
| `fee_collector`| Account                | No       | If present, designates the fee collector account. If **absent**, this `107feecol` block explicitly sets all fees to be **burned** from its index onward. |

(Other fields such as `op`, `created_at_time`, and `caller` are defined in the **Canonical `tx` Mapping**.)

## Semantics of `107feecol` Blocks

### Core State Transition

A `107feecol` block updates the ledger’s global fee collector configuration:

- If `tx` contains a `fee_collector` field encoding an account → all subsequent fees are **credited to that account**.
- If `tx` does **not** contain a `fee_collector` field → all subsequent fees are **burned**.

This configuration applies **to all subsequent blocks** until replaced by another `107feecol`.

> **Note:**  
> The presence or absence of the `fee_collector` field in the **latest** `107feecol` block is what determines the behavior.  
> The absence of any `107feecol` blocks means “use legacy behavior”.

---

### Fee Application under ICRC-107

For every block that charges a fee:

1. Execute the **pre-fee state transition** (per ICRC-3 evaluation model).  
2. Compute the **effective fee** (as defined in ICRC-3).  
3. Debit the **effective fee** from the block’s **fee payer**.  
4. Determine how the effective fee is handled:

   - **If at least one `107feecol` block exists:**  
     • If the most recent `107feecol` has `tx.fee_collector = []` → burn the effective fee.  
     • If the most recent `107feecol` has `tx.fee_collector = <Account>` → credit the effective fee to that account.  

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

This method allows a ledger controller to update the fee collector. It modifies the `fee_collector` account, which determines where collected fees are sent. The updated settings are recorded in a new block (of type `107feecol`) added to the ledger.

```

type Account = record {
    owner: principal;
    subaccount: opt blob;
};

type SetFeeCollectorArgs = record {
    fee_collector: opt Account;
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

### `icrc107_set_fee_collector` — Semantics

- **Authorization**  
  The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals. Unauthorized calls **MUST** fail with `AccessDenied`.

- **Effect (on success, non-retroactive)**  
  The ledger **MUST** append a new block with `btype = "107feecol"`.  
  **The block’s `tx` is a top-level field of that `107feecol` block** and **MUST** be constructed **exactly** as defined in **Canonical `tx` Mapping** (same keys, types, and value encodings) and embedded in the block.

  The `fee_collector` argument determines how `tx` is constructed and how fees are handled from the new block’s index onward:

  - If `fee_collector` is **provided** (`?Account`):  
    - `tx` **MUST** contain a `fee_collector` field whose value encodes that Account using the ICRC-3 Account representation.  
    - All effective fees for this and later blocks **MUST** be **credited** to that account, until changed by a later `107feecol`.

  - If `fee_collector` is **absent** (`null`):  
    - The `tx` map **MUST NOT** contain a `fee_collector` field.  
    - All effective fees for this and later blocks **MUST** be **burned**, until changed by a later `107feecol`.

  The new configuration **MUST** apply starting **at** the returned block index (non-retroactive). Earlier blocks **MUST NOT** be affected.

- **Return value**  
  On success, the method **MUST** return `Ok(index : nat)`, where `index` is the block index of the newly appended `107feecol` block.

- **Multiple updates over time**  
  If multiple `107feecol` blocks exist, the setting that applies to any block **MUST** be the **last** `107feecol` **at or before** that block’s index.

- **Deduplication & idempotency**  
  The ledger **MUST** perform deduplication (e.g., using `created_at_time` and any implementation-defined inputs).  
  If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

- **Error cases (normative)**  
  The method **MUST** return an error in these cases:
  - **AccessDenied** — caller is not authorized to modify the fee collector.  
  - **InvalidAccount** — provided collector account is invalid (e.g., minting account, anonymous principal, malformed principal/subaccount).  
  - **Duplicate** — a semantically identical transaction was already accepted (per the ledger’s deduplication rules); response **MUST** include `duplicate_of` with the existing block index.  
  - **GenericError** — any other failure that prevents constructing or appending a valid `107feecol` block.

- **Clarifications**  
  A `107feecol` block whose `tx` omits `fee_collector` (explicit **burn**) is **not** the same as “no `107feecol` has ever been set” (which implies **legacy behavior** applies until the first ICRC-107 setting appears).

---
### Canonical `tx` Mapping (normative)

The `tx` field of every `107feecol` block **MUST** be constructed exactly according to the following canonical mapping.  
All fields marked “Always included” MUST appear in the `tx` map of the block.  
The presence of `fee_collector` is the only conditional component.

| Field             | Type (ICRC-3 `Value`) | Canonical Encoding Rule |
|-------------------|------------------------|--------------------------|
| `op`              | `Text`                 | Always included. **MUST** be `"107set_fee_collector"`. |
| `fee_collector`   | Account                | **Included iff** the `fee_collector` argument is `?Account`.  
|                   |                        | **If the argument is `null`, the `fee_collector` field MUST be omitted** from the `tx` map. |
| `created_at_time` | `Nat`                  | Always included. Nanoseconds since Unix epoch. **MUST** fit in `nat64`. |
| `caller`          | `Blob`                 | Always included. Principal bytes of the caller. |

**Notes:**

- The **absence** of the `fee_collector` field in the `tx` map means:  
  **“burn all fees from this block onward.”**


#### System-Generated `107feecol` Blocks

For blocks created by the ledger itself (e.g., during an upgrade/migration):

- MUST include `tx` with:
  - `op` set to `"107set_fee_collector"`;
  - `fee_collector` present or absent according to the intended configuration (collect vs burn).
- MAY omit `tx.created_at_time` and `tx.caller`.
- If `tx.caller` is included, it SHOULD be the ledger canister principal.


### `icrc107_get_fee_collector`

This method retrieves the currently active fee collector account. Unless changed, these settings apply to the next block added to the ledger.

```
icrc107_get_fee_collector: () -> (variant { Ok: opt Account; Err: record { error_code : nat; message : text } }) query;
```


This method returns the **currently active** fee collector account:

- If the response is `opt Account = ?account`, fees are collected by that account. This corresponds to the ledger’s current configuration and, if a `107feecol` block exists, the most recent one having a `fee_collector` field in its `tx`.  
- If the response is `opt Account = null`, then **no account currently collects fees**:
  - If one or more `107feecol` blocks exist, this corresponds to the most recent such block **omitting** the `fee_collector` field in its `tx`, meaning “burn all fees”.  
  - If no `107feecol` block exists yet, the ledger is still in **legacy mode** and fees are handled as described in the *Legacy Fee Collection Mechanism*.

This method strictly reflects the ledger’s current internal configuration and does not retroactively interpret past blocks.








#### Example: set fee collector to a specific account (produced by `icrc107_set_fee_collector`)

Below is an example of a valid **fee collector configuration block** that sets the fee collector to a specific account:

```
variant { Map = vec {
  // Block type
  record { "btype"; variant { Text = "107feecol" }};

  // Top-level tx (constructed per Canonical `tx` Mapping)
  record { "tx"; variant { Map = vec {
    record { "op"; variant { Text = "107set_fee_collector" }};
    record { "fee_collector"; variant { Array = vec {
      // Account = [owner, subaccount?]
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // owner principal bytes
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" } // 32-byte subaccount
    }}};
    record { "created_at_time"; variant { Nat = 1_750_951_727_000_000_000 : nat }}; // ns since Unix epoch
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" }};     // caller principal bytes
  }}};

  // Standard block metadata
  record { "ts";    variant { Nat = 1_741_312_737_184_874_392 : nat }};
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51" }};
}}
```
If one wanted to set the fee collector to be the default account of principal
identified by `"\00\00\00\00\02\00\01\0d\01\01"`, then the
`"fee_collector"` field within `tx` should be set to:

`variant { Array = vec { variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01"} } }`


#### Example: explicitly burn all fees from this point onward (produced by `icrc107_set_fee_collector`)


A block that explicitly sets fee burning by **omitting** the fee collector (i.e., all fees are burned from this point onward):

```text
variant { Map = vec {
  // Block type
  record { "btype"; variant { Text = "107feecol" }};

  // Top-level tx (constructed per Canonical `tx` Mapping)
  record { "tx"; variant { Map = vec {
    record { "op"; variant { Text = "107set_fee_collector" }};
    // NOTE: there is intentionally NO "fee_collector" entry here.
    // Its absence means "burn all fees from this block onward".
    record { "created_at_time"; variant { Nat = 1_750_951_728_000_000_000 : nat }};
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\00\00\01\01" }};
  }}};

  // Standard block metadata
  record { "ts";    variant { Nat = 1_741_312_737_184_874_392 : nat }};
  record { "phash"; variant { Blob = blob "\2d\86\7f\34\c7\2d\1e\2d\00\84\10\a4\00\b0\b6\4c\3e\02\96\c9\e8\55\6f\dd\72\68\e8\df\8d\8e\8a\ee" }};
}}




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

The **DFINITY-maintained** ICRC ledgers include a fee collection mechanism which, for completeness, is described below.

Until the first block of type `107feecol`, the ledger follows the following legacy behavior.

- If `fee_col` is not set, all fees are burned.
- If `fee_col` is set in a block, the designated account collects only transfer fees (`1xfer`, `2xfer`). Fees for all other operations (e.g., `2approve`) are burned in DFINITY-maintained ICRC-3 ledgers.


New implementations SHOULD avoid using `fee_col` and instead use `fee_collector` for all fee collection settings. Legacy behavior is provided for backward compatibility only and MAY be deprecated in future versions of this standard.

###  Handling Fee Collection for ICRC-1 and ICRC-2 Blocks

To determine who collects the fee in a block:

1. Check for fee collector configuration

   - If a previous block set `fee_collector`, the ledger uses the behavior specified by this standard.

2. If no `fee_collector` setting exists (legacy behavior):

   - If the block is of type `2approve` then the fee is burned
   - If the block is a transfer block, i.e. of type `1xfer` or `2xfer`:
      - If `fee_col` is specified in the block the fee is collected by `fee_col`.
      - If `fee_col_block` is specified use the `fee_col` from the referenced block index.
      - If neither `fee_col` nor `fee_col_block` are specified, then the fee is burned.
