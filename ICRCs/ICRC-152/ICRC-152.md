# `ICRC‑152`: Privileged Mint & Burn API

| Status |
|:------:|
| Draft  |

`ICRC‑152` defines privileged minting and burning operations for ICRC‑compliant ledgers. It specifies two methods—`icrc152_mint` and `icrc152_burn`—and describes the canonical block representation of each call according to `ICRC‑3`. The methods map to `122mint` and `122burn` block types, while the `tx.op` field is namespaced with `152`. Each `tx` also records the caller’s principal and an optional human‑readable reason.



## Type Definitions


```
type MintArgs = record {
  to              : Account;    // target account receiving the minted tokens
  amount          : nat;        // number of tokens to mint
  created_at_time : nat64;      // timestamp in nanoseconds since Unix epoch
  reason          : opt text;   // optional human-readable reason for the mint
};

type BurnArgs = record {
  from            : Account;    // account from which tokens are burned
  amount          : nat;        // number of tokens to burn
  created_at_time : nat64;      // timestamp in nanoseconds since Unix epoch
  reason          : opt text;   // optional human-readable reason for the burn
};

type MintError = variant {
  Unauthorized;
  InvalidAccount;
  CreatedInFuture;
  Duplicate;
  TemporarilyUnavailable;
  Other : text;
};

type BurnError = variant {
  Unauthorized;
  InsufficientBalance;
  InvalidAccount;
  CreatedInFuture;
  Duplicate;
  TemporarilyUnavailable;
  Other : text;
};
```


## Method Signatures

```
icrc152_mint : (MintArgs) -> (variant { Ok : nat; Err : MintError });

icrc152_burn : (BurnArgs) -> (variant { Ok : nat; Err : BurnError });
```

## Semantics

- **Authorisation:** The ledger MUST verify that the caller is authorised to mint or burn. If not, it returns `Err.Unauthorized`.
- **Supply update:**  
  - For `icrc152_mint`, increase the total supply by `amount` and credit `amount` to the `to` account.  
  - For `icrc152_burn`, decrease the total supply by `amount` and debit `amount` from the `from` account. If the `from` account does not hold enough tokens, the ledger MUST return `Err.InsufficientBalance`.
- **Fees:** Neither operation charges a fee under this specification.
- **Return value:** On success, return the index of the newly appended block as `Ok(nat)`.
- **Caller field:** The `caller` field in the `tx` record MUST contain the principal of the caller encoded as a blob. This field is informational and does not affect the state transition.


## Canonical `tx` Mapping for Successful Calls

### Mint block (btype = "122mint")

```
tx.op     = "152mint"
tx.to     = MintArgs.to
tx.amt    = MintArgs.amount
tx.ts     = MintArgs.created_at_time
tx.caller = caller_principal (as Blob)
tx.reason = MintArgs.reason (if provided)
```

### Burn block (btype = "122burn")
```
tx.op     = "152burn"
tx.from   = BurnArgs.from
tx.amt    = BurnArgs.amount
tx.ts     = BurnArgs.created_at_time
tx.caller = caller_principal (as Blob)
tx.reason = BurnArgs.reason (if provided)
```


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

The block records the operation (`op = "152mint"`), the recipient account (`to`), the minted amount (`amt`), the timestamp supplied by the caller (`ts`), the caller’s principal (`caller`) and the optional `reason`.

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
Here, `from` s the account to be debited,, `amount` is the number of tokens to burn, `created_at_time` is the caller‑supplied timestamp, and `reason` provides an optional human‑readable note.

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
                variant { Blob = blob  
                "\ab\cd\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab\cd\ef\01\23\45\67\89\ab" }
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

Here, the block records the operation (`op = "152burn"`), the account being debited (`from`), the burned amount (`amt`), the caller‑provided timestamp (`ts`), the caller’s principal (`caller`) and the optional `reason`. 




## Supported block types (for `icrc3_supported_block_types`)
```
vec {
  record { block_type = "122mint"; url = "https://example.com/ICRC-122.md#122mint" },
  record { block_type = "122burn"; url = "https://example.com/ICRC-122.md#122burn" },
}


```

