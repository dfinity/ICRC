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

```motoko
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeaccount" }},
  record { "ts"; variant { Nat = 1_741_312_737_184_874_392 }},
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }},
  record { "tx"; variant { Map = vec {
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" },
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f..." }
    }}},
    record { "caller"; variant { Blob = blob "\94\85\a4\06..." }},
    record { "method"; variant { Text = "freeze_account" }},
    record { "reason"; variant { Text = "Account involved in suspicious activity" }}
  }}}
}};
```

This example illustrates a `freezeaccount` block where the `tx` field includes more than just the `account`. By including fields like `caller`, `method`, and `reason`, the ledger provides greater transparency and traceability.

---

## Block Examples

### Freeze Account Block

```motoko
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeaccount" }},
  record { "ts"; variant { Nat = 1_741_312_737_184_874_392 }},
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }},
  record { "tx"; variant { Map = vec {
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" },
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f..." }
    }}},
    record { "reason"; variant { Text = "Violation of terms" }}
  }}}
}};
```

### Unfreeze Account Block

```motoko
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeaccount" }},
  record { "ts"; variant { Nat = 1_741_312_737_184_874_392 }},
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }},
  record { "tx"; variant { Map = vec {
    record { "account"; variant { Array = vec {
      variant { Blob = blob "\00\00\00\00\02\00\01\0d\01\01" },
      variant { Blob = blob "\06\ec\cd\3a\97\fb\a8\5f..." }
    }}},
    record { "reason"; variant { Text = "Cleared by compliance team" }}
  }}}
}};
```

### Freeze Principal Block

```motoko
variant { Map = vec {
  record { "btype"; variant { Text = "123freezeprincipal" }},
  record { "ts"; variant { Nat = 1_741_312_737_184_874_392 }},
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }},
  record { "tx"; variant { Map = vec {
    record { "principal"; variant { Blob = blob "\94\85\a4\06..." }},
    record { "reason"; variant { Text = "Violation of platform policy" }}
  }}}
}};
```

### Unfreeze Principal Block

```motoko
variant { Map = vec {
  record { "btype"; variant { Text = "123unfreezeprincipal" }},
  record { "ts"; variant { Nat = 1_741_312_737_184_874_392 }},
  record { "phash"; variant { Blob = blob "\d5\c7\eb\57..." }},
  record { "tx"; variant { Map = vec {
    record { "principal"; variant { Blob = blob "\94\85\a4\06..." }},
    record { "reason"; variant { Text = "Review complete, reinstated" }}
  }}}
}};
```
