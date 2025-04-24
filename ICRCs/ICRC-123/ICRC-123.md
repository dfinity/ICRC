# ICRC-123: Freezing and Unfreezing Accounts and Principals

## Status

Draft

## Introduction

This standard defines new block types for ICRC-compliant ledgers that enable freezing and unfreezing of accounts and principals. These operations are primarily relevant in regulatory contexts or under specific legal or platform policy obligations where temporarily restricting interactions with certain accounts or identities is necessary. Freezing an account or principal must be reflected transparently on-chain, using a format designed for auditability and clear semantics.

## Motivation

Regulatory requirements or platform policies may necessitate the ability to freeze accounts or principals. This standard provides explicit block types (`123freezeaccount`, `123unfreezeaccount`, `123freezeprincipal`, `123unfreezeprincipal`) to record these actions transparently on the ledger, distinct from standard transactional blocks. It defines a minimal block structure sufficient for recording the action while relying on the ledger implementation to provide access to the full invocation context for auditability.

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`).
- **Principals** are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map that encodes the minimal information about the freeze/unfreeze operation required for identifying the target and providing basic context.

**Important Note on Transaction Recoverability:** The `tx` field defined below is intentionally minimal, containing only the data strictly necessary to identify the target entity (account or principal) and an optional reason. For full auditability and transparency, ledger implementations compliant with ICRC-123 **MUST** ensure that the complete details of the original transaction invocation that led to the freeze/unfreeze can be recovered independently. This includes, but is not limited to, the principal that invoked the ledger operation (the authorizer/caller), the specific ledger method called (e.g., `freeze_account`), and the full arguments passed to that method. Mechanisms for recovering this data (e.g., via archive queries or specific lookup methods) are implementation-dependent but necessary for compliance. The `tx` field itself is *not* designed to hold this exhaustive information.

Each block defined by this standard consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"123freezeaccount"`, `"123unfreezeaccount"`, `"123freezeprincipal"`, `"123unfreezeprincipal"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes minimal information about the freeze/unfreeze operation. See schemas below. |

### `tx` Field Schemas

#### For `123freezeaccount`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `account`    | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account being frozen. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for freezing the account. |

#### For `123unfreezeaccount`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `account`    | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account being unfrozen. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for unfreezing the account. |

#### For `123freezeprincipal`

| Field        | Type (ICRC-3 `Value`)                                    | Required | Description |
|--------------|----------------------------------------------------------|----------|-------------|
| `principal`  | `Value` (Must be `variant { Blob = <principal_bytes> }`) | Yes      | The principal being frozen. |
| `reason`     | `Text`                                                   | Optional | Human-readable reason for freezing the principal. |

#### For `123unfreezeprincipal`

| Field        | Type (ICRC-3 `Value`)                                    | Required | Description |
|--------------|----------------------------------------------------------|----------|-------------|
| `principal`  | `Value` (Must be `variant { Blob = <principal_bytes> }`) | Yes      | The principal being unfrozen. |
| `reason`     | `Text`                                                   | Optional | Human-readable reason for unfreezing the principal. |

¹ Where `V1` is `variant { Blob = <owner_principal> }` and `V2` is `variant { Blob = <subaccount> }`. If no subaccount exists, the `Array` contains only `V1`.

## Semantics

The recording of these blocks MUST influence the behavior of the ledger according to the following semantics. Implementations MUST clearly define how these states affect ledger operations (e.g., `icrc1_transfer`, `icrc2_approve`).

### Freeze Account (`123freezeaccount`)

- When a block of type `123freezeaccount` is recorded for a given `tx.account`, the ledger MUST enter a state where that specific account is considered "frozen".
- A frozen account MUST be prevented from being the initiator of operations that would typically require the account owner's control, such as:
    - Being the `from` account in an `icrc1_transfer`.
    - Being the `spender` account in an `icrc2_approve` (unless ledger policy allows approving while frozen).
- The ledger MAY allow incoming transfers (`icrc1_transfer` where the frozen account is the `to` account), depending on policy.
- The frozen status applies only to the specific account (owner + subaccount combination).

### Unfreeze Account (`123unfreezeaccount`)

- When a block of type `123unfreezeaccount` is recorded for a given `tx.account`, the ledger MUST reverse the "frozen" state for that specific account.
- The account MUST subsequently be allowed to initiate operations as normal (subject to other standard conditions like sufficient balance).

### Freeze Principal (`123freezeprincipal`)

- When a block of type `123freezeprincipal` is recorded for a given `tx.principal`, the ledger MUST enter a state where that principal is considered "frozen".
- A frozen principal MUST be prevented from initiating operations from *any* account where they are the owner (i.e., any account `{ owner = tx.principal; subaccount = * }`). This includes, but may not be limited to:
    - Preventing `icrc1_transfer` where the `from.owner` is the frozen principal.
    - Preventing `icrc2_approve` where the `spender.owner` is the frozen principal.
- The ledger MAY allow incoming transfers to accounts owned by the frozen principal, depending on policy.

### Unfreeze Principal (`123unfreezeprincipal`)

- When a block of type `123unfreezeprincipal` is recorded for a given `tx.principal`, the ledger MUST reverse the "frozen" state for that principal.
- Accounts owned by the principal MUST subsequently be allowed to initiate operations as normal (subject to other standard conditions).

## Compliance Reporting

Ledgers implementing this standard MUST return the following response (including entries for other supported types like ICRC-1) to `icrc3_supported_block_types`, with URLs pointing to the standards defining each block type:

```candid
vec {
    // ... other supported types like ICRC-1 ...
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }}; // Placeholder URL
    }};
}
```

## Block Examples

### Freeze Account Block Example

```candid
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeaccount" }};
  record { "ts"; variant { Nat = 1_741_319_263_000_000_000 : nat }}; // Approx 2025-04-14T15:07:43Z
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The account being frozen (owner + subaccount)
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // Example owner principal
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f..." }; // Example subaccount
    }}};
    // Optional reason
    record { "reason"; variant { Text = "Violation of terms" }};
  }}};
}};
```

### Unfreeze Account Block Example

```candid
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeaccount" }};
  record { "ts"; variant { Nat = 1_741_319_263_000_000_000 : nat }}; // Approx 2025-04-14T15:07:43Z
  record { "phash"; variant { Blob = blob "\e8\a1\03\ff..." }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The account being unfrozen
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // Example owner principal
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f..." }; // Example subaccount
    }}};
    // Optional reason
    record { "reason"; variant { Text = "Cleared by compliance team" }};
  }}};
}};
```

### Freeze Principal Block Example

```candid
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeprincipal" }};
  record { "ts"; variant { Nat = 1_741_319_263_000_000_000 : nat }}; // Approx 2025-04-14T15:07:43Z
  record { "phash"; variant { Blob = blob "\f0\1d\9b\2a..." }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal being frozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06..." }}; // Example principal
    // Optional reason
    record { "reason"; variant { Text = "Violation of platform policy" }};
  }}};
}};
```

### Unfreeze Principal Block Example

```candid
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeprincipal" }};
  record { "ts"; variant { Nat = 1_741_319_263_000_000_000 : nat }}; // Approx 2025-04-14T15:07:43Z
  record { "phash"; variant { Blob = blob "\c3\45\e6\b9..." }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal being unfrozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06..." }}; // Example principal
    // Optional reason
    record { "reason"; variant { Text = "Review complete, reinstated" }};
  }}};
}};
```
