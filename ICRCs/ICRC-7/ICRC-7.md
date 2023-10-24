|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai)|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Minimal Non-Fungible Token (NFT) Standard

ICRC-7 is the minimal standard for the implementation of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

## Data

### Account

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore, an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the _default account_ of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount };
```

The canonical textual representation of the account follows the [definition in ICRC-1](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md). ICRC-7 accounts have the same structure and follow the same overall principles as ICRC-1 accounts.

### Token Identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids.

## Methods

### Conventions

Unless specified explicitly otherwise, the ordering of response elements for batch calls is not defined, i.e., is arbitrary. The methods, both queries and update calls, that operate on token ids are batch methods that can receive vectors of token ids as input. The output of such a method is a vector with records comprising a token id and a response belonging to this token id. The ordering of the response elements is undefined. This API pattern does not require the caller to correlate the response elements to the input, but the response is self contained. For batch calls, the lenght of the output vector may be shorter than that of the input vector, e.g., in case of duplicate token ids in the input and a deduplication being performed by the method's implementation logic.

Methods that modify the state of the ledger have responses that comprise transaction indices as part of the response in the success case. Such a transaction index is an index into the chain of transactions that have been made for this ledger and therefore refers a specific block created for this ledger.

The response size for messages sent to a canister is constrained currently at 2MB. For requests that could result in larger response messages, the caller needs to ensure to constrain the input accordingly so that the response remains below the maximum allowed size, e.g., not query too many token ids in one batch call. If the maximum size of a response is hit, the ledger canister traps. The ledger SHOULD make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

### icrc7_collection_metadata

Returns all the collection-level metadata of the NFT collection in a single query. The data model for metadata is based on the generic `Value` type which allows for encoding arbitrarily complex data for each metadata element. The metadata attributes are expressed as `(key : text, value : Value)` pairs where `key` is the name of the metadata attribute and `value` the corresponding value expressed through the `Value` type.

Analogous to [ICRC-1 metadata](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-1#metadata), metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons. Namespace `icrc7` is reserved for keys defined in the ICRC-7 standard.

The set of elements contained in a specific ledger's metadata depends on the given ledger implementation, the list below establishes the currently defined fields.

The following metadata fields are defined by ICRC-7, starting with general collection-specific metadata fields:
  * `icrc7:symbol` of type `text`: The token currency code (see [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)). When present, should be the same as the result of the [`icrc1_symbol`](#icrc7_symbol) query call.
  * `icrc7:name` of type `text`: The name of the token. Should be the same as the result of the [`icrc7_name`](#icrc7_name) query call.
  * `icrc7:description` of type `text` (optional): A textual description of the token. When present, should be the same as the result of the [`icrc7_description`](#icrc7_description) query call.
  * `icrc7:logo` of type `text` (optional): The URL of the token logo. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself. When present, should be the same as the result of the [`icrc7_logo`](#icrc7_logo) query call.
  * `icrc7:total_supply` of type `nat`: The current total supply of the token, i.e., the number of tokens in existence. Should be the same as the result of the [`icrc7_total_supply`](#icrc7_total_supply) query call.
  * `icrc7:supply_cap` of type `nat` (optional): The current maximum supply for the token beyond which minting new tokens is not possible. When present, should be the same as the result of the [`icrc7_supply_cap`](#icrc7_supply_cap) query call.

The following are the more technical, implementation-oriented, metadata elements:
  * `icrc7:max_approvals_per_token_or_collection` of type `nat` (optional): The maximum number of active approvals this ledger implementation allows per token. When present, should be the same as the result of the [`icrc7_max_approvals_per_token_or_collection`](#icrc7_max_approvals_per_token) query call.
  * `icrc7:max_query_batch_size` of type `nat` (optional): The maximum batch size for query batch calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_query_batch_size`](#icrc7_max_query_batch_size) query call.
  * `icrc7:max_update_batch_size` of type `nat` (optional): The maximum batch size for update batch calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_update_batch_size`](#icrc7_max_update_batch_size) query call.
  * `icrc7:default_take_value` of type `nat` (optional): The default value this ledger uses for the `take` pagination parameter. When present, should be the same as the result of the [`icrc7_default_take_value`](#icrc7_default_take_value) query call.
  * `icrc7:max_take_value` of type `nat` (optional): The maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated calls the ledger exposes. When present, should be the same as the result of the [`icrc7_max_take_value`](#icrc7_max_take_value) query call.
  * `icrc7:max_revoke_approvals` of type `nat` (optional): The maximum number of approvals that may be revoked in a single invocation of `icrc7_revoke_token_approvals` or `icrc7_revoke_collection_approvals`. When present, should be the same as the result of the [`icrc7_max_revoke_approvals`](#icrc7_max_revoke_approvals) query call.

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
icrc7_collection_metadata : () -> (metadata : vec record { key: text; value: Value } ) query;
```

### icrc7_symbol

Returns the symbol, i.e., the token currency code (see [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)), of the NFT collection (e.g., `MS`).

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

Returns the default parameter the ledger uses for `take` in case the parameter is `null` in paginated methods.

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
```

```candid "Methods" +=
icrc7_token_metadata : (token_ids : vec nat)
    -> (vec record { token_id : nat; metadata : Value }) query;
```

### icrc7_owner_of

Returns the owner `Account` of each token in a list `token_ids` of token ids. The ordering of the response elements is arbitrary.

For non-existing token ids, the `NonExistingTokenId` error variant is returned.

The `NotMigrated` error type is used to help migration of NFT ledgers of other standards using ICP AccountId instead of ICRC-1 Account to store the owners. See section [Migration Path for Ledgers Using ICP AccountId](#migration-path-for-ledgers-using-icp-accountid).

The `GenericError` type is used for expressing any other error and is not further specified in ICRC-7.

```candid "Type definitions" +=
type GetOwnerError = variant {
    NonExistingTokenId;
    NotMigrated;
    GenericError : record { error_code : nat; message : text };
};
```
```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat)
    -> (vec record { token_id : nat; account : variant { Ok : Account; Err : GetOwnerError } }) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument, i.e., the number of tokens held by the account. For a non-existing account, the value `0` is returned.

```candid "Methods" +=
icrc7_balance_of : (account : Account) -> (balance : nat) query;
```

### icrc7_tokens

Returns the list of tokens in this ledger, sorted by their token id.

The result is paginated and pagination is controlled via the `prev` and `take` parameters: The response to a request results in at most `take` many token ids, starting with the next id following `prev`. If `prev` is `null`, the response elements start are the smallest ids in the ledger according to the sorting order. If the response contains no token ids, there are no further tokens following `prev`. If the response contains fewer token ids than the provided or default `take` value, there are no further tokens following the largest returned token id. The token ids in the response are sorted in any consistent sorting order used by the ledger. If `take` is omitted, the ledger's default `take` value as specified through `icrc7:default_take_value` is assumed.

For retrieving all tokens of the ledger, the pagination API is used such that the first call sets `prev = null` and specifies a suitable `take` value, then the method is called repeatedly such that the greatest token id of the previous response is used as `prev` value for the next call to the method. This way all tokens can be enumerated in ascending order, provided token ids are not inserted during normal operation. 

Each invocation is executed on the current memory state of the ledger.

```candid "Methods" +=
icrc7_tokens : (prev : opt nat, take : opt nat32)
    -> (token_ids : vec nat) query;
```

### icrc7_tokens_of

Returns a vector of `token_id`s of all tokens held by `account`, sorted by `token_id`. The result is paginated, the mechanics of pagination are the same as for `icrc7_tokens` using `prev` and `take` to control pagination.

```candid "Methods" +=
icrc7_tokens_of : (account : Account, prev : opt nat, take : opt nat32)
    -> (token_ids : vec nat) query;
```

### icrc7_approve_tokens

Entitles a `spender`, indicated through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount; }`, where `caller` is the caller of this method (and the token owner principal) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens can be transferred out from). The call resets the expiration date, memo, and creation timestamp for the approval to the specified values in case an approval for the `spender` and `from_subaccount` already exists for a token. The parameter `tokens` specifies a list of token ids to apply the approval to.

The ledger SHOULD reject the call if the spender account owner is equal to the caller account owner.

An approval that has been created, is not expired, has not been revoked, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initate a transfer.

In accordance with ICRC-2, multiple approvals can exist for the same `token_id` but different `spender`s and `from_subaccount`s. For the same token, spender, and subaccount triple a new approval shall always overwrite the old one. The ledger should limit the number of approvals that can be active per token. Such limit is exposed as ledger metadata through the metadata attribute `icrc7:max_approvals_per_token_or_collection`.

The response is a vector comprising records with a `token_id` as first element and an `Ok` variant with the transaction index for the success case or an `Err` variant indicating an error as second element.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. Of course, all historic approvals are contained in the block history the ledger creates.

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

type ApprovalError = variant {
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    NotMigrated;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_approve : (token_ids : vec nat, approval : ApprovalInfo)
    -> (vec record { token_id : opt nat; approval_response : variant { Ok : nat; Err : ApprovalError } });
```

### icrc7_approve_collection

Entitles a `spender`, indicated through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount; }`, where `caller` is the caller of this method (and the token owner principal) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens can be transferred out from). The call resets the expiration date, memo, and creation timestamp for the approval to the specified values in case an approval for the `spender` and `from_subaccount` already exists for a token.

The ledger SHOULD reject the call if the spender account owner is equal to the caller account owner.

An approval that has been created, is not expired, has not been revoked, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initate a transfer.

In accordance with ICRC-2, multiple approvals can exist for the same `collection` but different `spender`s and `from_subaccount`s. For the same token, spender, and subaccount triple a new approval shall always overwrite the old one. The ledger should limit the number of approvals that can be active per collection. Such limit is exposed as ledger metadata through the metadata attribute `icrc7:max_approvals_per_token_or_collection`.

The response contains a single element with the `Ok` variant containing the transaction index of the collection-level approval in the success case or an error `Err` otherwise.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. Of course, all historic approvals are contained in the block history the ledger creates.

Note: This method is analogous to `icrc7_approve_tokens`, but for approving whole collections. `ApprovalInfo` defines the approval to be made for the collection.

Note that collection-level approvals MUST be managed by the ledger as collection-level approvals and MUST NOT be translated into token-level approvals for all tokens the caller owns.

See the [#icrc7_approve_tokens](icrc7_approve_tokens) for the Candid types.

```candid "Methods" +=
icrc7_approve_collection : (ApprovalInfo)
    -> (approval_response : variant { Ok : nat; Err : ApprovalError });
```

### icrc7_transfer

Transfers one or more tokens from the `from` account to the `to` account. The transfer can be initiated either by the holder of the tokens or a party that has been authorized by the holder to execute transfers using `icrc7_approve_tokens` or `icrc7_approve_collection`. The `spender_subaccount` is used to identify the spender. The spender is an account comprised of the principal calling this method and the parameter `spender_subaccount`. Leaving out the `spender_subaccount` means to use the default subaccount.

The response is a vector of records each comprising a `token_id` and a corresponding `transfer_result` indicating success or error. In the success case, the `Ok` variant indicates the transaction index of the transfer, in the error case, the `Err` variant indicates the error through `TransferError`.

A transfer clears all approvals for the successfully transferred tokens. This implicit clearing of approvals only clears token-id-based approvals and never touches collection-level approvals.

```candid "Type definitions" +=
type TransferArgs = record {
    spender_subaccount: opt blob; // the subaccount of the caller (used to identify the spender)
    from : Account;
    to : Account;
    token_ids : vec nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferError = variant {
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    NotMigrated;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (TransferArgs)
    -> (vec record { token_id : nat; transfer_result : variant { Ok : nat; Err : TransferError } });
```

If a token id doesn't exist or if the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

### icrc7_revoke_token_approvals

Revokes the specified approvals for specific tokens `token_ids` from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. Not providing the `from_subaccount` or `spender` means to revoke approvals with any value for the omitted parameter(s). By not specifying both `from_subaccount` and `spender`, all token-level approvals of the caller for the given `token_ids` are revoked.

Only the owner of tokens can revoke approvals.

The response is a vector comprising records with a `token_id` and a corresponding variant with `Ok` containing a transaction index indicating the success case or an `Err` variant indicating the error case.

Note that the size of responses on ICP is limited. Callers of this method should take particular care to not exceed the response limit for their inputs, e.g., in case there are many approvals defined for a token id or many token ids with a few approvals each are provided as input.

Revoking an approval for one or more token ids does not affect collection-level approvals.

An ICRC-7 ledger implementation does not need to keep track of revoked approvals.

```candid "Type definitions" +=
type RevokeTokensArgs = record {
    token_ids : vec nat;
    from_subaccount : opt blob;
    spender : opt Account;
};

type RevokeError = variant {
    Unauthorized;
    ApprovalDoesNotExist;
    NotMigrated;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_revoke_token_approvals: (RevokeTokensArgs)
    -> (vec record { token_id : nat; from_subaccount : blob; spender : Account;
                     revoke_response : variant { Ok : nat; Err : RevokeError } });
```

### icrc7_revoke_collection_approvals

Revokes collection-level approvals from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval to be revoked applies, the `spender` the party for which the approval is to be revoked. Not providing the `from_subaccount` or `spender` means to revoke approvals with any value for the omitted parameter(s). By not specifying both `from_subaccount` and `spender`, all collection-level approvals of the caller are revoked.

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
```
```candid "Methods" +=
icrc7_revoke_collection_approvals: (RevokeCollectionArgs)
    -> (vec record { from_subaccount : blob; spender : Account; variant { Ok : nat; Err : RevokeError } });
```

### icrc7_is_approved

Returns `true` if an active approval exists that allows the `spender` to transfer the token `token_id` from the given `from_subaccount`, `false` otherwise.

```candid "Methods" +=
icrc7_is_approved : (spender : Account; from_subaccount : blob; token_id : nat)
    -> (bool) query;
```

### icrc7_get_token_approvals

Returns the token-level approvals that exist for the given vector of `token_ids`.  The result is paginated, the mechanics of pagination are the same as for `icrc7_tokens` using `prev` and `take` to control pagination. Note that `take` refers to the number of returned elements to be requested. The `prev` parameter is an `TokenApproval` with the meaning that `TokenApproval`s following the provided one are returned, based on a sorting order over `TokenApproval`s implemented by the ledger.

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

Returns the collection-level approvals that exist for the specified `owner`. The result is paginated, the mechanics of pagination are the same as for `icrc7_tokens` using `prev` and `take` to control pagination. The `prev` parameter is an `ApprovalInfo` with the meaning that `ApprovalInfo`s following the provided one are returned, based on a sorting order over `ApprovalInfo`s implemented by the ledger.

The response is a vector of `ApprovalInfo` elements.

The ordering of the elements in the response is undefined. An implementation of the ledger can use any internal sorting order for the elements of the response to implement pagination.

```candid "Methods" +=
icrc7_get_collection_approvals : (owner : Account, prev : opt ApprovalInfo, take : opt nat32)
    -> (vec ApprovalInfo) query;
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
