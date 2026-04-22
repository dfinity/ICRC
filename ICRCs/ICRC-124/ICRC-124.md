# ICRC-124:  Pause, Unpause & Deactivate Blocks

| Status |
|:------:|
| Draft  |

## Introduction

Ledger lifecycle management may require administrative actions like pausing for upgrades, unpausing after checks, or deactivating at the end of a token's useful life. ICRC-124 provides explicit block types to record these ledger-wide state transitions transparently on-chain:

- **`124pause`** — temporarily halt all state-changing operations (e.g., for maintenance).
- **`124unpause`** — resume normal operation after a pause.
- **`124deactivate`** — permanently deactivate the ledger (terminal state).

## Dependencies

- **ICRC-3** — Provides the block log format, Value encoding, hashing, certification, and the canonical `tx` mapping rules that this standard extends.

## Common Elements

This standard follows the conventions set by ICRC-3, inheriting key structural components:

- **Principals** are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- **Timestamps:** `ts` is **nanoseconds since the Unix epoch**, encoded as `Nat` but **MUST fit into `nat64`**.
- **Parent hash:** `phash : Blob` **MUST** be present if the block has a parent (omit for the genesis block).

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes the **minimal information** about the ledger state change. Additional provenance MAY be included but is not required for semantics.

Each block consists of the following top-level fields:

| Field | Type (ICRC-3 `Value`) | Required | Description |
|-------|------------------------|----------|-------------|
| `btype` | `Text` | Yes | MUST be one of: `"124pause"`, `"124unpause"`, or `"124deactivate"`. |
| `ts` | `Nat` | Yes | Timestamp (ns since Unix epoch) when the block was added to the ledger. MUST fit in `nat64`. |
| `phash` | `Blob` | Yes/No | Hash of the parent block; omitted only for the genesis block. |
| `tx` | `Map(Text, Value)` | Yes | Minimal operation details (see below). |

### `tx` Field Schema (minimal)

For all `124pause`, `124unpause`, and `124deactivate` blocks:

- No required fields are needed for semantics.  
- The presence of the block type alone (`btype`) determines the state transition.  

### Optional Provenance (non-semantic)

Producers MAY include fields such as:

- `caller : Blob` — principal that invoked the operation.  
- `reason : Text` — human-readable context.  
- `ts : Nat` — caller-supplied timestamp (ns; MUST fit nat64).  
- `policy_ref : Text` — identifier for proposal/vote/policy.  
- `mthd : Text` — namespaced method discriminator, e.g. `148pause_ledger`.

These fields MUST NOT affect semantics or verification. Verifiers MUST ignore them.

> **Informative note (recoverability):** Implementations **SHOULD** provide mechanisms (e.g., archives or lookups) to retrieve extended invocation context not present in `tx` when useful for audits. The authorization model that permits these actions is implementation-defined.

---

## Semantics

### Pause Ledger (`124pause`)
- When a `124pause` block is recorded, the ledger MUST enter a "paused" state.
- While paused, the ledger MUST reject all state-changing operations except those required for governance or recovery (e.g., `124unpause`, optionally `124deactivate`, and operations like freeze/unfreeze if permitted by governance policy).
- Query calls SHOULD remain operational.
- A `124pause` block has no effect if the ledger is already paused or if it is in a terminal state due to deactivation.

### Unpause Ledger (`124unpause`)
- When a `124unpause` block is recorded, the ledger MUST exit the "paused" state and resume normal operation, unless it is already in the terminal state due to deactivation.
- An `124unpause` block has no effect if the ledger is already unpaused or deactivated.

### Deactivate Ledger (`124deactivate`)
- When a `124deactivate` block is recorded, the ledger MUST transition to a permanent "terminal" state.
- In this state:
  - All ingress calls that modify state MUST be rejected (transfers, approvals, mints, burns, freezes, pauses, unpauses, etc.).
  - Query calls retrieving historical data MUST remain available.
- The deactivated state is irreversible.

### Querying Ledger State

Ledgers implementing this standard SHOULD expose queries (e.g., `is_paused() : bool`,
`is_deactivated() : bool`) that return the ledger's current operational state
per the rules above. These are a convenience and do not replace auditing from
history. See **ICRC-154** for a standardized API.

---

## Guidance for Standards That Define Methods

A standard that defines ledger methods which produce ICRC-124 blocks (e.g., “pause ledger” or “deactivate ledger”) SHOULD:

1. **Include a method discriminator** in the resulting block’s `tx` map.
   - The recommended field name is `mthd`; alternatives (e.g., `op`) are permitted provided the choice is documented in the canonical `tx` mapping.
   - Use a namespaced value per ICRC-3: `<icrc_number><op_name>` (e.g., `"mthd" = "148pause_ledger"`).
   - This makes the call uniquely identifiable and prevents collisions across standards.

2. **Define a canonical mapping** from the method’s call parameters to the block’s minimal `tx` fields.  
   - Since 124 blocks have no required fields, only provenance may be mapped.  

3. **Document deduplication inputs** (if any). If the method uses a caller-supplied timestamp, put it in `tx.ts`.

---

## Compliance Reporting

Ledgers implementing this standard MUST return the following entries (along with entries for other supported block types) from `icrc3_supported_block_types`:

```candid
vec {
  record { block_type = "124pause";      url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124/ICRC-124.md" };
  record { block_type = "124unpause";    url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124/ICRC-124.md" };
  record { block_type = "124deactivate"; url = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-124/ICRC-124.md" };
}
```

## Example Blocks

### 124pause Example

```candid
variant { Map = vec {
    record { "btype"; variant { Text = "124pause" }};
    record { "ts"; variant { Nat = 1_747_774_560_000_000_000 : nat }};
    record { "phash"; variant {
        Blob = blob "\de\ad\be\ef\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff\10\20\30\40\50\60\70\80\90\a0\b0\c0"
    }};
    record { "tx"; variant { Map = vec {
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }};
        record { "reason"; variant { Text = "DAO vote #78: pause for scheduled maintenance." }};
    }}};
}};
```

### 124unpause Example

```candid
variant { Map = vec {
    record { "btype"; variant { Text = "124unpause" }};
    record { "ts"; variant { Nat = 1_747_778_160_000_000_000 : nat }};
    record { "phash"; variant {
        Blob = blob "\be\ba\fe\ca\01\02\03\04\05\06\07\08\09\0a\0b\0c\0d\0e\0f\10\11\12\13\14\15\16\17\18\19\1a\1b"
    }};
    record { "tx"; variant { Map = vec {
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }};
        record { "reason"; variant { Text = "Ledger resumes after maintenance window (DAO vote #79)." }};
    }}};
}};
```

### 124deactivate Example

```candid
variant { Map = vec {
    record { "btype"; variant { Text = "124deactivate" }};
    record { "ts"; variant { Nat = 1_747_864_560_000_000_000 : nat }};
    record { "phash"; variant {
        Blob = blob "\c0\ff\ee\00\10\20\30\40\50\60\70\80\90\a0\b0\c0\d0\e0\f0\00\11\22\33\44\55\66\77\88\99\aa\bb\cc"
    }};
    record { "tx"; variant { Map = vec {
        record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }};
        record { "reason"; variant { Text = "Token project sunset. Ledger permanently archived as per SNS DAO proposal #314." }};
    }}};
}};
```

### Informative Example: Integration with a Standardized Method
ICRC-124 defines only block types and their semantics. It does not define any ledger methods.
However, future standards may specify methods that map directly to these block types.

For illustration, suppose a future standard (e.g., ICRC-148) introduces the method:
```
icrc148_pause_ledger : (opt text) -> result nat
```

Invoking this method with an optional reason could produce a `124pause` block:
```
variant { Map = vec {
  record { "btype"; variant { Text = "124pause" }};
  record { "ts"; variant { Nat = 1_747_900_000_000_000_000 : nat }};
  record { "phash"; variant {
    Blob = blob "\aa\bb\cc\dd\ee\ff\00\11\22\33\44\55\66\77\88\99\01\23\45\67\89\ab\cd\ef\10\32\54\76\98\ba\dc\fe"
  }};
  record { "tx"; variant { Map = vec {
    record { "mthd"; variant { Text = "148pause_ledger" }};
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }};
    record { "reason"; variant { Text = "DAO vote #101: emergency pause" }};
  }}};
}};
```

This example is non-normative and illustrates how a standardized method can map into the ICRC-124 block structure while using a namespaced method discriminator (`tx.mthd`) for unambiguous identification. The authoritative semantics remain defined by the ICRC-124 block types.
