# `ICRC‑152`: Privileged Mint & Burn API

| Status |
|:------:|
| Draft  |

## Introduction & Motivation

Most ICRC-based ledgers (e.g., ICRC-1, ICRC-2) treat minting and burning as
system-only operations that occur indirectly.

- Minting is represented as a transfer from the minting account.
- Burning is represented as a transfer into the minting account.

This approach works for decentralized tokens but is insufficient for
administrator-controlled or regulated assets, such as:

- Stablecoins, which must allow an issuer to adjust circulating supply.
- Real-World Asset (RWA) tokens, where compliance requires explicit
  mint and burn actions.
- Other managed ledgers (NFTs, reward points, internal credits).

Without a standardized API, integrators and tooling cannot reliably
distinguish user-initiated supply changes from privileged supply
management actions.

ICRC-152 addresses this gap by introducing:

- Two privileged methods, `icrc152_mint` and `icrc152_burn`, callable only by authorized principals.
- A canonical mapping from method inputs to block `tx` fields, including optional metadata and caller tracking.
- Recording of these operations using the typed block kinds **defined in ICRC-122**:
  `btype = "122mint"` for mints and `btype = "122burn"` for burns.


## Overview

ICRC-152 standardizes privileged supply management in ICRC-based ledgers.

Specifically, it defines:

- **APIs** for minting and burning tokens under ledger authorization.
- **Canonical `tx` mapping** rules ensuring deterministic block content.
- **Use of ICRC-122 block kinds** to record privileged supply actions
  (`btype = "122mint"` and `btype = "122burn"`).
- **Compliance reporting** through ICRC-10 methods.
  


This allows wallets, explorers, and auditors to:

- Reliably distinguish privileged supply changes from user-initiated transfers.
- Verify compliance with external requirements (e.g., MiCA).
- Interoperate across ledgers that adopt the same API and block semantics.


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
- Populate `tx.op` with namespaced values **introduced by this standard**:
  `"152mint"` and `"152burn"`.





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
    Unauthorized : text;              // caller not permitted
    InvalidAccount : text;             // target account invalid
    Duplicate : record { duplicate_of : nat }; 
    GenericError : record { error_code : nat; message : text };
};

icrc152_mint : (MintArgs) -> (variant { Ok : nat; Err : MintError });
```


#### Semantics
- **Credits** `amount` tokens to the `to` account.  
- **Increases** the total supply by `amount`.  
- **Creates** a block of type `122mint`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the core transition already defined for `122mint` blocks.  

#### Return Values
On success, the method returns

- `variant { Ok : nat }`  
  where the `nat` is the index of the block created in the ledger.  

On failure, the method returns

- `variant { Err : MintError }`  
  describing the reason for rejection.  



#### Canonical `tx` Mapping
A successful call to `icrc152_mint` produces a block of type `122mint`.  
The `tx` field is derived deterministically as follows:

- `op     = "152mint"`  
- `to     = MintArgs.to`  
- `amt    = MintArgs.amount`  
- `ts     = MintArgs.created_at_time`  
- `caller = caller_principal (as Blob)`  
- `reason = MintArgs.reason` (if provided)  

Optional fields (`reason`) MUST be omitted from `tx` if not supplied in the call.  



### `icrc152_burn`

This method allows a ledger controller (or other authorized principal) to burn
tokens. It debits the specified account, decreases total supply, and records
the action on-chain.

```
type BurnArgs = record {
  from            : Account;
  amount          : nat;
  created_at_time : nat64;
  reason          : opt text;
};

type BurnError = variant {
  Unauthorized : text;              // caller not permitted
  InvalidAccount : text;             // source account invalid
  InsufficientBalance : record { balance : nat };
  Duplicate : record { duplicate_of : nat }; 
  GenericError : record { error_code : nat; message : text };
};

icrc152_burn : (BurnArgs) -> (variant { Ok : nat; Err : BurnError });
```


#### Semantics
- **Debits** `amount` tokens from the `from` account.  
- **Decreases** the total supply by `amount`.  
- **Creates** a block of type `122burn`.  
- On success, returns the **index of the created block**.  
- On failure, returns an appropriate error.  
- Semantics are consistent with the core transition already defined for `122burn` blocks.  

#### Return Values
On success, the method returns

- `variant { Ok : nat }`  
  where the `nat` is the index of the block created in the ledger.  

On failure, the method returns

- `variant { Err : BurnError }`  
  describing the reason for rejection.  


#### Canonical `tx` Mapping
A successful call to `icrc152_burn` produces a block of type `122burn`.  
The `tx` field is derived deterministically as follows:

- `op     = "152burn"`  
- `from   = BurnArgs.from`  
- `amt    = BurnArgs.amount`  
- `ts     = BurnArgs.created_at_time`  
- `caller = caller_principal (as Blob)`  
- `reason = BurnArgs.reason` (if provided)  

Optional fields (`reason`) MUST be omitted from `tx` if not supplied in the call.  

### Reporting Compliance

#### Supported Standards

Ledgers implementing ICRC-152 MUST indicate compliance through the
`icrc1_supported_standards` and `icrc10_supported_standards` methods, by
including in the output of these methods:

```
variant { Vec = vec {
    record {
        "name"; variant { Text = "ICRC-152" };
        "url";  variant { Text = "https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-152.md" };
    }
}};
```

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
  to              = [principal "f5288412af11b299313a5b5a7c128311de102333c4adbe669f2ea1a308"];
  amount          = 500_000;
  created_at_time = 1_753_344_737_123_456_789 : nat64;
  reason          = ?"Initial allocation";
})
```
Here, `to` is the account of the recipient, `amount` is the number of tokens to mint, `created_at_time` is the caller‑supplied timestamp, and `reason` provides an optional human‑readable note.

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
    record { "ts"; variant { Nat64 = 1_753_344_738_000_000_000 : nat64 } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "op";     variant { Text  = "152mint" } };
          record {
            "to";
            variant {
              Array = vec {
                variant { Blob = blob "\15\28\84\12\af\11\b2\99\31\3a\5b\5a\7c\12\83\11\de\10\23\33\c4\ad\be\66\9f\2e\a1\a3\08" }
              }
            };
          };
          record { "amt";    variant { Nat64 = 500_000 : nat64 } };
          record { "ts";     variant { Nat64 = 1_753_344_737_123_456_789 : nat64 } };
          record { "caller"; variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text  = "Initial allocation" } };
        }
      };
    };
  }
};

```



The block records the operation (`op = "152mint"`), the recipient account (`to`), 
the minted amount (`amt`), the timestamp supplied by the caller (`ts`), the caller’s 
principal (`caller`), and the optional `reason`.  

In the method call example above, the recipient is shown as a human-readable 
principal literal. In the resulting block, the same principal is encoded canonically 
as `variant { Blob = ... }`, where the `Blob` contains the raw byte representation 
of the principal as defined by ICRC-3. Both forms represent the same identity; 
the difference is only in presentation.  


### Example burn call and resulting block

#### Call
The caller invokes
```
icrc152_burn({
  from            = [principal "abcd0123456789abcdef0123456789abcdef0123456789abcdef0123456789"];
  amount          = 200_000;
  created_at_time = 1_753_344_740_000_000_000 : nat64;
  reason          = ?"Burn to reduce supply";
})

```
Here, `from` is the account to be debited, `amount` is the number of tokens to burn, `created_at_time` is the caller‑supplied timestamp, and `reason` provides an optional human‑readable note.

#### Resulting block

This call results in a block with btype = "122burn" and the following contents:

```
variant {
  Map = vec {
    record { "btype"; variant { Text = "122burn" } };
    record {
      "phash";
      variant {
        Blob = blob "\7f\89\42\a5\be\4d\af\50\3b\6e\2a\8e\9c\c7\dd\f1\c9\e8\24\f0\98\bb\d7\af\ae\d2\90\10\67\df\1e\c1\0a"
      };
    };
    record { "ts"; variant { Nat64 = 1_753_344_740_500_000_000 : nat64 } };
    record {
      "tx";
      variant {
        Map = vec {
          record { "op";     variant { Text  = "152burn" } };
          record {
            "from";
            variant {
              Array = vec {
                variant { Blob = blob "\ab\cd\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab" }
              }
            };
          };
          record { "amt";    variant { Nat64 = 200_000 : nat64 } };
          record { "ts";     variant { Nat64 = 1_753_344_740_000_000_000 : nat64 } };
          record { "caller"; variant { Blob  = blob "\00\00\00\00\02\30\02\17\01\01" } };
          record { "reason"; variant { Text  = "Burn to reduce supply" } };
        }
      };
    };
  }
};


```

Here, the block records the operation (`op = "152burn"`), the account being debited (`from`), 
the burned amount (`amt`), the caller-provided timestamp (`ts`), the caller’s principal (`caller`), 
and the optional `reason`.  

In the method call example, the account is shown as a principal literal. In the resulting block, 
the same principal is encoded canonically as `variant { Blob = ... }`. These two forms 
represent the same identity; the difference is only in presentation.  

