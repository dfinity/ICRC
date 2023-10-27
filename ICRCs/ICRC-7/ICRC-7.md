|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai)|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Minimal Non-Fungible Token (NFT) Standard

ICRC-7 is the minimal standard for the implementation of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

A token ledger implementation following this standard hosts an *NFT collection* (*collection*), a set of NFT tokens.

## Data

### Accounts

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore, an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the _default account_ of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount };
```

The canonical textual representation of the account follows the [definition in ICRC-1](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md). ICRC-7 accounts have the same structure and follow the same overall principles as ICRC-1 accounts.

ICRC-7 views the ICRC-1 `Account` as primary concept, meaning that operations like approvals and transfers refer to the full account and not only the principal part thereof. Thus, some methods comprise an extra optional `from_subaccount` or `spender_subaccount` parameter that together with the caller form an account to perform the respective operation on. Leaving such subaccount parameter `null` always has the semantics of referring to the default subaccount comprised of all zeroes.

### Token Identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids, which is, for example, important for making other NFT standards that use strings as token identifiers compatible with ICRC-7.

## Methods

### Conventions

The methods, both queries and update calls, that operate on token ids as inputs are *batch methods* that can receive vectors of token ids, i.e., batches of token ids, as input. The batch methods are to be used also for operations on single token ids. The output of most batch methods is a vector of records comprising a token id and a response belonging to the token id. Unless specified explicitly otherwise, the ordering of response elements for batch calls is not defined, i.e., is arbitrary. For batch calls, every distinct token id of the input MUST occur in at least one response element. If the input contains duplicate token ids for update calls, the ledger must trap. Duplicate token ids in batch query calls may result in duplicate token ids in the response or may be deduplicated at the discretion of the ledger. The lenght of the response vector may be shorter than that of the input vector in batch query calls, e.g., in case of duplicate token ids in the input and a deduplication being performed by the method's implementation logic. For update batch calls, the call MUST be executed on all provided token ids or none.

Methods that modify the state of the ledger have responses that comprise transaction indices as part of the response in the success case. Such a transaction index is an index into the chain of blocks containing the transaction history of this ledger. The format of the transaction history is not part of the ICRC-7 standard, but will be published as a separate standard.

All update methods have error variants defined for their responses that cover the error cases for the respective call. For query methods, error variants are only defined for methods for which a specific error needs to be handled. Other query calls do not have error variants defined for their responses. Both query and update calls can trap in specific circumstances.

The response size for responses to messages sent to a canister smart contract on the IC is constrained to a fixed constant size. For requests that could result in larger response messages, the caller SHOULD ensure to constrain the input accordingly so that the response remains below the maximum allowed size, e.g., not query too many token ids in one batch call. If the size limit of a response is hit, the ledger canister MUST trap. The ledger SHOULD make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

Each defined Candid type is only presented once in the text upon its first use in a method. Likewise, error responses are not always explained repeatedly for all methods after having been first explained.

### icrc7_collection_metadata

Returns all the collection-level metadata of the NFT collection in a single query. The data model for metadata is based on the generic `Value` type which allows for encoding arbitrarily complex data for each metadata attribute. The metadata attributes are expressed as `(text, value)` pairs where the first element is the name of the metadata attribute and the second element the corresponding value expressed through the `Value` type.

Analogous to [ICRC-1 metadata](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-1#metadata), metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons. Namespace `icrc7` is reserved for keys defined in the ICRC-7 standard.

The set of elements contained in a specific ledger's metadata depends on the given ledger implementation, the list below establishes the currently defined fields.

The following metadata fields are defined by ICRC-7, starting with general collection-specific metadata fields:
  * `icrc7:symbol` of type `text`: The token symbol. Token symbols are often represented similar to [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)) currency codes. When present, should be the same as the result of the [`icrc1_symbol`](#symbol_method) query call.
  * `icrc7:name` of type `text`: The name of the token. Should be the same as the result of the [`icrc7_name`](#icrc7_name) query call.
  * `icrc7:description` of type `text` (optional): A textual description of the token. When present, should be the same as the result of the [`icrc7_description`](#icrc7_description) query call.
  * `icrc7:logo` of type `text` (optional): The URL of the token logo. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself. When present, should be the same as the result of the [`icrc7_logo`](#icrc7_logo) query call.
  * `icrc7:total_supply` of type `nat`: The current total supply of the token, i.e., the number of tokens in existence. Should be the same as the result of the [`icrc7_total_supply`](#icrc7_total_supply) query call.
  * `icrc7:supply_cap` of type `nat` (optional): The current maximum supply for the token beyond which minting new tokens is not possible. When present, should be the same as the result of the [`icrc7_supply_cap`](#icrc7_supply_cap) query call.

The following are the more technical, implementation-oriented, metadata elements:
  * `icrc7:max_approvals_per_token_or_collection` of type `nat` (optional): The maximum number of active approvals this ledger implementation allows per token. When present, should be the same as the result of the [`icrc7_max_approvals_per_token_or_collection`](#icrc7_max_approvals_per_token) query call.
  * `icrc7:max_query_batch_size` of type `nat` (optional): The maximum batch size for query batch calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_query_batch_size`](#icrc7_max_query_batch_size) query call.
  * `icrc7:max_update_batch_size` of type `nat` (optional): The maximum batch size for update batch calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_update_batch_size`](#icrc7_max_update_batch_size) query call.
  * `icrc7:default_take_value` of type `nat` (optional): The default value this ledger uses for the `take` pagination parameter which is used in some queries. When present, should be the same as the result of the [`icrc7_default_take_value`](#icrc7_default_take_value) query call.
  * `icrc7:max_take_value` of type `nat` (optional): The maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated queries the ledger exposes. When present, should be the same as the result of the [`icrc7_max_take_value`](#icrc7_max_take_value) query call.
  * `icrc7:max_revoke_approvals` of type `nat` (optional): The maximum number of approvals that may be revoked in a single invocation of `icrc7_revoke_token_approvals` or `icrc7_revoke_collection_approvals`. When present, should be the same as the result of the [`icrc7_max_revoke_approvals`](#icrc7_max_revoke_approvals) query call.

Note that if max values specified through metadata are violated in a query call by providing larger argument lists or resulting in larger responses than permitted, the canister traps with an according system error message.

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
```

```candid "Methods" +=
icrc7_collection_metadata : () -> (metadata : vec record { text; Value } ) query;
```

### icrc7_symbol

Returns the token symbol of the NFT collection (e.g., `MS`).

```candid "Methods" +=
icrc7_symbol : () -> (text) query;
```

### icrc7_name

Returns the name of the NFT collection (e.g., `My Super NFT`).

```candid "Methods" +=
icrc7_name : () -> (text) query;
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

### icrc7_max_approvals_per_token_or_collection

Returns the maximum number of approvals this ledger implementation allows to be active per token.

```candid "Methods" +=
icrc7_max_approvals_per_token_or_collection : () -> (opt nat) query;
```

### icrc7_max_query_batch_size

Returns the maximum batch size for query batch calls this ledger implementation supports.

```candid "Methods" +=
icrc7_max_query_batch_size : () -> (opt nat) query;
```

### icrc7_max_update_batch_size

Returns the maximum number of token ids allowed for being used as input in a batch update method.

```candid "Methods" +=
icrc7_max_update_batch_size : () -> (opt nat) query;
```

### icrc7_default_take_value

Returns the default parameter the ledger uses for `take` in case the parameter is `null` in paginated queries.

```candid "Methods" +=
icrc7_default_take_value : () -> (opt nat) query;
```

### icrc7_max_take_value

Returns the maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated calls the ledger exposes.

```candid "Methods" +=
icrc7_max_take_value : () -> (opt nat) query;
```

### icrc7_max_revoke_approvals

Returns the maximum number of approvals that may be revoked in a single invocation of `icrc7_revoke_token_approvals` or `icrc7_revoke_collection_approvals`.

```candid "Methods" +=
icrc7_max_revoke_approvals : () -> (opt nat) query;
```

### icrc7_token_metadata

Returns the token metadata for `token_ids`, a list of token ids. Each tuple in the response vector comprises a token id as first element and the metadata corresponding to this token expressed as a `Value` as second element.

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
```

```candid "Methods" +=
icrc7_token_metadata : (token_ids : vec nat)
    -> (vec record { nat; opt Value }) query;
```

### icrc7_owner_of

Returns the owner `Account` of each token in a list `token_ids` of token ids. The response elements are sorted following an ordering depending on the ledger implementation.

For non-existing token ids, the `NonExistingTokenId` error variant is returned. The `NotMigrated` error type is used to help migration of NFT ledgers of other standards using ICP AccountId instead of ICRC-1 Account to store the owners. See section [Migration Path for Ledgers Using ICP AccountId](#migration-path-for-ledgers-using-icp-accountid). The `GenericError` type is used for expressing any other error and is not further specified in ICRC-7.

```candid "Type definitions" +=
type OwnerOfError = variant {
    NonExistingTokenId;
    NotMigrated;
    GenericError : record { error_code : nat; message : text };
};
```
```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat)
    -> (vec record { token_id : nat; account : variant { Ok : Account; Err : OwnerOfError } }) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument, i.e., the number of tokens held by the account. For a non-existing account, the value `0` is returned.

```candid "Methods" +=
icrc7_balance_of : (account : Account) -> (balance : nat) query;
```

### icrc7_tokens

Returns the list of tokens in this ledger, sorted by their token id.

The result is paginated and pagination is controlled via the `prev` and `take` parameters: The response to a request results in at most `take` many token ids, starting with the next id following `prev`. The token ids in the response are sorted in any consistent sorting order used by the ledger. If `prev` is `null`, the response elements start with the smallest ids in the ledger according to the sorting order. If the response to a call with a non-null `prev` value contains no token ids, there are no further tokens following `prev`. If `take` is omitted, the ledger's default `take` value as specified through `icrc7:default_take_value` is assumed. If the response to a call contains fewer token ids than the provided or default `take` value, there are no further tokens in the ledger following the largest returned token id.

For retrieving all tokens of the ledger, the pagination API is used such that the first call sets `prev = null` and specifies a suitable `take` value. Then, the method is called repeatedly such that the greatest token id of the previous response is used as `prev` value for the next call to the method. Using this approach, all tokens can be enumerated in ascending order, provided the ledger state does not change between the method calls.

Each invocation is executed on the current memory state of the ledger. I.e., it is not possible to enumerate the exact list of token ids of the ledger at a given time or of a "snapshot" of the ledger state. Rather, the ledger state can change between the multiple calls required to enumerate all the tokens.

```candid "Methods" +=
icrc7_tokens : (prev : opt nat, take : opt nat32)
    -> (token_ids : vec nat) query;
```

### icrc7_tokens_of

Returns a vector of `token_id`s of all tokens held by `account`, sorted by `token_id`.  The token ids in the response are sorted in any consistent sorting order used by the ledger. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination.

```candid "Methods" +=
icrc7_tokens_of : (account : Account, prev : opt nat, take : opt nat32)
    -> (token_ids : vec nat) query;
```

### icrc7_approve_tokens

Entitles a `spender`, indicated through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method (and also the owner principal of the tokens that are subject to approval) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens must reside on and can be transferred out from). Note that the `from_subaccount` parameter needs to be explicitly specified because accounts are a primary concept in this standard and thereby the `from_subaccount` needs to be specified as part of the account that holds the token. The `expires_at` value specifies the expiration date of the approval, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The `created_at_time` field specifies when the approval has been created. The parameter `token_ids` specifies a list of tokens to apply the approval to.

The response is a vector comprising records with a `token_id` as first element and an `Ok` variant with the transaction index for the success case or an `Err` variant indicating an error as second element.

The ledger SHOULD reject the call if the spender account owner is equal to the caller account owner.

An approval that has been created, has not expired (i.e., the `expires_at` field is a date in the future), has not been revoked, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initiate a transfer. Only one approval can be active for a given `(token_id, spender)` pair (the `from_subaccount` of the approval must be equal to the subaccount the token is held on).

In accordance with ICRC-2, multiple approvals can exist for the same `token_id` but different `spender`s (the `from_subaccount` field must be the same and equal to the subaccount the token is held on). In case an approval for the specified `spender` already exists for a token on `from_subaccount` of the caller, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters for the approval (`expires_at`, `memo`, `created_at_time`) apply. The ledger SHOULD limit the number of approvals that can be active per token to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc7:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block history the ledger creates.

An `Unauthorized` error is returned in case the caller is not authorized to perform this action on the token, i.e., it does not own the token or the token is not held in the account specified through `from_subaccount`.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

```candid "Type definitions" +=
type ApprovalInfo = record {
    from_subaccount : opt blob;
    spender : Account;             // Approval is given to an ICRC Account
    memo : opt blob;
    expires_at : opt nat64;
    created_at_time : opt nat64; 
};

type ApproveTokensError = variant {
    NonExistingTokenId;
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_approve : (token_ids : vec nat, approval : ApprovalInfo)
    -> (vec record { token_id : nat; approval_result : variant { Ok : nat; Err : ApproveTokensError } });
```

### icrc7_approve_collection

Entitles a `spender`, indicated through an `Account`, to transfer any NFT of the collection hosted on this ledger and owned by the caller at the time of transfer on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which tokens the approval should apply to must reside on and can be transferred out from). Note that the `from_subaccount` parameter needs to be explicitly specified not only because accounts are a primary concept in this standard, but also because the approval applies to the collection, i.e., all tokens on the ledger the caller holds, and those tokens may be held on different subaccounts. The `expires_at` value specifies the expiration date of the approval, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The `created_at_time` field specifies when the approval has been created.

The response contains a single element with the `Ok` variant containing the transaction index of the collection-level approval in the success case or an error `Err` otherwise.

The ledger SHOULD reject the call if the spender account owner is equal to the caller account owner.

An approval that has been created, has not expired (i.e., the `expires_at` field is a date in the future), has not been revoked, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initiate a transfer.

In accordance with ICRC-2, multiple approvals can exist for the collection for a caller but different `spender`s and `from_subaccount`s, i.e., one approval per `(spender, from_subaccount)` pair. In case an approval for the specified `spender` and `from_subaccount` of the caller for the collection already exists, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters for the approval (`expires_at`, `memo`, `created_at_time`) apply. The ledger SHOULD limit the number of approvals that can be active per collection to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc7:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block history the ledger creates.

Collection-level approvals can be successfully created independently of currently owning tokens of the collection at approval time.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

Note: This method is analogous to `icrc7_approve_tokens`, but for approving whole collections. `ApprovalInfo` specifies the approval to be made for the collection.

Note that collection-level approvals MUST be managed by the ledger as collection-level approvals and MUST NOT be translated into token-level approvals for all tokens the caller owns.

See the [#icrc7_approve_tokens](icrc7_approve_tokens) for the Candid types.

```candid "Type definitions" +=
type ApproveCollectionError = variant {
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_approve_collection : (ApprovalInfo)
    -> (approval_result : variant { Ok : nat; Err : ApproveCollectionError });
```

### icrc7_transfer

Transfers one or more tokens from the `from` account to the `to` account. The transfer can be initiated either by the holder of the tokens or a party that has been authorized by the holder to execute transfers using `icrc7_approve_tokens` or `icrc7_approve_collection`. The `spender_subaccount` is used to identify the spender. The spender is an account comprised of the principal calling this method and the parameter `spender_subaccount`. Omitting the `spender_subaccount` means using the default subaccount.

The response is a vector of records each comprising a `token_id` and a corresponding `transfer_result` indicating success or error. In the success case, the `Ok` variant indicates the transaction index of the transfer, in the error case, the `Err` variant indicates the error through `TransferError`.

A transfer clears all active token-level approvals for the successfully transferred tokens. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals.

```candid "Type definitions" +=
TransferArgs = record {
    spender_subaccount: opt blob; // the subaccount of the caller (used to identify the spender)
    from : Account;
    to : Account;
    token_ids : vec nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferError = variant {
    NonExistingTokenId;
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (TransferArgs)
    -> (vec record { token_id : nat; transfer_result : variant { Ok : nat; Err : TransferError } });
```

If the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response. This is the case if someone not owning a token and not being the spender in an active token-level or collection-level approval attempts to transfer a token or the token is not held in the subaccount specified in the `from` account.


The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

### icrc7_revoke_token_approvals

Revokes the specified approvals for specific tokens `token_ids` from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` value of `from_subaccount` indicates the default subaccount. A `null` value for `spender` means to revoke approvals with any value for the spender.

Only the owner of tokens can revoke approvals.

The response is a vector comprising records with a `token_id` and a corresponding variant with `Ok` containing a transaction index indicating the success case or an `Err` variant indicating the error case.

Note that the size of responses on ICP is limited. Callers of this method SHOULD take particular care to not exceed the response limit for their inputs, e.g., in case there are many approvals defined for a token id or many token ids with a few approvals each are provided as input.

Revoking an approval for one or more token ids does not affect collection-level approvals.

An ICRC-7 ledger implementation does not need to keep track of revoked approvals.

```candid "Type definitions" +=
type RevokeTokensArgs = record {
    token_ids : vec nat;
    from_subaccount : opt blob;
    spender : opt Account;
};

type RevokeTokensError = variant {
    NonExistingTokenId;
    Unauthorized;
    ApprovalDoesNotExist;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_revoke_token_approvals: (RevokeTokensArgs)
    -> (vec record { token_id : nat; spender : Account; revoke_result : variant { Ok : nat; Err : RevokeTokensError } });
```

### icrc7_revoke_collection_approvals

Revokes collection-level approvals from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` value of `from_subaccount` indicates the default subaccount. A `null` value for `spender` means to revoke approvals with any value for the spender.

The response is a vector containing a record for each revoked approval. Each element is the `Ok` variant in the success case containing the transaction index and the `Err` variant in the error case containing an error.

This is the analogous method to `icrc7_revoke_token_approvals` for revoking collection-level approvals.

Revoking a collection-level approval does not affect token-level approvals for individual token ids.

Note that the size of responses on ICP is limited. Callers of this method should take particular care to not exceed the response limit for their inputs by revoking too many collection-level approvals with one request.

An ICRC-7 ledger implementation does not need to keep track of revoked approvals.

```candid "Type definitions" +=
type RevokeCollectionArgs = record {
    from_subaccount : opt blob;
    spender : opt Account;
};

type RevokeCollectionError = variant {
    ApprovalDoesNotExist;
    GenericError : record { error_code : nat; message : text };
};
```
```candid "Methods" +=
icrc7_revoke_collection_approvals: (RevokeCollectionArgs)
    -> (vec record { spender : Account; from_subaccount : blob; revoke_result : variant { Ok : nat; Err : RevokeCollectionError } });
```

### icrc7_is_approved

Returns `true` if an active approval exists that allows the `spender` to transfer the token `token_id` from the given `from_subaccount`, `false` otherwise.

```candid "Methods" +=
icrc7_is_approved : (spender : Account; from_subaccount : opt blob; token_id : nat)
    -> (bool) query;
```

### icrc7_get_token_approvals

Returns the token-level approvals that exist for the given vector of `token_ids`.  The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination, with `prev` being of type `TokenApproval`. Note that `take` refers to the number of returned elements to be requested. The `prev` parameter is a `TokenApproval` element with the meaning that `TokenApproval`s following the provided one are returned, based on a sorting order over `TokenApproval`s implemented by the ledger.

The response is a vector of `TokenApproval` elements. If multiple approvals exist for a token id, multiple entries of type `TokenApproval` with the same token id are contained in the response.

The ordering of the elements in the response is undefined. An implementation of the ledger can use any internal sorting order for the elements of the response to implement pagination.

```candid "Type definitions" +=
type TokenApproval = record {
    token_id : nat;
    approvalInfo : ApprovalInfo;
};
```

```candid "Methods" +=
icrc7_get_approvals : (token_ids : vec nat, prev : opt TokenApproval; take : opt nat32)
    -> (vec TokenApproval) query;
```

### icrc7_get_collection_approvals

Returns the collection-level approvals that exist for the specified `owner`. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination. The `prev` parameter is an `CollectionApproval` with the meaning that `CollectionApproval`s following the provided one are returned, based on a sorting order over `ApprovalInfo`s implemented by the ledger.

The response is a vector of `CollectionApproval` elements.

The ordering of the elements in the response is undefined. An implementation of the ledger can use any internal sorting order for the elements of the response to implement pagination.

```candid "Type definitions" +=
type CollectionApproval = ApprovalInfo;
```

```candid "Methods" +=
icrc7_get_collection_approvals : (owner : Account, prev : opt CollectionApproval, take : opt nat32)
    -> (vec CollectionApproval) query;
```

### icrc7_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc7_supported_standards : () -> (vec record { name : text; url : text }) query;
```

The result of the call should always have at least one entry,

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7"; }
```

## Migration Path for Ledgers Using ICP AccountId

For historical reasons, multiple NFT standards, such as the EXT standard, use the ICP AccountId (a hash of the principal and subaccount) instead of the ICRC-1 Account (a pair of principal and subaccount) to store the owners. Since the ICP AccountId can be calculated from an ICRC-1 Account, but computability does not hold in the inverse direction, there is no way for a ledger implementing ICP AccountId to display `icrc7_owner_of` data. To help with the transition, ledgers using ICP AccountId can return error type `NotMigrated` for tokens that exist, but have not migrated to ICRC-1 Account yet (the owner of the NFT token would need to call a migration endpoint in the canister as part of the migration process, which may take an arbitrary amount of time to migrate all tokens).

## Extensions

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  - Reliable transaction notifications for smart contracts.
  - The block structure and the interface for fetching blocks.
  - Pre-signed transactions.

The standard defines the `icrc7_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-42"` or `"DIP-20"`) implemented by the ledger.

## Transaction Deduplication

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-7 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-7 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer`](#icrc7_transfer) call argument:
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
