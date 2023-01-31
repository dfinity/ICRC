|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai)|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Minimal Non-Fungible Token (NFT) Standard

The ICRC-7 is a standard for the minimal implementaion of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).


## Methods

### icrc7_collection_metadata
Returns all the collection-level metadata of the NFT collection in a single query.
```candid "Methods" +=
icrc7_collection_metadata : () -> record { 
  icrc1_name : text; 
  icrc7_symbol : text;
  icrcX_royalties : opt nat; 
  icrcX_royalty_recipient : opt principal;
  icrcX_description : opt text;
  icrcX_image : opt blob;
  icrcX_total_supply : nat;
} query;
```



### icrc7_metadata <span id="metadata_method"></span>

Returns the token metadata for a particular tokenId.

```candid "Type definitions" +=
type Metadata = variant { Nat : nat; Int : int; Text : text; Blob : blob };
```

```candid "Methods" +=
icrc7_metadata : (nat) -> (vec record { text; Metadata }) query;
```


### icrc7_minting_account

Returns the [minting account](#minting_account) if this ledger supports minting tokens. Sending an NFT to the minting account burns it.

```candid "Methods" +=
icrc7_minting_account : () -> (opt principal) query;
```

### icrc7_balance_of

Returns the balance of the account given as an argument.

```candid "Methods" +=
icrc7_balance_of : (principal) -> (vec nat) query;
```

### icrc7_tokens_of

Returns the list of tokenIds of the account given as an argument.

```candid "Methods" +=
icrc7_balance_of : (principal) -> (nat) query;
```

### icrc7_balance_of

Returns the balance of the account given as an argument.

```candid "Methods" +=
icrc7_balance_of : (Account) -> (nat) query;
```

### icrc7_transfer <span id="transfer_method"></span>

Transfers `amount` of tokens from account `record { of = caller; subaccount = from_subaccount }` to the `to` account.
The caller pays `fee` tokens for the transfer.

```candid "Type definitions" +=
type TransferArgs = record {
    from_subaccount : opt Subaccount;
    to : Account;
    amount : nat;
    fee : opt nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferError = variant {
    BadFee : record { expected_fee : nat };
    BadBurn : record { min_burn_amount : nat };
    InsufficientFunds : record { balance : nat };
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (TransferArgs) -> (variant { Ok: nat; Err: TransferError; });
```

The caller pays the `fee`.
If the caller does not set the `fee` argument, the ledger applies the default transfer fee.
If the `fee` argument does not agree with the ledger fee, the ledger MUST return `variant { BadFee = record { expected_fee = ... } }` error.

The `memo` parameter is an arbitrary blob that has no meaning to the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction_deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

The result is either the transaction index of the transfer or an error.

### icrc7_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc7_supported_standards : () -> (vec record { name : text; url : text }) query;
```

The result of the call should always have at least one entry,

```candid
record { name = "ICRC-1"; url = "https://github.com/dfinity/ICRC-1" }
```

## Extensions <span id="extensions"></span>

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  - Reliable transaction notifications for smart contracts.
  - The block structure and the interface for fetching blocks.
  - Pre-signed transactions.

The standard defines the `icrc7_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-42"` or `"DIP-20"`) implemented by the ledger.

## Metadata

A ledger can expose metadata to simplify integration with wallets and improve user experience.
The client can use the [`icrc7_metadata`](#metadata_method) method to fetch the metadata entries. 
All the metadata entries are optional.

### Key format

The metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons.
Namespace `icrc7` is reserved for keys defined in this standard.

### Standard metadata entries
| Key | Semantics | Example value
| --- | ------------- | --------- |
| `icrc7:symbol` | The token currency code (see [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)). When present, should be the same as the result of the [`icrc7_symbol`](#symbol_method) query call. | `variant { Text = "XTKN" }` | 
| `icrc7:name` | The name of the token. When present, should be the same as the result of the [`icrc7_name`](#name_method) query call. | `variant { Text = "Test Token" }` | 
| `icrc7:decimals` |  The number of decimals the token uses. For example, 8 means to divide the token amount by 10<sup>8</sup> to get its user representation. When present, should be the same as the result of the [`icrc7_decimals`](#decimals_method) query call. | `variant { Nat = 8 }` |
| `icrc7:fee` | The default transfer fee. When present, should be the same as the result of the [`icrc7_fee`](#fee_method) query call. |  `variant { Nat = 10_000 }` |
| `icrc7:logo` | The URL of the token logo. The value can contain the actual image if it's a [Data URL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs).  | `variant { Text = "data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMSIgaGVpZ2h0PSIxIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxyZWN0IHdpZHRoPSIxMDAlIiBoZWlnaHQ9IjEwMCUiIGZpbGw9InJlZCIvPjwvc3ZnPg==" }` | 


## Transaction deduplication <span id="transfer_deduplication"></span>

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-1 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-1 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer`](#transfer_method) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client did not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

## Minting account <span id="minting_account"></span>

The minting account is a unique account that can create new tokens and acts as the receiver of burnt tokens.

Transfers _from_ the minting account act as _mint_ transactions depositing fresh tokens on the destination account.
Mint transactions have no fee.

Transfers _to_ the minting account act as _burn_ transactions, removing tokens from the token supply.
Burn transactions have no fee but might have minimal burn amount requirements.
If the client tries to burn an amount that is too small, the ledger SHOULD reply with

```
variant { Err = variant { BadBurn = record { min_burn_amount = ... } } }
```

The minting account is also the receiver of the fees burnt in regular transfers.

## Textual representation of accounts

We specify a _canonical textual format_ that all applications should use to display ICRC-1 accounts.
This format relies on the textual encoding of principals specified in the [Internet Computer Interface Specification](https://internetcomputer.org/docs/current/references/ic-interface-spec/#textual-ids), referred to as `Principal.toText` and `Principal.fromText` below.
The format has the following desirable properties:

1. A textual encoding of any non-reserved principal is a valid textual encoding of the default account of that principal on the ledger.
2. The decoding function is injective (i.e., different valid encodings correspond to different accounts).
   This property enables applications to use text representation as a key.
3. A typo in the textual encoding invalidates it with a high probability.

### Encoding

Applications SHOULD encode accounts as follows:

  1. The encoding of the default account (the subaccount is null or a blob with 32 zeros) is the encoding of the owner principal.
  2. The encoding of accounts with a non-default subaccount is the textual principal encoding of the concatenation of the owner principal bytes, the subaccount bytes with the leading zeros omitted, the length of the subaccount without the leading zeros (a single byte), and an extra byte `7F`<sub>16</sub>.

In pseudocode:

```sml
encodeAccount({ owner; subaccount }) = case subaccount of
  | None ⇒ Principal.toText(owner)
  | Some([32; 0]) ⇒ Principal.toText(owner)
  | Some(bytes) ⇒ Principal.toText(owner · shrink(bytes) · [|shrink(bytes)|, 0x7f])

shrink(bytes) = case bytes of
  | 0x00 :: rest ⇒ shrink(rest)
  | bytes ⇒ bytes
```

### Decoding

Applications SHOULD decode textual representation as follows:

  1. Decode the text as if it was a principal into `raw_bytes`, ignoring the principal length check (some decoders allow the principal to be at most 29 bytes long).
  2. If `raw_bytes` do not end with byte `7F`<sub>16</sub>, return an account with `raw_bytes` as the owner and an empty subaccount.
  3. If `raw_bytes` end with `7F`<sub>16</sub>:
     1. Drop the last `7F`<sub>16</sub> byte.
     2. Read the last byte `N` and drop it. If `N > 32` or `N = 0`, raise an error.
     3. Take the last N bytes and strip them from the input.
        If the first byte in the stripped sequence is zero, raise an error.
        Prepend the bytes with (32 - N) zeros on the left to get a 32-byte subaccount.
     4. Return an account with the owner being the rest of the input sequence as the owner and the subaccount being the byte array constructed in the previous step.

In pseudocode:

```sml
decodeAccount(text) = case Principal.fromText(text) of
  | (prefix · [n, 0x7f]) where Blob.size(prefix) < n ⇒ raise Error
  | (prefix · [n, 0x7f]) where n > 32 orelse n = 0 ⇒ raise Error
  | (prefix · suffix · [n, 0x7f]) where Blob.size(suffix) = n ⇒
    if suffix[0] = 0
    then raise Error
    else { owner = Principal.fromBlob(prefix); subaccount = Some(expand(suffix)) }
  | raw_bytes ⇒ { owner = Principal.fromBlob(raw_bytes); subaccount = None }

expand(bytes) = if Blob.size(bytes) < 32
                then expand(0x00 :: bytes)
                else bytes
```

<!--
```candid ICRC-1.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->
