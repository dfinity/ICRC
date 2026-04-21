# `ICRC‑152`: Privileged Mint & Burn API

| Status |
|:------:|
| Draft  |

## Introduction

Most ICRC-based ledgers (e.g., ICRC-1, ICRC-2) treat minting and burning as
system-only operations that occur indirectly: minting is a transfer *from* the
minting account; burning is a transfer *into* it.

This approach is insufficient for administrator-controlled or regulated assets
such as stablecoins, Real-World Asset (RWA) tokens, or managed ledgers (NFTs,
reward points, internal credits), where compliance requires explicit, auditable
supply management actions by an authorized principal.

ICRC-152 standardizes this by defining:

- **`icrc152_mint` and `icrc152_burn`** — privileged methods callable only by
  authorized principals.
- **Canonical `tx` mapping** — deterministic block content rules, including
  optional metadata and caller tracking.
- **ICRC-122 block kinds** — `btype = "122mint"` and `btype = "122burn"` to
  record privileged supply actions.
- **Compliance reporting** — via ICRC-10 methods.

This allows wallets, explorers, and auditors to reliably distinguish privileged
supply changes from user-initiated transfers, verify compliance with external
requirements (e.g., MiCA), and interoperate across ledgers that adopt this standard.


## Dependencies

This standard does not introduce new block kinds.

- **ICRC-3** — Provides the block log format, hashing, certification, and rules
  for canonical `tx` mapping.
- **ICRC-122** — Defines the typed block kinds used by this standard:
  - `btype = "122mint"` (authorized mint block)
  - `btype = "122burn"` (authorized burn block)

A ledger implementing ICRC-152 MUST:
- Emit `122mint` for successful `icrc152_mint` calls and `122burn` for successful
  `icrc152_burn` calls.
- Populate `tx.mthd` with namespaced values **introduced by this standard**:
  `"152mint"` and `"152burn"`.


## Common Elements

This standard inherits core conventions from **ICRC-3** (block log format, Value encoding, hashing, certification) and **ICRC-122** (typed block kinds).

- **Accounts**  
  Encoded as ICRC-3 `Value` `variant { Array = vec { V1 [, V2] } }` where  
  `V1 = variant { Blob = <owner_principal_bytes> }` and optionally  
  `V2 = variant { Blob = <32-byte_subaccount_bytes> }`.  
  If no subaccount is provided, the array contains only the owner principal.

- **Principals**  
  Encoded as `variant { Blob = <principal_bytes> }`.

- **Timestamps**  
  Caller-supplied `created_at_time` is in **nanoseconds since Unix epoch**.  
  Encoded as `Nat` in ICRC-3 `Value` and **MUST** fit in `nat64`.

- **Blocks & Parent Hash**  
  Resulting blocks for these APIs use `btype = "122mint"` and `btype = "122burn"` (per ICRC-122).  
  Standard metadata (e.g., `phash`, `ts`) follows ICRC-3.

- **`reason` field**  
  An optional human-readable explanation for the operation, encoded as `Text` in the block.  
  Unlike `memo` in ICRC-1/2, which carries machine-readable bytes (e.g., an invoice number),  
  `reason` is plain text intended for display in explorers and audit logs, with no structural  
  constraint on its value.



## Methods

### `icrc152_mint`


This method allows a ledger controller (or other authorized principal) to mint
tokens. It credits the specified account, increases total supply, and records
the action on-chain.


#### Arguments
```
type MintArgs = record {
  to              : Account;
  amount          : nat;
  created_at_time : nat64;
  reason          : opt text;
};

type MintError = variant {
    Unauthorized        : text;        // caller not permitted; text is for diagnostics only
    InvalidAccount      : text;        // target account invalid; text is for diagnostics only
    InvalidAmount       : text;        // amount must be greater than zero
    TooOld;
    CreatedInFuture     : record { ledger_time : nat64 };
    Duplicate           : record { duplicate_of : nat };
    GenericError        : record { error_code : nat; message : text };
};

icrc152_mint : (MintArgs) -> (variant { Ok : nat; Err : MintError });
```


#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- **Credit** `amount` to `to`.  
- **Increase** total supply by `amount`.  
- **Append** a block with `btype = "122mint"`.  
- The block’s top-level `tx` **MUST** be constructed **exactly** as in **Canonical `tx` Mapping** (keys, types, encodings), and embedded in the block.

**Return value**  
- On success, **MUST** return `variant { Ok : nat }` where the `nat` is the index of the created block.  
- On failure, **MUST** return `variant { Err : MintError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication (e.g., using `created_at_time` and any implementation-defined inputs).  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `InvalidAccount` — malformed/invalid target account (e.g., minting account, anonymous principal, invalid subaccount bytes).  
- `InvalidAmount` — `amount` is zero.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — an identical transaction was already accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid `122mint` block.

**Clarifications**  
- Optional fields **MUST be omitted** from `tx` if not supplied in the call (no null placeholders).  
- Representation-independent hashing (ICRC-3) applies; field presence and values matter, not map ordering.


#### Canonical `tx` Mapping (normative)
A successful call to `icrc152_mint` produces a block of type `122mint`.
The `tx` field is derived deterministically as follows:

| Field             | Type (ICRC-3 `Value`) | Source / Encoding Rule                                                     |
|-------------------|------------------------|-----------------------------------------------------------------------------|
| `mthd`            | `Text`                 | **Constant** `"152mint"`.                                                  |
| `to`              | `Array` (Account)      | From `MintArgs.to`, encoded as ICRC-3 Account.                             |
| `amt`             | `Nat`                  | From `MintArgs.amount`.                                                    |
| `ts`              | `Nat`                  | From `MintArgs.created_at_time` (ns since Unix epoch; **MUST** fit `nat64`). |
| `caller`          | `Blob`                 | Principal of the caller (raw bytes).                                       |
| `reason`          | `Text` *(optional)*    | From `MintArgs.reason` if provided; **omit** if absent.                    |




### `icrc152_burn`

This method allows a ledger controller (or other authorized principal) to burn
tokens. It debits the specified account, decreases total supply, and records
the action on-chain.

#### Arguments
```
type BurnArgs = record {
  from            : Account;
  amount          : nat;
  created_at_time : nat64;
  reason          : opt text;
};

type BurnError = variant {
  Unauthorized        : text;        // caller not permitted; text is for diagnostics only
  InvalidAccount      : text;        // source account invalid; text is for diagnostics only
  InvalidAmount       : text;        // amount must be greater than zero
  InsufficientBalance : record { balance : nat };
  TooOld;
  CreatedInFuture     : record { ledger_time : nat64 };
  Duplicate           : record { duplicate_of : nat };
  GenericError        : record { error_code : nat; message : text };
};

icrc152_burn : (BurnArgs) -> (variant { Ok : nat; Err : BurnError });
```


#### Semantics

**Authorization**  
- The method **MUST** be callable only by a **controller** of the ledger or other explicitly authorized principals.  
- Unauthorized calls **MUST** fail with `Unauthorized`.

**Effect (on success, non-retroactive)**  
- **Debit** `amount` from `from`.  
- **Decrease** total supply by `amount`.  
- **Append** a block with `btype = "122burn"`.  
- The block's top-level `tx` **MUST** be constructed **exactly** as in **Canonical `tx` Mapping** (keys, types, encodings), and embedded in the block.

**Return value**  
- On success, **MUST** return `variant { Ok : nat }` where the `nat` is the index of the created block.  
- On failure, **MUST** return `variant { Err : BurnError }`.

**Deduplication & idempotency**  
- The ledger **MUST** perform deduplication (e.g., using `created_at_time` and any implementation-defined inputs).  
- If a duplicate is detected, the ledger **MUST NOT** append a new block and **MUST** return `Err(Duplicate { duplicate_of = <index> })`.

**Error cases (normative)**  
- `Unauthorized` — caller not permitted.  
- `InvalidAccount` — malformed/invalid source account (e.g., anonymous principal, invalid subaccount bytes).  
- `InvalidAmount` — `amount` is zero.  
- `InsufficientBalance { balance }` — the `from` account holds fewer tokens than `amount`.  
- `TooOld` — `created_at_time` is before the ledger's deduplication window.  
- `CreatedInFuture { ledger_time }` — `created_at_time` is ahead of the ledger's current time.  
- `Duplicate { duplicate_of }` — an identical transaction was already accepted.  
- `GenericError { error_code, message }` — any other failure preventing a valid `122burn` block.

**Clarifications**  
- Optional fields **MUST be omitted** from `tx` if not supplied in the call (no null placeholders).  
- Representation-independent hashing (ICRC-3) applies; field presence and values matter, not map ordering.


#### Canonical `tx` Mapping (normative)
A successful call to `icrc152_burn` produces a block of type `122burn`.
The `tx` field is derived deterministically as follows:

| Field             | Type (ICRC-3 `Value`) | Source / Encoding Rule                                                      |
|-------------------|------------------------|-----------------------------------------------------------------------------|
| `mthd`            | `Text`                 | **Constant** `"152burn"`.                                                   |
| `from`            | `Array` (Account)      | From `BurnArgs.from`, encoded as ICRC-3 Account.                            |
| `amt`             | `Nat`                  | From `BurnArgs.amount`.                                                     |
| `ts`              | `Nat`                  | From `BurnArgs.created_at_time` (ns since Unix epoch; **MUST** fit `nat64`). |
| `caller`          | `Blob`                 | Principal of the caller (raw bytes).                                        |
| `reason`          | `Text` *(optional)*    | From `BurnArgs.reason` if provided; **omit** if absent.                     |

### Reporting Compliance

#### Supported Standards

Ledgers implementing ICRC-152 MUST indicate compliance through the
`icrc10_supported_standards` method by including in its output:

```
vec {
    record {
        name = "ICRC-152";
        url  = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-152/ICRC-152.md";
    }
}
```

Ledgers that also implement ICRC-1 MUST additionally include this entry in
the output of `icrc1_supported_standards`.

#### Supported Block Types

ICRC-152 extends ICRC-122 and does not introduce any new block kinds.
Accordingly, ledgers implementing ICRC-152 MUST already advertise support
for `122mint` and `122burn` as required by ICRC-122.
No additional block types need to be reported.

### Example mint call and resulting block

#### Call
The caller invokes
```
icrc152_mint({
  to              = record { owner = principal "ryjl3-tyaaa-aaaaa-aaaba-cai"; subaccount = null };
  amount          = 500_000;
  created_at_time = 1_753_344_737_123_456_789 : nat64;
  reason          = opt "Initial allocation";
})
```
Here, `to` is the Candid `Account` of the recipient (with `owner` as a principal and an optional `subaccount`), `amount` is the number of tokens to mint, `created_at_time` is the caller‑supplied timestamp, and `reason` provides an optional human‑readable note.

#### Resulting block

This call results in a block with btype = "122mint" and the following contents:

```
variant {
  Map = vec {
    record { "btype"; variant { Text = "122mint" } };
    record {
      "phash";
      variant {
        Blob = blob "\a0\5f\d2\f3\4c\26\73\58\00\7f\ea\02\18\43\47\70\85\50\2e\d2\1f\23\e0\dc\e6\af\3c\cf\9e\6f\4a\d8"
      };
    };
    record { "ts"; variant { Nat = 1_753_344_738_000_000_000 } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";   variant { Text  = "152mint" } };
          record {
            "to";
            variant {
              Array = vec {
                variant { Blob = blob "\00\00\00\00\00\00\00\02\01\01" }
              }
            };
          };
          record { "amt";    variant { Nat = 500_000 } };
          record { "ts";     variant { Nat = 1_753_344_737_123_456_789 } };
          record { "caller"; variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text  = "Initial allocation" } };
        }
      };
    };
  }
};

```



The block records the method (`mthd = "152mint"`), the recipient account (`to`),
the minted amount (`amt`), the timestamp supplied by the caller (`ts`), the caller’s
principal (`caller`), and the optional `reason`.  

In the method call example above, the recipient is specified as a Candid `Account`
record. In the resulting block, the account is encoded as an ICRC-3 `Array` containing
the principal's raw bytes as a `Blob`, as defined by ICRC-3.


### Example burn call and resulting block

#### Call
The caller invokes
```
icrc152_burn({
  from            = record { owner = principal "uf6dk-hyaaa-aaaaq-qaaaq-cai"; subaccount = null };
  amount          = 200_000;
  created_at_time = 1_753_344_740_000_000_000 : nat64;
  reason          = opt "Burn to reduce supply";
})
```
Here, `from` is the Candid `Account` to be debited (with `owner` as a principal and an optional `subaccount`), `amount` is the number of tokens to burn, `created_at_time` is the caller‑supplied timestamp, and `reason` provides an optional human‑readable note.

#### Resulting block

This call results in a block with btype = "122burn" and the following contents:

```
variant {
  Map = vec {
    record { "btype"; variant { Text = "122burn" } };
    record {
      "phash";
      variant {
        Blob = blob "\7f\89\42\a5\be\4d\af\50\3b\6e\2a\8e\9c\c7\dd\f1\c9\e8\24\f0\98\bb\d7\af\ae\d2\90\10\67\df\1e\c1"
      };
    };
    record { "ts"; variant { Nat = 1_753_344_740_500_000_000 } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "mthd";   variant { Text  = "152burn" } };
          record {
            "from";
            variant {
              Array = vec {
                variant { Blob = blob "\00\00\00\00\02\10\00\01\01\01" }
              }
            };
          };
          record { "amt";    variant { Nat = 200_000 } };
          record { "ts";     variant { Nat = 1_753_344_740_000_000_000 } };
          record { "caller"; variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text  = "Burn to reduce supply" } };
        }
      };
    };
  }
};


```

Here, the block records the method (`mthd = "152burn"`), the account being debited (`from`),
the burned amount (`amt`), the caller-provided timestamp (`ts`), the caller’s principal (`caller`),
and the optional `reason`.

The block contains two `ts` fields. The top-level `ts` (`1_753_344_740_500_000_000`) is the
ledger’s consensus timestamp, assigned when the block is committed to the log. The `tx.ts`
(`1_753_344_740_000_000_000`) is the caller-supplied `created_at_time` copied from `BurnArgs`.
These will generally differ.

In the method call example, the account is specified as a Candid `Account` record.
In the resulting block, the account is encoded as an ICRC-3 `Array` containing the
principal’s raw bytes as a `Blob`, as defined by ICRC-3.

