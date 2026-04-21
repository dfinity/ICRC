# ICRC-123: Freeze and Unfreeze Blocks

| Status |
|:------:|
| Draft  |

## Introduction

Regulatory requirements or platform policies may require temporarily restricting interactions with certain accounts or principals. ICRC-123 provides explicit block types to record these actions transparently on the ledger, distinct from standard transactional blocks:

- **`123freezeaccount`** / **`123unfreezeaccount`** — freeze or unfreeze a specific account.
- **`123freezeprincipal`** / **`123unfreezeprincipal`** — freeze or unfreeze all accounts owned by a principal.

Each block carries a minimal `tx` sufficient to determine its semantic effect; additional provenance MAY be included to enhance auditability.

## Dependencies

- **ICRC-3** — Provides the block log format, Value encoding, hashing, certification,
  and the canonical `tx` mapping rules that this standard extends.

## Common Elements

This standard follows the conventions set by ICRC-3, inheriting key structural components.

- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`). If present, the subaccount **MUST** be exactly 32 bytes.
- **Principals** are represented as `variant { Blob = <principal_bytes> }`.
- **Timestamps:** `ts` is **nanoseconds since the Unix epoch**, encoded as `Nat` but **MUST fit into `nat64`**.
- **Parent hash:** `phash : Blob` **MUST** be present if the block has a parent (omit for the genesis block).

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes the **minimal information** about the freeze/unfreeze operation sufficient to determine its semantic effect. Additional provenance (e.g., `caller`, `reason`, `ts`) MAY be included but is not required for semantics.

Each block consists of the following top-level fields:

| Field | Type (ICRC-3 `Value`) | Required | Description |
|------|-------------------------|----------|-------------|
| `btype` | `Text` | Yes | MUST be one of: `"123freezeaccount"`, `"123unfreezeaccount"`, `"123freezeprincipal"`, `"123unfreezeprincipal"`. |
| `ts` | `Nat` | Yes | Timestamp (ns since Unix epoch) when the block was added to the ledger. MUST fit in `nat64`. |
| `phash` | `Blob` | Yes/No | Hash of the parent block; omitted only for the genesis block. |
| `tx` | `Map(Text, Value)` | Yes | Minimal operation details (see below). |

### `tx` Field Schemas (minimal)

#### `123freezeaccount`

| Field   | Type (ICRC-3 `Value`)                               | Req | Description            |
|---------|------------------------------------------------------|-----|------------------------|
| `account` | `variant { Array = vec { V1 [, V2] } }`¹          | Yes | The account being frozen. |

#### `123unfreezeaccount`

| Field   | Type (ICRC-3 `Value`)                               | Req | Description              |
|---------|------------------------------------------------------|-----|--------------------------|
| `account` | `variant { Array = vec { V1 [, V2] } }`¹          | Yes | The account being unfrozen. |

#### `123freezeprincipal`

| Field      | Type (ICRC-3 `Value`)                 | Req | Description             |
|------------|----------------------------------------|-----|-------------------------|
| `principal` | `variant { Blob = <principal_bytes> }` | Yes | The principal being frozen. |

#### `123unfreezeprincipal`

| Field      | Type (ICRC-3 `Value`)                 | Req | Description               |
|------------|----------------------------------------|-----|---------------------------|
| `principal` | `variant { Blob = <principal_bytes> }` | Yes | The principal being unfrozen. |

¹ `V1 = variant { Blob = <owner_principal> }`; optional `V2 = variant { Blob = <subaccount32> }` (exactly 32 bytes).

### Optional Provenance (non-semantic)

Producers MAY include non-semantic provenance fields within `tx`, such as:

- `caller : Blob` — principal that initiated the action (when applicable).
- `reason : Text` — human-readable context for the action.
- `ts : Nat` — caller-supplied timestamp in nanoseconds (MUST fit in `nat64`).
- `policy_ref : Text` — identifier for the policy/order/proposal under which the action occurred.
- `mthd : Text` — the method discriminator identifying which standardized method produced the block. This field is **optional** in ICRC-123, but when a separate standard defines methods that create ICRC-123 blocks, that standard **SHOULD** include a method discriminator (recommended field name: `mthd`) to make the call uniquely identifiable from `tx`.

  **Namespacing rule (from ICRC-3):** Method discriminator values SHOULD be namespaced by the standard that defines the method to avoid collisions. Use a numeric ICRC prefix and a lowercase op name:

  - Format: `<icrc_number><op_name>`
  - Examples: `147freeze_principal`, `147unfreeze_account`

  The method discriminator is **provenance only**; it MUST NOT affect block semantics or verification.




> **Informative note (recoverability):** Implementations **SHOULD** provide mechanisms (e.g., archives or lookups) to retrieve extended invocation context not present in `tx` when useful for audits. The authorization model that permits these actions is implementation-defined.


### Guidance for Standards That Define Methods

A standard that defines ledger methods which produce ICRC-123 blocks (e.g., “freeze principal” or “unfreeze account”) SHOULD:

1. **Include a method discriminator** in the resulting block’s `tx` map.
   - The recommended field name is `mthd`; alternatives (e.g., `op`) are permitted provided the choice is documented in the canonical `tx` mapping.
   - Use a namespaced value per ICRC-3: `<icrc_number><op_name>` (e.g., `"mthd" = "147freezeprincipal"`).
   - This makes the call uniquely identifiable and prevents collisions across standards.

2. **Define a canonical mapping** from the method’s call parameters to the block’s minimal `tx` fields:  
   - `123freezeaccount` / `123unfreezeaccount`: map the account argument to `tx.account`.  
   - `123freezeprincipal` / `123unfreezeprincipal`: map the principal argument to `tx.principal`.  
   - Do **not** add extra semantic fields; provenance such as `caller`, `reason`, `ts`, `policy_ref` MAY be included but MUST NOT affect semantics.

3. **Document deduplication inputs** (if any). If the method uses a caller-supplied timestamp, put it in `tx.ts` (ns; MUST fit `nat64`).



## Semantics

### Account Status

Given the state of the ledger at a particular block height `h`, an account `acc = (owner: Principal, subaccount: opt Blob)` is considered **RESTRICTED** iff the most recent freeze/unfreeze block at or before `h` that affects `acc` is a freeze block (`123freezeaccount` or `123freezeprincipal`).

A block affects `acc` if it satisfies one of the following:
- It is a `123freezeaccount` or `123unfreezeaccount` block where `tx.account` matches `acc`.
- It is a `123freezeprincipal` or `123unfreezeprincipal` block where `tx.principal` equals `owner` of `acc`.

If the most recent affecting block is an unfreeze block (`123unfreezeaccount` or `123unfreezeprincipal`), or if none exist, the account is **NON-RESTRICTED**.

Implications:
- An account freeze can be lifted by a later unfreeze of the **same account** or a later unfreeze of the **owning principal**.
- A principal freeze applies to all accounts owned by that principal until lifted, unless a more recent specific `123freezeaccount` targets an account after the principal is unfrozen.

### Ledger Enforcement Rules

- **Transfers:**  
  - MUST reject any `icrc1_transfer` / `icrc2_transfer_from` where the **sender** (`from`) is RESTRICTED.  
  - MAY reject or allow incoming transfers to a RESTRICTED **recipient** (`to`) per ledger policy.
- **ICRC-2 Operations:**  
  - `icrc2_approve` (granting approval): a RESTRICTED account MUST NOT grant approvals.  
  - `icrc2_approve` (receiving approval): policy-defined; even if granted, a RESTRICTED spender MUST NOT use it while restricted.  
  - `icrc2_transfer_from` (acting as spender): a RESTRICTED account MUST NOT act as spender.
- Freeze/unfreeze blocks do not retroactively modify prior transactions; they apply to transactions attempted **at or after** their block height.
- Freeze/unfreeze blocks MUST be permanently recorded and included in the hash chain.

### Idempotency and Redundancy

- A ledger MAY reject **freeze** blocks that would have no effect on the current status at the same level (e.g., freezing an already-frozen account via the same mechanism), or MAY choose to record them for auditability.
- A ledger MUST accept **unfreeze** blocks even when the target is not frozen at that level. Under the latest-action-wins rule, an unfreeze block at one level (e.g., principal) can lift freeze blocks recorded at another level (e.g., account). Rejecting such calls would prevent legitimate cross-level unfreezing.
- Clients interpreting freeze status MUST follow the **"latest-action-wins" rule** as defined in "Account Status": the most recent freeze or unfreeze block affecting an account or principal determines its effective status.

### Querying Freeze Status

Ledgers implementing this standard SHOULD expose a query (e.g., `is_account_restricted(account): bool`) that returns whether an account is currently RESTRICTED per the rules above. This is a convenience and does not replace auditing from history.

## Compliance Reporting

Ledgers implementing this standard MUST report supported block types via `icrc3_supported_block_types`:

```candid
vec {
    record { block_type = "123freezeaccount";    url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123/ICRC-123.md" };
    record { block_type = "123unfreezeaccount";  url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123/ICRC-123.md" };
    record { block_type = "123freezeprincipal";  url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123/ICRC-123.md" };
    record { block_type = "123unfreezeprincipal"; url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123/ICRC-123.md" };
}
```


## Example Blocks




### 123freezeaccount Example

```
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeaccount" }};
  record { "ts"; variant { Nat = 1_747_773_480_000_000_000 : nat }}; // Example: 2025-05-19T12:38:00Z
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\3a\1b\fc\20\d4\e7\05\91\8c\72\b3\4f" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // Optional provenance: the principal that invoked the operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }}; // Example caller principal (e.g., a compliance officer canister)
    // The account being frozen (owner + subaccount)
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // Example owner principal of the account
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Example subaccount
    }}};
    // Optional reason
    record { "reason"; variant { Text = "Regulatory compliance order #REG-1138" }};
    record { "policy_ref"; variant { Text = "REG-1138" }};
  }}};
}};
```



### 123unfreezeaccount Example

```
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeaccount" }};
  record { "ts"; variant { Nat = 1_747_773_540_000_000_000 : nat }}; // Example: 2025-05-19T12:39:00Z
  record { "phash"; variant { Blob = blob "\e8\a1\03\ff\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\12\34\56\78\9a\bc\de\f0\13\57\9b\df" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // Optional provenance: the principal that invoked the operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }}; // Example caller principal
    // The account being unfrozen
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };
    }}};
    // Optional reason
    record { "reason"; variant { Text = "Compliance review complete. Order #REG-1138 lifted." }};
  }}};
}};

```


### 123freezeprincipal Example
```
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeprincipal" }};
  record { "ts"; variant { Nat = 1_747_773_600_000_000_000 : nat }}; // Example: 2025-05-19T12:40:00Z
  record { "phash"; variant { Blob = blob "\f0\1d\9b\2a\10\20\30\40\50\60\70\80\90\a0\b0\c0\d0\e0\f0\00\24\68\ac\e0\13\57\9b\df\48\ac\01\23" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // Optional provenance: the principal that invoked the operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }}; // Example caller (e.g., DAO canister)
    // The principal being frozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06\ef\cd\ab\01\23\45\67\89\12\34\56\78\90\ab\cd\02" }}; // Example principal to freeze
    // Optional reason
    record { "reason"; variant { Text = "Platform terms of service violation." }};
  }}};
}};

```

### 123unfreezeprincipal Example
```
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeprincipal" }};
  record { "ts"; variant { Nat = 1_747_773_660_000_000_000 : nat }}; // Example: 2025-05-19T12:41:00Z
  record { "phash"; variant { Blob = blob "\c3\45\e6\b9\fe\dc\ba\98\76\54\32\10\ef\cd\ab\89\67\45\23\01\fe\dc\ba\98\76\54\32\10\ef\cd\ab\89" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // Optional provenance: the principal that invoked the operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }}; // Example caller
    // The principal being unfrozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06\ef\cd\ab\01\23\45\67\89\12\34\56\78\90\ab\cd\02" }};
    // Optional reason (example of omission for brevity, or if not applicable)
    // record { "reason"; variant { Text = "Appeal successful." }};
  }}};
}};

```

### Informative Example: Integration with a Standardized Method

ICRC-123 defines only block types and their semantics. It does not define any ledger methods.
However, future standards may specify methods that map directly to these block types.
For illustration, suppose a future standard (e.g., ICRC-147) introduces the method:

```
icrc147_freeze_principal : (principal, opt text) -> result nat
```

Invoking this method with a target principal and an optional reason could produce a
`123freezeprincipal` block on-chain. A possible encoding is shown below:

```
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeprincipal" }};
  record { "ts"; variant { Nat = 1_747_800_000_000_000_000 : nat }};
  record { "phash"; variant { Blob = blob "\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99" }};
  record { "tx"; variant { Map = vec {
    // Namespaced op from the method-defining standard (ICRC-147)
    record { "mthd"; variant { Text = "147freezeprincipal" }};
    // Optional provenance (non-semantic)
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }};
    record { "principal"; variant { Blob = blob "\94\85\a4\06\ef\cd\ab\01\23\45\67\89\12\34\56\78\90\ab\cd\02" }};
    record { "reason"; variant { Text = "Sanctions order #147-2025" }};
  }}};
}}

```
This example is non-normative and illustrates how a standardized method can map into the ICRC-123 block structure while using a namespaced method discriminator (`tx.mthd`) for unambiguous identification. The authoritative semantics remain defined by the ICRC-123 block types.