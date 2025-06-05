# ICRC-123: Freezing and Unfreezing Accounts and Principals

## Status

Draft

## Introduction

This standard defines new block types for ICRC-compliant ledgers that enable freezing and unfreezing of accounts and principals. These operations are primarily relevant in regulatory contexts or under specific legal or platform policy obligations where temporarily restricting interactions with certain accounts or identities is necessary. Freezing an account or principal must be reflected transparently on-chain, using a format designed for auditability and clear semantics. The transaction details (`tx`) within each block explicitly include the `caller` principal that authorized the operation.

## Motivation

Regulatory requirements or platform policies may necessitate the ability to freeze accounts or principals. This standard provides explicit block types (`123freezeaccount`, `123unfreezeaccount`, `123freezeprincipal`, `123unfreezeprincipal`) to record these actions transparently on the ledger, distinct from standard transactional blocks. It defines a block structure that includes the initiator (`caller`) and essential details for the operation, enhancing on-chain auditability.

## Common Elements
This standard follows the conventions set by ICRC-3, inheriting key structural components.
- **Accounts** are represented using the ICRC-3 `Value` type, specifically as a `variant { Array = vec { V1 [, V2] } }` where `V1` is `variant { Blob = <owner_principal> }` representing the account owner, and `V2` is `variant { Blob = <subaccount> }` representing the subaccount. If no subaccount is specified, the `Array` MUST contain only one element (`V1`).
- **Principals** (such as the `caller`) are represented using the ICRC-3 `Value` type as `variant { Blob = <principal_bytes> }`.
- Each block includes `phash`, a `Blob` representing the hash of the parent block, and `ts`, a `Nat` representing the timestamp of the block.

## Block Types & Schema

Each block introduced by this standard MUST include a `tx` field containing a map. This map encodes information about the freeze/unfreeze operation, including the `caller` principal, the target entity, and basic context.

**Important Note on Transaction Recoverability:** The `tx` field defined below now includes the `caller` principal. However, for full auditability and transparency in complex scenarios, ledger implementations compliant with ICRC-123 **MUST** ensure that any other details of the original transaction invocation not captured in `tx` can be recovered independently. This could include, but is not limited to, the full arguments passed to the ledger method (if more complex than the data in `tx`), or any intermediary calls if the operation was part of a multi-step process. Mechanisms for recovering such extended data (e.g., via archive queries or specific lookup methods) remain implementation-dependent. The rules determining *who* is authorized to invoke these freeze/unfreeze operations are an implementation detail of the ledger's governance model.

Each block defined by this standard consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                 | Yes      | MUST be one of: `"123freezeaccount"`, `"123unfreezeaccount"`, `"123freezeprincipal"`, `"123unfreezeprincipal"`. |
| `ts`     | `Nat`                  | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                 | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`     | Yes      | Encodes information about the freeze/unfreeze operation, including the caller. See schemas below. |

### `tx` Field Schemas

#### For `123freezeaccount`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method causing this block. |
| `account`    | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account being frozen. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for freezing the account. |

#### For `123unfreezeaccount`

| Field        | Type (ICRC-3 `Value`)                                        | Required | Description |
|--------------|--------------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method causing this block. |
| `account`    | `Value` (Must be `variant { Array = vec { V1 [, V2] } }`)¹ | Yes      | The account being unfrozen. |
| `reason`     | `Text`                                                       | Optional | Human-readable reason for unfreezing the account. |

#### For `123freezeprincipal`

| Field        | Type (ICRC-3 `Value`)                                    | Required | Description |
|--------------|----------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method causing this block. |
| `principal`  | `Value` (Must be `variant { Blob = <principal_bytes> }`) | Yes      | The principal being frozen. |
| `reason`     | `Text`                                                   | Optional | Human-readable reason for freezing the principal. |

#### For `123unfreezeprincipal`

| Field        | Type (ICRC-3 `Value`)                                    | Required | Description |
|--------------|----------------------------------------------------------|----------|-------------|
| `caller`     | `Value` (Must be `variant { Blob = <principal_bytes> }`)     | Yes      | The principal that invoked the ledger method causing this block. |
| `principal`  | `Value` (Must be `variant { Blob = <principal_bytes> }`) | Yes      | The principal being unfrozen. |
| `reason`     | `Text`                                                   | Optional | Human-readable reason for unfreezing the principal. |

¹ Where `V1` is `variant { Blob = <owner_principal> }` and `V2` is `variant { Blob = <subaccount> }`. If no subaccount exists, the `Array` contains only `V1`.

## Semantics

This section defines the semantics of the freeze and unfreeze block types introduced by this standard.

### Account Status

Given the state of the ledger at a particular block height `h`, an account `acc = (owner: Principal, subaccount: opt Blob)` is considered **RESTRICTED** if and only if the most recent freeze or unfreeze block at or before height `h` that affects `acc` is a freeze block (either `123freezeaccount` or `123freezeprincipal`).

A block is considered to affect an account `acc` if it satisfies one of the following:
- It is a `123freezeaccount` or `123unfreezeaccount` block where the `tx.account` field matches `acc`.
- It is a `123freezeprincipal` or `123unfreezeprincipal` block where the `tx.principal` field matches the `owner` of `acc`.

To determine whether an account is RESTRICTED, ledgers MUST identify the most recent block at or before height `h` that affects the account, and check whether its `btype` is `123freezeaccount` or `123freezeprincipal`. If the most recent affecting block is an unfreeze block (`123unfreezeaccount` or `123unfreezeprincipal`), or if no such affecting blocks exist, the account is **NON-RESTRICTED**.

This "latest-action-wins" rule implies:
- A freeze of an account (`123freezeaccount`) can be lifted by a later unfreeze of the same account (`123unfreezeaccount`) or by a later unfreeze of the owning principal (`123unfreezeprincipal`).
- A freeze of a principal (`123freezeprincipal`) can be lifted by a later unfreeze of that principal (`123unfreezeprincipal`). It also implicitly unfreezes all accounts owned by that principal unless a more recent, specific `123freezeaccount` block targets one of those accounts.

### Ledger Enforcement Rules

- **Transfers:**
    - A ledger **MUST reject** any `icrc1_transfer` or `icrc2_transfer_from` transaction where the **sender** account (the `from` account in the operation) is currently RESTRICTED.
    - The ledger **MAY**, according to its policy, also reject `icrc1_transfer` or `icrc2_transfer_from` transactions if the **recipient** account (the `to` account in the operation) is RESTRICTED, or it MAY allow incoming funds to a RESTRICTED recipient.
- **ICRC-2 Operations:**
    - **`icrc2_approve` (Granting Approval):** If an account is RESTRICTED, its owner **MUST NOT** be able to authorize an `icrc2_approve` transaction where this restricted account is the one granting the approval (i.e., the `account` argument in `icrc2_approve` which specifies the owner of the funds being approved for spending).
    - **`icrc2_approve` (Receiving Approval):** The ledger's policy SHOULD define whether an approval can be granted *to* a RESTRICTED account (i.e., a RESTRICTED account being the `spender` argument in an `icrc2_approve` call initiated by an unrestricted account owner). Even if an approval is granted to a RESTRICTED account, that account **MUST NOT** be able to use this approval (e.g., by calling `icrc2_transfer_from`) while it remains RESTRICTED.
    - **`icrc2_transfer_from` (Acting as Spender):** An account that is RESTRICTED **MUST NOT** be able to initiate an `icrc2_transfer_from` call (i.e., act as an approved spender), even if it holds a valid approval for another account.
- Freeze and unfreeze blocks do **not** modify or invalidate previous transactions. They apply only to transactions attempted **at or after** the block height at which the freeze/unfreeze block is recorded and its state change takes effect.
- Freeze and unfreeze blocks MUST be **permanently recorded** and included in the block hash chain.

### Idempotency and Redundancy

- A ledger MAY reject freeze or unfreeze blocks that would have **no effect** on the current RESTRICTED status of the target account or principal (e.g., freezing an already frozen account via the same mechanism), or MAY choose to **record them anyway** for auditability.
- Clients interpreting freeze status MUST follow the **"latest-action-wins" rule** as defined in "Account Status": the most recent freeze or unfreeze block affecting an account or principal determines its effective status.

### Querying Freeze Status

Ledgers implementing this standard SHOULD expose a query interface (e.g., `is_account_restricted(account): bool`) that returns whether an account is currently RESTRICTED according to the rules defined in "Account Status". This serves as a convenience layer and does not replace auditing based on block history.

## Compliance Reporting

Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:

```candid
vec {
    // ... other supported types like ICRC-1 ...
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeaccount" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeaccount" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeprincipal" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md)" }}; // Placeholder URL
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeprincipal" }};
        record { "url"; variant { Text = "[https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md)" }}; // Placeholder URL
    }};
}

```

## Example Blocks




### 123freezeaccount Example

```
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeaccount" }};
  record { "ts"; variant { Nat = 1_747_773_480_000_000_000 : nat }}; // Example: 2025-05-19T12:38:00Z
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal that invoked the freeze_account operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\01" }}; // Example caller principal (e.g., a compliance officer canister)
    // The account being frozen (owner + subaccount)
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" }; // Example owner principal of the account
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" }; // Example subaccount
    }}};
    // Optional reason
    record { "reason"; variant { Text = "Regulatory compliance order #REG-1138" }};
  }}};
}};
```



### 123unfreezeaccount Example

```
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeaccount" }};
  record { "ts"; variant { Nat = 1_747_773_540_000_000_000 : nat }}; // Example: 2025-05-19T12:39:00Z
  record { "phash"; variant { Blob = blob "\e8\a1\03\ff\00\11\22\33\44\55\66\77\88\99\aa\bb\cc\dd\ee\ff" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal that invoked the unfreeze_account operation
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
  record { "phash"; variant { Blob = blob "\f0\1d\9b\2a\10\20\30\40\50\60\70\80\90\a0\b0\c0\d0\e0\f0\00" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal that invoked the freeze_principal operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\02" }}; // Example caller (e.g., DAO canister)
    // The principal being frozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06\ef\cd\ab\01\23\45\67\89\12\34\56\78\90\ab\cd\ef" }}; // Example principal to freeze
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
  record { "phash"; variant { Blob = blob "\c3\45\e6\b9\fe\dc\ba\98\76\54\32\10\00\00\00\00\00\00\00\00" }}; // Example parent hash
  record { "tx"; variant { Map = vec {
    // The principal that invoked the unfreeze_principal operation
    record { "caller"; variant { Blob = blob "\00\00\00\00\00\00\f0\0d\01\02" }}; // Example caller
    // The principal being unfrozen
    record { "principal"; variant { Blob = blob "\94\85\a4\06\ef\cd\ab\01\23\45\67\89\12\34\56\78\90\ab\cd\ef" }};
    // Optional reason (example of omission for brevity, or if not applicable)
    // record { "reason"; variant { Text = "Appeal successful." }};
  }}};
}};

```
