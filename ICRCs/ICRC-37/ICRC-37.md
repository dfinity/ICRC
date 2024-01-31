|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|37|Approval Functionality for the Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai), Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer), Thomas (@sea-snake), Moritz Fuller (@letmejustputthishere), Matthew Harmon|https://github.com/dfinity/ICRC/issues/37|Draft|Standards Track||2023-11-22|


# ICRC-37: Approval Support for the Minimal Non-Fungible Token (NFT) Standard

This document specifies approval support for the ICRC-7 minimal NFT standard for the Internet Computer. It defines all the methods required for realizing approval semantics for an NFT token ledger, i.e., creating approvals, revoking approvals, querying approval information, and making transfers based on approvals. The scope of ICRC-37 has been part of ICRC-7 originally, however, the NFT Working Group has decided to split it out into a separate standard for the following reasons:
  * ICRC-7 and ICRC-37 are much shorter and hence easier to navigate on their own due to their respective foci;
  * Ledgers that do not want to implement approval and transfer from semantics do not need to provide dummy implementations of the corresponding methods that fail by default.

This standard extends the ICRC-7 NFT standard and is intended to be implemented by token ledgers that implement ICRC-7. An ICRC-7 ledger may implement ICRC-37 in case it intends to offer approve and transfer from semantics. Principles put forth in ICRC-7 apply to ICRC-37 as well, e.g., the design of the update and query API.

## Concepts

*Approvals* allow a principal, the *spender*, to transfer tokens owned by another account that has approved the spender, where the transfer is performed on behalf of the owner. Approvals can be created on a per-token basis using `icrc37_approve_tokens` or for the whole collection, i.e., all tokens of the collection, using `icrc37_approve_collection`. The owner principal can explicitly revoke an active approval at their discretion using the `icrc37_revoke_token_approvals` for revoking token-level approvals and `icrc37_revoke_collection_approvals` for revoking collection-level approvals. A transfer of a token implicitly revokes all token-level approvals of the transferred token. A collection-level approval is not affected by any changes of token ownerships and is not related to specific tokens. An approval that has been created, has not expired (i.e., the `expires_at` field is a date in the future), has not been revoked implicitly through a transfer of the approved token, has not been revoked explicitly, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initiate a transfer.

When an active approval exists for a token or for an account for the whole collection, the spender specified in the approval can transfer tokens within the scope of the approval using the `transfer_from` method. A successful transfer implicitly revokes all active token-level approvals for the token in addition to performing the actual token transfer. Collection-level approvals are never revoked by transfers as they are not related to specific tokens.

Analogous to ICRC-7, also ICRC-37 uses the ICRC-1 *account* as entity that the source account (`from`), destination account (`to`), and spending account (`spender`) are expressed with, i.e., a *subaccount* is always used besides the principal. We follow the naming convention of using `from_subaccount` for subaccounts being part of the source account and `spender_subaccount` for subaccounts being part the spender account. In many practical situations the default subaccount, i.e., the all-`0` subaccount, is expected to be used.

## Methods

### Generally-Applicable Specification

The general principles on the API as put forth in [ICRC-7](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md) apply also to ICRC-37.

To summarize, all eligible calls, i.e., such with (at most) one response element per request, are batch calls. Batch calls have *positional responses*, i.e., the `i`-th response element is the response to the `i`-th request element. The response may contain responses only to a prefix of the request. For update calls, responses can contain `null` elements, with the meaning that processing of the corresponding request has not been initiated. For query calls, `null` responses have a meaning as determined by the respective query call.

### icrc37_metadata

Returns the approval-related metadata of the ledger implementation. The metadata representation is analogous to that of [ICRC-7](https://github.com/dfinity/ICRC/ICRCs/ICRC-7/ICRC-7.md#icrc7_collection_metadata) using the `Value` type to represent properties.

The following metadata property is defined for ICRC-37:
  * `icrc37:max_approvals_per_token_or_collection` of type `nat` (optional): The maximum number of active approvals this ledger implementation allows per token or per principal for the collection. When present, should be the same as the result of the [`icrc37_max_approvals_per_token_or_collection`](#icrc37_max_approvals_per_token_or_collection) query call.
  * `icrc37:max_revoke_approvals` of type `nat` (optional): The maximum number of approvals that may be revoked in a single invocation of `icrc37_revoke_token_approvals` or `icrc37_revoke_collection_approvals`. When present, should be the same as the result of the [`icrc37_max_revoke_approvals`](#icrc37_max_revoke_approvals) query call.

Note that all other relevant ``icrc7:...` metadata properties from the ICRC-7 implementation that this standard is an extension of apply to this standard, e.g., the maximum batch sizes for queries and updates.

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
icrc37_metadata : () -> (vec record { text; Value } ) query;
```

### icrc37_max_approvals_per_token_or_collection

Returns the maximum number of approvals this ledger implementation allows to be active per token or per principal for the collection.

```candid "Methods" +=
icrc37_max_approvals_per_token_or_collection : () -> (opt nat) query;
```

### icrc37_max_revoke_approvals

Returns the maximum number of approvals that may be revoked in a single invocation of `icrc37_revoke_token_approvals` or `icrc37_revoke_collection_approvals`.

```candid "Methods" +=
icrc37_max_revoke_approvals : () -> (opt nat) query;
```

### icrc37_approve_tokens

Entitles a `spender`, specified through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method (and also the owner principal of the tokens that are subject to approval) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens must be held on and can be transferred out from). The `from_subaccount` being `null` refers to the default subaccount. Note that the `from_subaccount` parameter needs to be specified because accounts are a primary concept in this standard and thereby the `from_subaccount` needs to be specified as part of the account that holds the token. The `expires_at` value specifies the expiration date of the approval. The method has batch semantics and allows for submitting a batch of such token-level approvals with a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the token-level approval in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

Only one approval can be active for a given `(token_id, spender)` pair (the `from_subaccount` of the approval must be equal to the subaccount the token is held on).

In accordance with ICRC-2, multiple approvals can exist for the same `token_id` but different `spender`s (the `from_subaccount` field must be the same and equal to the subaccount the token is held on). In case an approval for the specified `spender` already exists for a token on `from_subaccount` of the caller, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters for the approval (`expires_at`, `memo`, `created_at_time`) apply. The ledger SHOULD limit the number of approvals that can be active per token to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc37:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block history the ledger creates.

The ledger returns an `InvalidSpender` error if the spender account owner is equal to the caller account owner. I.e., a principal cannot create an approval for themselves, because a principal always has an implicit approval to act on their own tokens.

An `Unauthorized` error is returned in case the caller is not authorized to perform this action on the token, i.e., it does not own the token or the token is not held in the account specified through `from_subaccount`.

A `NonExistingTokenId` error is returned in case the referred-to token does not exist.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

```candid "Type definitions" +=
type ApprovalInfo = {
    spender : Account;             // Approval is given to an ICRC Account
    from_subaccount : opt blob;    // the subaccount the token can be transferred out from with the approval
    expires_at : opt nat64;
    memo : opt blob;
    created_at_time : nat64; 
};

type ApproveTokenArg = record {
    token_id : nat;
    approval_info : ApprovalInfo;
};

type ApproveTokenResult = variant {
    Ok : nat; // Transaction index for successful approval
    Err : ApproveTokenError;
};

type ApproveTokenError = variant {
    InvalidSpender;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_approve_tokens : (vec ApproveTokenArg)
    -> (vec opt ApproveTokenResult);
```

### icrc37_approve_collection

Entitles a `spender`, specified through an `Account`, to transfer any NFT of the collection hosted on this ledger and owned by the caller at the time of transfer on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which tokens the approval should apply to must be held on and can be transferred out from). The `from_subaccount` being `null` refers to the default subaccount. Note that the `from_subaccount` parameter needs to be specified not only because accounts are a primary concept in this standard, but also because the approval applies to the collection, i.e., all tokens on the ledger the caller holds, and those tokens may be held on different subaccounts. The `expires_at` value specifies the expiration date of the approval. The method has batch semantics and allows for submitting a batch of such collection approvals with a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the collection-level approval in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

Only one approval can be active for a given `(spender, from_subaccount)` pair. Note that it is not required that tokens be held by the caller on their `from_subaccount` for the approval to be active.

In accordance with ICRC-2, multiple approvals can exist for the collection for a caller but different `spender`s and `from_subaccount`s, i.e., one approval per `(spender, from_subaccount)` pair. In case an approval for the specified `spender` and `from_subaccount` of the caller for the collection already exists, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters (`expires_at`, `memo`, `created_at_time`) apply to the approval defined by `from_subaccount` and `spender`. The ledger SHOULD limit the number of approvals that can be active per collection to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc37:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block log history the ledger creates.

It is left to the ledger implementation to decide whether collection-level approvals can be successfully created independently of currently owning tokens of the collection at approval time.

The ledger returns an `InvalidSpender` error if the spender account owner is equal to the caller account owner. I.e., a principal cannot create an approval for themselves, because a principal always has an implicit approval to act on their own tokens.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

Note that this method is analogous to `icrc37_approve_tokens`, but for approving whole collections. `ApproveCollectionArg` specifies the approval to be made for the collection.

To ensure proper semantics, collection-level approvals MUST be managed by the ledger as collection-level approvals and MUST NOT be translated into token-level approvals for all tokens the caller currently owns.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) for the `ApprovalInfo` type.

```candid "Type definitions" +=
type ApproveCollectionArg = record {
    approval_info : ApprovalInfo;
};

type ApproveCollectionResult = variant {
    Ok : nat; // Transaction index for successful approval
    Err : ApproveCollectionError;
};

type ApproveCollectionError = variant {
    InvalidSpender;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_approve_collection : (vec ApproveCollectionArg)
    -> (vec opt ApproveCollectionError);
```

### icrc37_revoke_token_approvals

Revokes the specified approvals for a token given by `token_id` from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` for the spender means to revoke approvals that match the request for all spenders. A `null` value of `from_subaccount` indicates the default subaccount. The method allows for a batch of token approval revocations in a single invocation.

Only the owner of tokens can revoke approvals through this method.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the token-level approval revocation in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

An `ApprovalDoesNotExist` error is returned in case the approval to be revoked is not an existing active approval.

An `Unauthorized` error is returned in case the caller is not authorized to perform this action on the token.

A `NonExistingTokenId` error is returned in case the referred-to token does not exist.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

Revoking an approval for one or more token ids does not affect collection-level approvals.

An ICRC-37 ledger implementation does not need to keep track of revoked approvals in memory. Revoked approvals are always available in the transaction log.

```candid "Type definitions" +=
type RevokeTokenApprovalArg = record {
    spender : opt Account;      // null revokes matching approvals for all spenders
    from_subaccount : opt blob; // null refers to the default subaccount
    token_id : nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type RevokeTokenApprovalResponse = variant {
    Ok : nat; // Transaction indices for successful approval revocation
    Err : RevokeTokenApprovalError;
};

type RevokeTokenApprovalError = variant {
    ApprovalDoesNotExist;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_revoke_token_approvals: (vec RevokeTokenApprovalArg)
    -> (vec opt RevokeTokenApprovalResponse);
```

### icrc37_revoke_collection_approvals

Revokes collection-level approvals from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` for the spender means to revoke approvals that match the request for all spenders. A `null` value of `from_subaccount` indicates the default subaccount. The method allows for a batch of token approval revocations in a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the collection-level approval revocation in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

This is the analogous method to `icrc37_revoke_token_approvals` for revoking collection-level approvals.

Revoking a collection-level approval does not affect token-level approvals for individual token ids.

An `ApprovalDoesNotExist` error is returned in case the approval to be revoked is not an existing active approval.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

An ICRC-37 ledger implementation does not need to keep track of revoked approvals in memory. Revoked approvals are always available in the transaction log.

```candid "Type definitions" +=
type RevokeCollectionApprovalArg = record {
    spender : opt Account;      // null revokes approvals for all spenders that match the remaining parameters
    from_subaccount : opt blob; // null refers to the default subaccount
    memo : opt blob;
    created_at_time : opt nat64;
};

type RevokeCollectionApprovalResult = variant {
    Ok : nat; // Transaction index for successful revocation
    Err : RevokeCollectionApprovalError;
};

type RevokeCollectionApprovalError = variant {
    ApprovalDoesNotExist;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_revoke_collection_approvals: (vec RevokeCollectionApprovalArg)
    -> (vec opt RevokeCollectionApprovalResult);
```

### icrc37_is_approved

Returns `true` if an active approval, i.e., a token-level approval or collection-level approval, exists that allows the `spender` to transfer the token `token_id` from the given `from_subaccount`, `false` otherwise.

```candid "Type definitions" +=
type IsApprovedArg = record {
    spender : Account;
    from_subaccount : opt blob;
    token_id : nat;
};
```

```candid "Methods" +=
icrc37_is_approved : (vec IsApprovedArg)
    -> (vec bool) query;
```

### icrc37_get_token_approvals

Returns the token-level approvals that exist for the given `token_id`.  The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination, with `prev` being of type `TokenApproval`. Note that `take` refers to the number of returned elements to be requested. The `prev` parameter is a `TokenApproval` element with the meaning that `TokenApproval`s following the provided one are returned, based on a sorting order over `TokenApproval`s implemented by the ledger.

The response is a vector of `TokenApproval` elements. If multiple approvals exist for a token id, multiple entries of type `TokenApproval` with the same token id are contained in the response.

The ordering of the elements in the response is follows an internal ordering used by the ledger. An implementation of the ledger can use any internal sorting order for the elements of the response to implement pagination.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) for the `ApprovalInfo` type.

```candid "Type definitions" +=
type TokenApproval = record {
    token_id : nat;
    approval_info : ApprovalInfo;
};
```

```candid "Methods" +=
icrc37_get_token_approvals : (token_id : nat, prev : opt TokenApproval; take : opt nat)
    -> (vec TokenApproval) query;
```

### icrc37_get_collection_approvals

Returns the collection-level approvals that exist for the specified `owner`. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination. The `prev` parameter is a `CollectionApproval` with the meaning that `CollectionApproval`s following the provided one are returned, based on a sorting order over `CollectionApproval`s implemented by the ledger.

The response is a vector of `CollectionApproval` elements.

The elements in the response are ordered following a sorting order defined by the implementation. An implementation of the ledger can use any suitable sorting order for the elements of the response to implement pagination.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) for the `ApprovalInfo` type.

```candid "Type definitions" +=
type CollectionApproval = ApprovalInfo;
```

```candid "Methods" +=
icrc37_get_collection_approvals : (owner : Account, prev : opt CollectionApproval, take : opt nat)
    -> (vec CollectionApproval) query;
```

### icrc37_transfer_from

Transfers one or more tokens from the `from` account to the `to` account. The transfer can be initiated by the holder of the tokens (the holder has an implicit approval for acting on all their tokens on their own behalf) or a party that has been authorized by the holder to execute transfers using `icrc37_approve_tokens` or `icrc37_approve_collection`. The `spender_subaccount` is used to identify the spender: The spender is an account comprised of the principal calling this method and the parameter `spender_subaccount`. Omitting the `spender_subaccount` means using the default subaccount.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the transfer in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

The ledger returns an `InvalidRecipient` error in case `to` equals `from`.

Each of the successful transfers in the batch implicitly clears all active token-level approvals for the successfully transferred token. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals. No explicit entry in the block log is created for the clearing, but it is implied by the transfer entry.

Batch transfers are not atomic by default, i.e., a user SHOULD not assume that either all or none of the transfers have been executed. A ledger implementation MAY choose to implement atomic batch transfers, in which case the metadata attribute `icrc7_atomic_batch_transfers` is set to `true`. If an implementation does not specifically implement batch atomicity, batch transfers are not atomic due to the asynchronous call semantics of the Internet Computer platform. An implementor of this standard who implements atomic batch transfers and advertises those through the `icrc7_atomic_batch_transfers` metadata attribute MUST take great care to ensure everything required has been considered to achieve atomicity of the batch of transfers.

```candid "Type definitions" +=
TransferFromArg = record {
    spender_subaccount: opt blob; // the subaccount of the caller (used to identify the spender)
    from : Account;
    to : Account;
    token_id : nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferFromResponse = variant {
    Ok : nat; // Transaction index for successful transfer
    Err : TransferFromError;
};

type TransferFromError = variant {
    InvalidRecipient;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_transfer_from : (vec TransferFromArg)
    -> (vec opt TransferFromResponse);
```

If the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response. This is the case if someone not owning a token and not being the spender in an active token-level or collection-level approval attempts to transfer a token or the token is not held in the subaccount specified in the `from` account.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

## Transaction Deduplication

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-37 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-37 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer_from`](#icrc37_transfer_from) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client has set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client has not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

### ICRC-37 Block Schema

ICRC-37 builds on the [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) specification for defining the format for storing transactions in blocks of the log of the ledger. ICRC-3 defines a generic, extensible, block schema that can be further instantiated in standards implementing ICRC-3. We next define the concrete block schema for ICRC-37 as extension of the ICRC-3 block schema.

All ICRC-37 batch methods result in one block per input of the batch. The blocks need not appear in the block log in the same relative sequence as the token ids appear in the vector of input token identifiers in order to not unnecessarily constrain the potential concurrency of an implementation. The block sequence corresponding to the token ids in the input can be interspersed with blocks from other (batch) methods executed by the ledger in an interleaved execution sequence. This allows for high-performance ledger implementations that can make asynchronous calls to other canisters in the scope of operations on tokens and process multiple batch update calls concurrently.

#### Generic ICRC-37 Block Schema

The following generic schema extends the generic schema of ICRC-3 with ICRC-37-specific elements and applies to all block defintions for blocks generated by ICRC-37.

// FIX What is `memo` at the top-level used for? Can we remove it? There is now a possibly available `memo` field defined in the `tx` record

1. it MUST contain a field `ts: Nat` which is the timestamp of when the block was added to the Ledger
2. it MAY contain a field `memo: Blob` if specified by the canister
3. its field `tx`
    1. MAY contain the `memo: Blob` field if specified by the user
    2. MAY contain the `ts: Nat` field if the user sets the `created_at_time` field in the request.

The `tx` field contains the transaction data as provided by the caller and is further refined for each the different update calls as specified below.

#### icrc37_approve_tokens Block Schema

1. the `tx.op` field MUST be `"37appr"`
2. it MUST contain a field `tx.tid: Nat`
3. it MUST contain a field `tx.from: Account`
4. it MUST contain a field `tx.spender: Account`
5. it MAY contain a field `tx.exp: Nat` if set by the user

Note that `tid` refers to the token id and `exp` to the expiry time of the approval. The names of the other fiels should speak for themselves.

#### icrc37_approve_collection Block Schema

1. the `tx.op` field MUST be `"37appr_coll"`
2. it MUST contain a field `tx.from: Account`
3. it MUST contain a field `tx.spender: Account`
4. it MAY contain a field `tx.exp: Nat` if set by the user

#### icrc37_revoke_token_approvals Block Schema

1. the `tx.op` field MUST be `"37revoke"`
2. it MUST contain a field `tx.tid: Nat`
3. it MUST contain a field `tx.from: Account`
4. it MAY contain a field `tx.spender: Account`

If the field `spender` is present, only the one token-level approval for this `spender`, `tid` (token id) and `from` is revoked, if the field `spender` is omitted, token-level approvals are revoked for all spenders for which approvals exist as specified by the remaining parameters `tid` (token id) and `from`. This helps reduce the log storage space required for handling explicit token-level revocations.

#### icrc37_revoke_collection_approvals Block Schema

1. the `tx.op` field MUST be `"37revoke_coll"`
2. it MUST contain a field `tx.from: Account`
3. it MAY contain a field `tx.spender: Account`

If the field `spender` is present, only the one collection-level approval for this `spender` and the given `from` value is revoked, if the field `spender` is omitted, collection-level approvals are revoked for all spenders for which approvals exist as specified by the remaining parameter `from`. This helps reduce the log storage space required for handling collection-level revocations.

#### icrc37_transfer_from Block Schema

1. the `tx.op` field MUST be `"37xfer"`
2. it MUST contain a field `tx.tid: Nat`
3. it MUST contain a field `tx.spender: Account`
4. it MUST contain a field `tx.from: Account`
5. it MUST contain a field `tx.to: Account`

## Extensions

If extension standards are used in the context of ICRC-37, those are listed with the according `supported_standards` method of the base standard that gets extended.

Conceptually, it would seem a good idea to expose a separate `supported_standards` method in ICRC-37 and list all ICRC-37 extensions with this method. However, it has been decided to expose only a single such method per ledger implementation, i.e., the `icrc7_supported_standards` method. The rationale behind not exposing a separate `icrc37_supported_standards` method that covers extensions of ICRC-37 is the following: In case of doing so, the caller would need to invoke multiple, in the general case a tree of, `supported_standards` methods, one per supported standard listed in the base method, in order to get the complete view of standards implemented by a given leder. By subsuming all supported standards in the base standard, the client can obtain this information with a single call. In most practical cases, the number of supported standards is expected to be easily manageable with this simpler approach.

