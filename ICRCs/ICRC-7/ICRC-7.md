|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai)|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Base Non-Fungible Token (NFT) Standard

The ICRC-7 is a standard for the base implementaion of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

## Data

### account

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the _default account_ of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount; };
```

The canonical textual representation of the account follows the definition in ICRC-1.

## Methods

### icrc7_collection_metadata
Returns all the collection-level metadata of the NFT collection in a single query.
```candid "Methods" +=
icrc7_collection_metadata : () -> record { 
  icrc7_name : text; 
  icrc7_symbol : text;
  icrc7_royalties : opt nat16; 
  icrc7_royalty_recipient : opt Account;
  icrc7_description : opt text;
  icrc7_image : opt text;  // The URL of the token logo. The value can contain the actual image if it's a Data URL.
  icrc7_total_supply : nat;
  icrc7_supply_cap : opt nat;
} query;
```
### icrc7_name

Returns the name of the NFT collection (e.g., `My Super NFT`).

```candid "Methods" +=
icrc7_name : () -> (text) query;
```

### icrc7_symbol

Returns the symbol of the collection (e.g., `MS`).

```candid "Methods" +=
icrc7_symbol : () -> (text) query;
```

### icrc7_royalties

Returns the default royalty percentage in bps (i.e 150 means 1.50%). Note that 
only one royalty can be specified. For more complex use cases please consider using a fee splitter.

```candid "Methods" +=
icrc7_royalties : () -> (opt nat16) query;
```

### icrc7_royalty_recipient

Returns the default royalty percentage. Note that only one royalty recipient can be specified. For 
more complex use cases please consider using a fee splitter. The account specified must be able to
handle arbitrary ICRC-1 tokens as royalties might be paid in any token.

```candid "Methods" +=
icrc7_royalty_recipient : () -> (opt Account) query;
```

### icrc7_description

Returns the text description of the collection.

```candid "Methods" +=
icrc7_description : () -> (opt text) query;
```

### icrc7_image

Returns a link to the image of the collection. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the image itself.

```candid "Methods" +=
icrc7_image : () -> (opt text) query;
```

### icrc7_total_supply

Returns the total number of NFTs on all accounts.

```candid "Methods" +=
icrc7_total_supply : () -> (nat) query;
```

### icrc7_supply_cap

Returns the maximum number of NFTs possible for this collection. Any attempt to mint more NFTs 
than this supply cap shall be rejected.

```candid "Methods" +=
icrc7_supply_cap : () -> (opt nat) query;
```

### icrc7_metadata

Returns the token metadata for a particular tokenId.

```candid "Type definitions" +=
type Metadata = variant { Nat : nat; Int : int; Text : text; Blob : blob };
```

```candid "Methods" +=
icrc7_metadata : (nat) -> (vec record { text; Metadata }) query;
```

### icrc7_owner_of

Returns the owner of a tokenId.

```candid "Methods" +=
icrc7_owner_of : (nat) -> (Account) query;
```

### icrc7_balance_of

Returns the balance of the account given as an argument.

```candid "Methods" +=
icrc7_balance_of : (Account) -> (nat) query;
```

### icrc7_tokens_of

Returns the list of tokenIds of the account given as an argument.

```candid "Methods" +=
icrc7_tokens_of : (Account) -> (vec nat) query;
```

### icrc7_transfer

Transfers one or more tokens from account to the `to` account.

```candid "Type definitions" +=
type TransferArgs = record {
    spender_subaccount: opt blob; // the subaccount of the caller (used to identify the spender)
    from : Account;
    to : Account;
    token_ids : vec nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
    is_atomic : opt bool;
};

type TransferError = variant {
    Unauthorized: record { token_ids : vec nat };
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

If a tokenId doesn't exist or if the caller principal is not permitted to act on the tokenId, then the 
tokenId would be added to the `Unauthorized` list. If `is_atomic` is true (default), then the transfer of tokens in the `token_ids` list must all succeed or all fail. 

The `memo` parameter is an arbitrary blob that has no meaning to the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction_deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

The result is either the transaction index of the transfer or an error.

### icrc7_approve

```candid "Type definitions" +=
type ApprovalArgs = record {
    from_subaccount : opt blob;
    spender : Account;    // Approval is given to an ICRC Account
    token_ids : opt vec nat;            // TBD: change into variant?
    expires_at : opt nat64;
    memo : opt blob;
    created_at_time : opt nat64; 
};

type ApprovalError = variant {
    Unauthorized : vec nat;
    TooOld;
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_approve : (ApprovalArgs) -> (variant { Ok: nat; Err: ApprovalError; });
```
### icrc7_approve
TBD

### icrc7_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc7_supported_standards : () -> (vec record { name : text; url : text }) query;
```

The result of the call should always have at least one entry,

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7" }
```

## Extensions <span id="extensions"></span>

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  - Reliable transaction notifications for smart contracts.
  - The block structure and the interface for fetching blocks.
  - Pre-signed transactions.

The standard defines the `icrc7_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-42"` or `"DIP-20"`) implemented by the ledger.





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


<!--
```candid ICRC-1.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->
