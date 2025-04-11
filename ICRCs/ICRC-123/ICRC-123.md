# ICRC-123: Account and Principal Freezing & Unfreezing

ICRC-123 introduces new block types for recording account and unfreezing events in ICRC-compliant ledgers. These blocks provide a standardized way to document administrative actions restricting or restoring account activity. The `123freezeaccount` block records an account freeze event, while the `123unfreezeaccount` block documents the removal of such restrictions. Additionally, the `123freezeprincipal` block records the freezing of all accounts belonging to a principal, and the `123unfreezeprincipal` block records the unfreezing of all accounts belonging to a principal.

## Common Elements

This standard follows the conventions set by ICRC-3, inheriting key structural components. Accounts are recorded as an `Array` of one or two `Value` variants, where:

- The first element is a `variant { Blob = <owner principal> }`, representing the account owner.
- The second element is a `variant { Blob = <subaccount> }`, representing the subaccount. If no subaccount is specified, only the owner is included.

Additionally, each block includes:

- `phash`: a `Blob` representing the hash of the parent block.
- `ts`: a `Nat` representing the timestamp of the block.


## Block Types & Schema

This standard introduces four new block types:

- **Freeze Account**: `123freezeaccount`
- **Unfreeze Account**: `123unfreezeaccount`
- **Freeze Principal**: `123freezeprincipal`
- **Unfreeze Principal**: `123unfreezeprincipal`


Each block introduced by this standard MUST include a `tx` field containing a map that encodes the freeze or unfreeze transaction submitted to the ledger that caused this block to be created, similarly to how blocks that hold ICRC-1 and ICRC-2 transactions include, explicitly, the transaction that was submitted.

This enables canister clients, indexers, and auditors to reconstruct the exact instruction that led to the block being appended to the ledger.

Each block consists of the following top-level fields:

| Field    | Type (ICRC-3 `Value`) | Required | Description |
|----------|------------------------|----------|-------------|
| `btype`  | `Text`                | Yes      | MUST be one of: `"123freezeaccount"`, `"123unfreezeaccount"`, `"123freezeprincipal"`, or `"123unfreezeprincipal"`. |
| `ts`     | `Nat`                 | Yes      | Timestamp in nanoseconds when the block was added to the ledger. |
| `phash`  | `Blob`                | Yes      | Hash of the parent block. |
| `tx`     | `Map(Text, Value)`    | Yes      | Encodes the specific transaction that was submitted to the ledger and which resulted in the block being created. |

### `tx` Field Schemas

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


### 123unfreezeaccount Example

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
