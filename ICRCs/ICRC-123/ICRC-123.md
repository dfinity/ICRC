# ICRC-123: Freezing and Unfreezing Accounts and Principals

## Status

Draft

## Introduction

This standard defines new block types for ICRC-compliant ledgers that enable freezing and unfreezing of accounts and principals. These operations are relevant in regulatory contexts or under legal obligations where temporarily or permanently disabling interactions with certain accounts or identities is necessary.

## Motivation

Freezing an account or principal can be a regulatory requirement in certain jurisdictions, or be required by platform policy. These operations must be reflected transparently on-chain, and must be designed to minimize ambiguity and maximize auditability. This standard provides a minimal yet extensible structure for such blocks.

## Block Types

This standard introduces the following block types:

- `123freezeaccount`: Freezes an account, preventing it from initiating transfers.
- `123unfreezeaccount`: Unfreezes a previously frozen account.
- `123freezeprincipal`: Freezes a principal, disabling interactions from any account controlled by this principal.
- `123unfreezeprincipal`: Unfreezes a previously frozen principal.

Each block contains a `tx` field with minimal information about the entity affected, but can be extended with additional fields to include more context.

## Role of `tx`

The `tx` field in each of the above block types captures the payload of the operation that triggered the block. This field is structured as a map (a vector of key-value records) and is intentionally minimal. It typically contains only the identity of the account or principal that is being frozen or unfrozen.

The field is designed to be extensible. Implementations are free to include additional keys in the `tx` map to provide extra context, such as:

- The principal that initiated the operation (e.g., a governance canister or privileged controller).
- The method that was invoked to trigger the operation.
- The reason for freezing or unfreezing.

Including such fields improves auditability and can support more nuanced policies in the ledger's business logic.

### Example: Extending the `tx` field

#### For `123freezeaccount`

| Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `account`    | `Array(vec { Blob [, Blob] })` | Yes     | The account to freeze. |
| `reason`     | `Text`                         | Yes     | Reason for freezing the account. |

#### For `123unfreezeaccount`

|Field        | Type (ICRC-3 `Value`)         | Required | Description |
|--------------|-------------------------------|----------|-------------|
| `account`    | `Array(vec { Blob [, Blob] })` | Yes     | The account to unfreeze. |
| `reason`     | `Text`                         | Yes     | Reason for unfreezing the account. |

#### For `123freezeprincipal`

| Field        | Type (ICRC-3 `Value`) | Required | Description |
|--------------|------------------------|----------|-------------|
| `principal`  | `Blob`                | Yes      | The principal to freeze. |
| `reason`     | `Text`                | Yes      | Reason for freezing the principal. |

#### For `123unfreezeprincipal`

| Field        | Type (ICRC-3 `Value`) | Required | Description |
|--------------|------------------------|----------|-------------|
| `principal`  | `Blob`                | Yes      | The principal to unfreeze. |
| `reason`     | `Text`                | Yes      | Reason for unfreezing the principal. |
## Semantics

This section defines the semantics of the freeze and unfreeze block types introduced by this standard.

### Account Status

Given the state of the ledger at a particular block height `h`, an account `acc = (owner: Principal, subaccount: Blob)` is considered **RESTRICTED** if and only if the most recent freeze or unfreeze block at or before height `h` that affects `acc` is a freeze.

A block is considered to affect an account if it satisfies one of the following:

- It is a `123freezeaccount` or `123unfreezeaccount` block whose `account` field matches `acc`.
- It is a `123freezeprincipal` or `123unfreezeprincipal` block whose `principal` field matches the `owner` of `acc`.

To determine whether an account is RESTRICTED, ledgers MUST identify the most recent block at or before height `h` that affects the account, and check whether it is of type `123freezeaccount` or `123freezeprincipal`.

This means:

- A freeze of an account (`123freezeaccount`) can be lifted by a later unfreeze of the same account (`123unfreezeaccount`) **or** by a later unfreeze of the owning principal (`123unfreezeprincipal`).
- A freeze of a principal (`123freezeprincipal`) can be lifted by a later unfreeze of that principal (`123unfreezeprincipal`).
- An unfreeze block always overrides any earlier freeze affecting the same account or principal, regardless of whether the freeze was explicit (account-level) or implicit (principal-level).

Otherwise, the account is considered **NON-RESTRICTED**.


### Ledger Enforcement Rules

- A ledger **MUST reject** any transfer transaction (`icrc1_transfer` or `icrc2_transfer_from`) where the **sender or recipient account is currently RESTRICTED**.
- Freeze and unfreeze blocks do **not** modify or invalidate previous transactions. They apply only to transactions **at or after** the block height at which the freeze/unfreeze block is recorded.
- Freeze and unfreeze blocks MUST be **permanently recorded** and included in the block hash chain.

### Authorization

- Each freeze and unfreeze block includes an `authorizer` field, which records the principal who authorized the action.
- This standard does **not prescribe** how the ledger verifies that the `authorizer` is permitted to freeze or unfreeze. Ledger implementations MAY use governance mechanisms, access control lists, or DAO-based authorization.

### Idempotency and Redundancy

- A ledger MAY reject freeze or unfreeze blocks that would have **no effect** (e.g., freezing an already frozen account), or MAY choose to **record them anyway** for auditability.
- Clients interpreting freeze status MUST follow a **"latest-action-wins" rule**: the most recent freeze or unfreeze block affecting an account or principal determines its effective status.

### Querying Freeze Status

Ledgers implementing this standard SHOULD expose a query interface (e.g., `is_account_frozen(account)`) that returns whether an account is currently restricted. This serves as a convenience layer and does not replace auditing based on block history.




## Compliance Reporting

Ledgers implementing this standard MUST return the following response to `icrc3_supported_block_types` with a URL pointing to the standard defining each block type:



```
vec {
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeaccount" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123freezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
    variant { Record = vec {
        record { "btype"; variant { Text = "123unfreezeprincipal" }};
        record { "url"; variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-123.md" }};
    }};
}
```

## Example Blocks




### 123freezeaccount Example

```
variant {
    Map = vec {
      record { "btype"; variant { Text = "123freezeaccount" }};  // Block type identifier
      record { "ts"; variant { Nat = 1741312737184874393 }};     // Timestamp when the block was appended (nanoseconds since epoch)
      record { "phash"; variant { Blob = blob "\9a\6f\bd\5b\18\65\2c\fa\6d\20\de\4d\fa\43\fc\96\33\e5\6a\1b" }};  // Hash of the previous block in the ledger chain
      record { "tx"; variant { Map = vec {
          record { "account"; variant { Array = vec {
              variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };   // Account to be frozen (owner principal)
              variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };   // Another account to be frozen
          }}};
          record { "reason"; variant { Text = "Security breach" }};  // The reason for the freeze operation
      }}};
    }};
```

This example illustrates a `freezeaccount` block where the `tx` field includes more than just the `account`. By including fields like `caller`, `method`, and `reason`, the ledger provides greater transparency and traceability.

---

```
variant {
    Map = vec {
      record { "btype"; variant { Text = "123unfreezeaccount" }};  // Block type identifier
      record { "ts"; variant { Nat = 1741312737184874392 }};       // Timestamp when the block was appended (nanoseconds since epoch)
      record { "phash"; variant { Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51" }};  // Hash of the previous block in the ledger chain
      record { "tx"; variant { Map = vec {
          record { "account"; variant { Array = vec {
              variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };   // Account to be unfrozen (owner principal)
              variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f\bc\8d\a3\3e\5d\ba\bc\2f\38\69\60\5d\c7\a1\c9\53\1f\70\a3\66\c5\a7\e4\21" };   // Another account to be unfrozen
          }}};
          record { "reason"; variant { Text = "Legal case resolved" }};  // The reason for the unfreeze operation
      }}};

    }};
```


### 123freezeprincipal Example
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123freezeprincipal" }};  // Block type identifier
    record { "ts"; variant { Nat = 1741312737184874393 }};     // Timestamp when the block was appended (nanoseconds since epoch)
    record { "phash"; variant { Blob = blob "\9a\6f\bd\5b\18\65\2c\fa\6d\20\de\4d\fa\43\fc\96\33\e5\6a\1b" }};  // Hash of the previous block in the ledger chain
    record { "tx"; variant { Map = vec {
      record { "principal"; variant { Array = vec {
          variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };   // Principal to be frozen
      }}};
      record { "reason"; variant { Text = "Suspicion of illicit activity" }};  // Reason for freezing the principal
  }}};

}};
```

### 123unfreezeprincipal Example
```
variant {
  Map = vec {
    record { "btype"; variant { Text = "123unfreezeprincipal" }};  // Block type identifier
    record { "ts"; variant { Nat = 1741312737184874392 }};       // Timestamp when the block was appended (nanoseconds since epoch)
    record { "phash"; variant { Blob = blob "\d5\c7\eb\57\a2\4e\fa\d4\8b\d1\cc\54\9e\49\c6\9f\d1\93\8d\e8\02\d4\60\65\e2\f2\3c\00\04\3b\2e\51" }};  // Hash of the previous block in the ledger chain
    record { "tx"; variant { Map = vec {
      record { "principal"; variant { Array = vec {
          variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" };   // Principal to be unfrozen
      }}};
      record { "reason"; variant { Text = "Court order" }};  // Reason for unfreezing the principal
  }}};

}};
```
