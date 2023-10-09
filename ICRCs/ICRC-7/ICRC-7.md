|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai)|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Base Non-Fungible Token (NFT) Standard

// Open issues

// - specify error codes

// - Royalties: there were suggestions to remove the royalties elements entirely

The ICRC-7 is the base standard for the implementation of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

## Data

### account

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the _default account_ of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount; };
```

The canonical textual representation of the account follows the definition in [ICRC-1](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md). Note that ICRC-7 accounts have the same structure and follow the same overall principles as ICRC-1 accounts.

### Token identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids.

## Methods

### Conventions

Unless specified explicitly otherwise, the ordering of response elements for batch calls is not defined, i.e., is arbitrary. The methods, both queries and update calls, that operate on token ids are batch methods that can receive vectors of token ids as input. The output of such a method is a vector with records comprising a token id and a response belonging to this token id. The ordering of the response elements is undefined. This API pattern does not require the caller to correlate the response elements to the input, but the response is self contained. For batch calls, the lenght of the output vector may be shorter than that of the input vector, e.g., in case of duplicate token ids in the input and a deduplication being performed by the method's implementation logic.

### icrc7_metadata

Returns the generic metadata for this ledger. The data model for metadata is based on the generic `Value` type, which allows for encoding arbitrarily complex data.

The elements contained in the metadata depends on the implementation of the ledger. Some examples of elements follow in the list below:
  * `max_approvals_per_token`: The maximum number of active approvals this ledger implementation allows per token.
  * `max_update_batch_size`: The maximum batch size for update batch calls this ledger implementation supports.

// FIX anything else we can think of already now?

```candid "Type definitions" +=
// Generic value in accordance with ICRC-3
type Value = variant { 
    Blob : blob; 
    Text : text; 
    Nat : nat;
    Int : int;
    Array : vec Value; 
    Map : vec record { text; Value }; 
};
type Metadata = Value;
```

```candid "Methods" +=
icrc7_metadata : () -> (metadata : vec Metadata) query;
```

// FIX: the Value and Metadata are currently defined in multiple places

### icrc7_collection_metadata

Returns all the collection-level metadata of the NFT collection in a single query.

// FIX: There are discussions on moving this into the generic representation as in ICRC-1. One issue here is that this would mean that the royalties account would need to be encoded as blob, which is harder to handle. In this case, the below attributes would be explicitly provided as required or optional attributes.

```candid "Methods" +=
icrc7_collection_metadata : () -> record { 
  icrc7_name : text; 
  icrc7_symbol : text;
  icrc7_royalties : opt nat16; 
  icrc7_royalty_recipient : opt Account;
  icrc7_description : opt text;
  icrc7_logo : opt text;  // The URL of the token logo. The value can contain the actual image if it's a Data URL.
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

### icrc7_suggested_royalties

Returns the default royalty percentage in bps (i.e. 150 means 1.50%). Note that 
only one royalty can be specified. For more complex use cases please consider using a fee splitter.

If the royalties field is `null``, royalties are either unspecified or specified through an extension standard.

// FIX: there are opinions that the royalties-related methods should be removed and delegated to a specific standard (analogous to how ERC-2981 handles royalties for ERC-721); this needs to be resolved still in the WG; another option is to keep the methods in here for the simple case when this is sufficient and delegate only more complex requirements to a separate standard; this would have the advantage that simple royalties mechanisms could already be expressed with this base standard without an extension; as the royalties-related fields are optional, the fields add value, but do not constrain future extensibility using a different standard; recommendation to leave them in and leave them empty in case another standard is to be applied

```candid "Methods" +=
icrc7_suggested_royalties : () -> (opt nat16) query;
```

### icrc7_royalty_recipient

Returns the default royalty percentage. Note that only one royalty recipient can be specified. For 
more complex use cases please consider using a fee splitter. The account specified must be able to
handle arbitrary ICRC-1 tokens as royalties might be paid in any token.

If the royalties recipient is `null`, royalties are either unspecified or specified through an extension standard.

```candid "Methods" +=
icrc7_royalty_recipient : () -> (opt Account) query;
```

### icrc7_description

Returns the text description of the collection.

```candid "Methods" +=
icrc7_description : () -> (opt text) query;
```

### icrc7_logo

Returns a link to the logo of the collection. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself.

```candid "Methods" +=
icrc7_logo : () -> (opt text) query;
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

### icrc7_token_metadata

Returns the token metadata for `token_ids`, a list of token ids. Each element of the response vector comprises a `token_id` and the `metadata` corresponding to this token.

```candid "Type definitions" +=
// Generic value in accordance with ICRC-3
type Value = variant { 
    Blob : blob; 
    Text : text; 
    Nat : nat;
    Int : int;
    Array : vec Value; 
    Map : vec record { text; Value }; 
};
type Metadata = Value;
```

```candid "Methods" +=
icrc7_token_metadata : (token_ids : vec nat) -> (vec record { token_id : nat; metadata : Metadata; }) query;
```

### icrc7_owner_of

Returns the owner of a list of tokens identified by `token_ids`. For non-existing token ids, the `null` value is returned for the `account`. The ordering of the response elements is arbitrary.

```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat) -> (vec record { token_id : nat; account : opt Account; }) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument. For a non-existing account, the `null` value is returned.

```candid "Methods" +=
icrc7_balance_of : (account : Account) -> (balance : opt nat) query;
```

### icrc7_tokens

Returns the list of tokens in this ledger, sorted by their token id. The result is paginated and pagination is controlled via the `skip` and `take` parameters: The response to a request results in at most `take` many token ids, starting with the smallest id following `skip`. The token ids in the response are sorted in ascending order. If `take` is omitted, a reasonable value is assumed.

For retrieving all tokens of the ledger, the pagination API is used such that the first call leaves `skip` empty and specifies a suitable `take` value, then the method is called repeatedly such that the greatest token id of the previous response is used as `skip` value for the next call to the method. This way all tokens can be enumerated in ascending order.

Each invocation is executed on the current memory state of the ledger.

// take is now optional

```candid "Methods" +=
icrc7_tokens : (skip : opt nat, take : opt nat32) -> (token_ids : vec nat) query;
```

### icrc7_tokens_of

Returns a vector of `token_id`s of all tokens held by `account`, sorted by their token id. The result is paginated, the mechanics of pagination is the same as for `icrc7_tokens` using `skip` and `take` to control pagination.

```candid "Methods" +=
icrc7_tokens_of : (account : Account, skip : opt nat, take : opt nat32) -> (token_ids : vec nat) query;
```

### icrc7_approve

Entitles a `spender`, indicated through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount; }`, where `caller` is the caller of this method (and the token owner principal) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens can be transferred out from). The call resets the expiration date, memo, and creation timestamp for the approval to the specified values in case an approval for the `spender` and `from_subaccount` already exists for a token. The parameter `tokens` can either specify the approval to apply for the whole collection (`Collection` variant) or for a list of tokens (`TokenIds` variant).

The ledger SHOULD reject the call if the spender account owner is equal to the caller account owner.

An approval that has been created, is not expired, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initate a transfer.

In accordance with ICRC-2, multiple approvals can exist for the same `token_id` but different `spender`s and `subaccount`s. For the same token (or collection), spender, and subaccount triple a new approval shall always overwrite the old one. The ledger should limit the number of approvals that can be active per token. Such limit is exposed as ledger metadata through the metadata attribute `max_approvals_per_token`.

In case of `tokens` being of the `Collection` variant, the response contains only a single element with a `null` `token_id`, the `approval_response` indicating the success with an `Ok` variant with the transaction index, or error `Err` of the collection-level approval. In case of `tokens` being a vector of token ids, the response is a vector comprising records with a `token_id` as first element and an `Ok` variant with the transaction index for the success case or an `Err` variant indicating an error as second element.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. Of course, all historic approvals are contained in the block history the ledger creates.

```candid "Type definitions" +=
type ApprovalArgs = record {
    from_subaccount : opt blob;
    spender : Account;    // Approval is given to an ICRC Account
    tokens : variant { Collection; TokenIds : vec nat }; 
    expires_at : opt nat64;
    memo : opt blob;
    created_at_time : opt nat64; 
};

type ApprovalError = variant {  // TO REVIEW
    Unauthorized;
    TooOld;
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text; };
};
```

```candid "Methods" +=
icrc7_approve : (ApprovalArgs) -> (vec record { token_id : opt nat; approval_response : variant { Ok : nat; Err : ApprovalError; } } );
```

### icrc7_transfer

Transfers one or more tokens from the `from` account to the `to` account. The transfer can be initiated either by the holder of the tokens or a party that has been authorized by the holder to execute transfers using `icrc7_approve`. The `spender_subaccount` is used to identify the spender. The spender is an account comprised of the principal calling this method and the parameter `spender_subaccount`.

The response is a vector of records each comprising a `token_id` and a corresponding `transfer_result` indicating success or error. In the success case, the `Ok` variant indicates the transaction index of the transfer, in the error case, the `Err` variant indicates the error through `TransferError`.

A transfer clears all approvals for the successfully transferred tokens. This implicit approval clearing only clears token-id-based approvals and never touches collection-level approvals.

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
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (TransferArgs) -> (vec record { token_id : nat; transfer_result : variant { Ok : nat; Err : TransferError; }; });
```

If a token id doesn't exist or if the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response. If `is_atomic` is true (default), then the transfer of tokens in the `token_ids` vector must all succeed, otherwise no transfer must be executed.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction_deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction_deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

### icrc7_revoke_approval

Revokes the specified approvals from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked.

In case of `tokens` being of the `Collection` variant, the response vector contains only a single element with a transaction index indicating the success or an error of the collection-level approval revocation. In case of `tokens` being a vector of token ids, the response is a vector comprising records with a `token_id` and a variant with `Ok` containing a transaction index indicating the success case and an `Err` indicating the error case.

Revoking an approval for one or more token ids does not affect collection-level approvals. Revoking a collection-level approval does not affect approvals for individual token ids.

An ICRC-7 ledger implementation does not need to keep track of revoked approvals.

```candid "Type definitions" +=
type RevokeArgs = record {
    from_subaccount : opt blob;
    spender : opt Account;
    tokens : variant { Collection; TokenIds : vec nat; };
};

type RevokeError = variant {
    Unauthorized;
    ApprovalDoesNotExist;
    TooOld;
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text; };
};
```
```candid "Methods" +=
icrc7_revoke_approval: (RevokeArgs) -> (vec record { token_id : opt nat; revoke_response : variant { Ok : nat; Err : RevokeError; }; } );
```

### icrc7_revoke_all_approvals

Revokes all approvals applying to tokens of the sender. This includes collection-level approvals as well as approvals related to specific token ids.

The return value is the `Ok` variant in case all approvals could be successfully removed and an error `Err` otherwise. In case of success, the result is the number of revoked approvals, where each collection-level approval and each per-token-id approval count as an approval. For example, if two collection-level approvals and 3 token-id-level approvals are revoked, the `Ok` variant has value 5.

```candid "Type definitions" +=
type RevokeError = variant {
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text; };
};
```
```candid "Methods" +=
icrc7_revoke_all_approvals: () -> (variant { Ok: nat; Err: RevokeError; });
```

### icrc7_get_approvals

Returns the approvals that exist for the given vector of `token_ids`.

The response is a vector the elements of which comprise a `token_id` and a vector of approval records for the token with this id. The ordering of the elements of the response vector and the nested vector of approval records is both undefined and the size of the reponse vector is at most that of the `token_ids` input parameter. The size can be shorter, for example, if the input contains duplicate elements.

```candid "Methods" +=
icrc7_get_approvals: (token_ids : vec nat)
    -> (vec record { token_id : nat; approvals : vec record { token_id : nat; spender : Account; expires_at : opt nat64; created_at_time : opt nat64; } });
```

// FIX from_subaccount and memo fields missing?

Note: As ledgers are recommended to limit the number of approvals per token, pagination is not required for this method.
// FIX check this with the working group

### icrc7_get_collection_approvals

Returns all collection-level approvals that exist for the specified `owner`.

The response is a vector of approval records.

```candid "Methods" +=
icrc7_get_collection_approvals : (owner : Account)
    -> (vec record { spender : Account; expires_at : opt nat64; created_at_time : opt nat64; });
```

// FIX as above

As ledgers are recommended to limit the number of approvals per token, pagination is not required for this method.
// FIX check this with the group

### icrc7_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc7_supported_standards : () -> (vec record { name : text; url : text; }) query;
```

The result of the call should always have at least one entry,

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7"; }
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
