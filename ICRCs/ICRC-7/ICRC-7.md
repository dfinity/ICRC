|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai), Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer), Thomas (@sea-snake), Moritz Fuller (@letmejustputthishere), Matthew Harmon|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Minimal Non-Fungible Token (NFT) Standard

ICRC-7 is the minimal standard for the implementation of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

A token ledger implementation following this standard hosts an *NFT collection* (*collection*), which is a set of NFTs.

ICRC-7 does not handle approval-related operations such as `approve` and `transfer_from` itself. Those operations are specified by ICRC-30 which extends ICRC-7 with approval semantics.

## Data Representation

This section specifies the core principles of data representation used in this standard.

### Accounts

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore, an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the *default account* of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount };
```

The canonical textual representation of the account follows the [definition in ICRC-1](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md). ICRC-7 accounts have the same structure and follow the same overall principles as ICRC-1 accounts.

ICRC-7 views the ICRC-1 `Account` as a primary concept, meaning that operations like transfers (or approvals as defined in ICRC-30) always refer to the full account and not only the principal part thereof. Thus, some methods comprise an extra optional `from_subaccount` or `spender_subaccount` parameter that together with the caller form an account to perform the respective operation on. Leaving such subaccount parameter `null` always has the semantics of referring to the default subaccount comprised of all zeroes.

### Token Identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value and thus unbounded in length. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids, which is, for example, important for making other NFT standards that use strings as token identifiers compatible with ICRC-7.

## Methods

### Generally-Applicable Specification

We next outline general aspects of the specification and behaviour of query and update calls defined in this standard. Those general aspects are not repeated with the specification of every method, but specified once for all query and update calls in this section.

#### Batch Methods

The methods, both queries and update calls, that operate on token ids as inputs are *batch methods* that can receive vectors of token ids, i.e., batches of token ids, as input. The batch methods in this standard perform the same operation with the same parameters on a batch of token ids, thus the operations in the batch are closely related and are not independent operations. Some people refer to this modus operandi as bulk processing, but we prefer to use the more general term of batch processing also for this case. The batch methods must be used also for the common case of operations on single token ids (the non-batch case) — no separate methods are provided for this purpose. Most batch methods provide a vector of records comprising a token id and a response for the token id as output. Unless specified explicitly otherwise, the ordering of response elements for batch calls is not defined, i.e., is arbitrary. For batch calls, every distinct token id of the input MUST occur in at least one response element, where a response can be a success or error variant.

**Batch Update Methods**

If the input contains duplicate token ids for an update call, the ledger MUST trap. For update batch calls, the method's logic MUST be executed on all provided token ids or none, but it need not be successful for all, in which case error responses are returned for token ids for which exection was not successful.

**Batch Query Methods**

Duplicate token ids in batch query calls may result in duplicate token ids in the response or may be deduplicated at the discretion of the ledger. The lenght of the response vector may be shorter than that of the input vector in batch query calls, e.g., in case of duplicate token ids in the input and a deduplication being performed by the method's implementation logic or in case of the token not existing.

#### State-Changing Methods

Methods that modify the state of the ledger have responses that comprise transaction indices as part of the response in the success case. Such a transaction index is an index into the chain of blocks containing the transaction history of this ledger. The details of how to access the transaction history is not part of the ICRC-7 standard, but will be published as a separate standard, similar to how ICRC-3 specifies access to ICRC-1 and ICRC-2 historic blocks.

#### Error Handling

All update methods have error variants defined for their responses that cover the error cases for the respective call. Batch update methods expose a top-level error for handling general errors applicable to the batch itself and a possible error per element in the batch. Query methods do not have error variants defined in order to keep their API easy to use.

Both query and update calls can trap in specific error cases instead of returning an error. The general principle is that a ledger SHOULD, or, depending on the recoverability, MUST, trap in case of an error that is not related to a specific token and thus can be addressed by the error response for the token, but is a general error related to the call. For example, a ledger SHOULD trap if a limit expressed through an `icrc7:max_...` metadata attribute is violated, e.g., the maximum batch size is exceeded. The ledger MUST trap in case of an error that it cannot recover from, i.e., it cannot execute the call and perform all state changes according to specification.

#### Other Aspects

The size of responses to messages sent to a canister smart contract on the IC is constrained to a fixed constant size. For requests that could potentially result in larger response messages that breach this limit, the caller SHOULD ensure to constrain the input of the methods accordingly so that the response remains below the maximum allowed size, e.g., should not query too many token ids in one batch call. If the size limit of a response is hit, the ledger canister MUST trap. The ledger SHOULD make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

All update methods take a `memo` parameter as input. An implementation of this standard SHOULD allow memos of at least 32 bytes in length for all methods.

Each defined Candid type is only specified once in the standard text upon its first use. Likewise, error responses are not always explained repeatedly for all methods after having been explained already upon their first use.

### icrc7_collection_metadata

Returns all the collection-level metadata of the NFT collection in a single query.

The data model for metadata is based on the generic `Value` type which allows for encoding arbitrarily complex data for each metadata attribute. The metadata attributes are expressed as `(text, value)` pairs where the first element is the name of the metadata attribute and the second element the corresponding value expressed through the `Value` type.

Analogous to [ICRC-1 metadata](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-1#metadata), metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons. The namespace `icrc7` is reserved for keys defined in the ICRC-7 standard.

The set of elements contained in a specific ledger's metadata depends on the ledger implementation, the list below establishes the currently defined fields.

The following metadata fields are defined by ICRC-7, starting with general collection-specific metadata fields:
  * `icrc7:symbol` of type `text`: The token symbol. Token symbols are often represented similar to [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)) currency codes. When present, should be the same as the result of the [`icrc7_symbol`](#icrc7_symbol) query call.
  * `icrc7:name` of type `text`: The name of the token. Should be the same as the result of the [`icrc7_name`](#icrc7_name) query call.
  * `icrc7:description` of type `text` (optional): A textual description of the token. When present, should be the same as the result of the [`icrc7_description`](#icrc7_description) query call.
  * `icrc7:logo` of type `text` (optional): The URL of the token logo. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself. When present, should be the same as the result of the [`icrc7_logo`](#icrc7_logo) query call.
  * `icrc7:total_supply` of type `nat`: The current total supply of the token, i.e., the number of tokens in existence. Should be the same as the result of the [`icrc7_total_supply`](#icrc7_total_supply) query call.
  * `icrc7:supply_cap` of type `nat` (optional): The current maximum supply for the token beyond which minting new tokens is not possible. When present, should be the same as the result of the [`icrc7_supply_cap`](#icrc7_supply_cap) query call.

The following are the more technical, implementation-oriented, metadata elements:
  * `icrc7:max_query_batch_size` of type `nat` (optional): The maximum batch size for batch query calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_query_batch_size`](#icrc7_max_query_batch_size) query call.
  * `icrc7:max_update_batch_size` of type `nat` (optional): The maximum batch size for batch update calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_update_batch_size`](#icrc7_max_update_batch_size) query call.
  * `icrc7:default_take_value` of type `nat` (optional): The default value this ledger uses for the `take` pagination parameter which is used in some queries. When present, should be the same as the result of the [`icrc7_default_take_value`](#icrc7_default_take_value) query call.
  * `icrc7:max_take_value` of type `nat` (optional): The maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated queries the ledger exposes. When present, should be the same as the result of the [`icrc7_max_take_value`](#icrc7_max_take_value) query call.
  * `icrc7:max_memo_size` of type `nat` (optional): The maximum size of `memo`s as supported by an implementation. When present, should be the same as the result of the [`icrc7_max_memo_size`](#icrc7_max_memo_size) query call.

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
icrc7_collection_metadata : () -> (vec record { text; Value } ) query;
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

### icrc7_max_query_batch_size

Returns the maximum batch size for batch query calls this ledger implementation supports.

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

### icrc7_max_memo_size

Returns the maximum size of `memo`s as supported by an implementation.

```candid "Methods" +=
icrc7_max_memo_size : () -> (opt nat) query;
```

### icrc7_token_metadata

Returns the token metadata for `token_ids`, a list of token ids. Each tuple in the response vector comprises a token id as first element and the metadata corresponding to this token expressed as an optional record comprising `text` and `Value` pairs expressing the token metadata as second element. In case a token does not exist, no element corresponding to it is returned in the response. If a token does not have metadata, its associated metadata vector is the empty vector.

ICRC-7 does not specify the representation of token metadata any further than that it is represented in a generic manner as a vector of `(text, Value)`-pairs. This is left to future standards, the collections, or the implementations in order to not constrain the utility and applicability of this standard.

> [!NOTE]
> Encoding of types not contained in the `Value` type SHOULD be handled according to best practices as put forth in the context of the ICRC-3 standard.

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
    -> (vec record { token_id : nat; metadata: vec record { text; Value } }) query;
```

### icrc7_owner_of

Returns the owner `Account` of each token in a list `token_ids` of token ids. The response elements are sorted following an ordering depending on the ledger implementation.

Note that tokens for which an ICRC-1 account cannot be found have a `null` response. This can for example be the case for a ledger that has originally used a different token standard, e.g., based on the ICP account model, and tokens of this ledger have not been fully migrated yet to ICRC-7.

```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat)
    -> (vec record { token_id : nat; account : opt Account }) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument, i.e., the number of tokens held by the account. For a non-existing account, the value `0` is returned.

```candid "Methods" +=
icrc7_balance_of : (account : Account) -> (nat) query;
```

### icrc7_tokens

Returns the list of tokens in this ledger, sorted by their token id.

The result is paginated and pagination is controlled via the `prev` and `take` parameters: The response to a request results in at most `take` many token ids, starting with the next id following `prev`. The token ids in the response are sorted in any consistent sorting order used by the ledger. If `prev` is `null`, the response elements start with the smallest ids in the ledger according to the sorting order. If the response to a call with a non-null `prev` value contains no token ids, there are no further tokens following `prev`. If the response to a call contains fewer token ids than the provided or default `take` value, there are no further tokens in the ledger following the largest returned token id. If `take` is omitted, the ledger's default `take` value as specified through `icrc7:default_take_value` is assumed.

For retrieving all tokens of the ledger, the pagination API is used such that the first call sets `prev = null` and specifies a suitable `take` value. Then, the method is called repeatedly such that the greatest token id of the previous response is used as `prev` value for the next call to the method. The method is called in this manner as long as the response comprises `take` many elements if take has been specified or `icrc7:default_take_value` many elements if `take` has not been specified. Using this approach, all tokens can be enumerated in ascending order, provided the ledger state does not change between the method calls.

Each invocation is executed on the current memory state of the ledger. I.e., it is not possible to enumerate the exact list of token ids of the ledger at a given time or of a "snapshot" of the ledger state. Rather, the ledger state can change between the multiple calls required to enumerate all the tokens.

```candid "Methods" +=
icrc7_tokens : (prev : opt nat, take : opt nat)
    -> (vec nat) query;
```

### icrc7_tokens_of

Returns a vector of `token_id`s of all tokens held by `account`, sorted by `token_id`.  The token ids in the response are sorted in any consistent sorting order used by the ledger. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination.

```candid "Methods" +=
icrc7_tokens_of : (account : Account, prev : opt nat, take : opt nat)
    -> (vec nat) query;
```

### icrc7_transfer

Transfers one or more tokens from the account defined by the caller principal and `subaccount` to the `to` account. The transfer can only be initiated by the holder of the tokens.

The response is a vector of records each comprising a `token_id` and a corresponding `transfer_result` indicating success or error. In the success case, the `Ok` variant indicates the transaction index of the transfer, in the error case, the `Err` variant indicates the error through `TransferError`.

The ledger returns an `InvalidRecipient` error in case `to` equals `from`.

A transfer clears all active token-level approvals for the successfully transferred tokens. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals.

```candid "Type definitions" +=
TransferArgs = record {
    subaccount: opt blob; // the subaccount of the caller (used to identify the spender)
    to : Account;
    token_ids : vec nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferBatchError = variant {
    InvalidRecipient;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
};

type TransferError = variant {
    NonExistingTokenId;
    Unauthorized;
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (TransferArgs)
    -> (variant { Ok : vec record { token_id : nat; transfer_result : variant { Ok : nat; Err : TransferError } };
                  Err : TransferBatchError });
```

If the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response. This may be the case if the token is not held in the subaccount specified in the `from` account.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

> [!NOTE] Note that multiple concurrently executing batch transfers triggered by method invocations can lead to an interleaved execution of the corresponding sequences of token transfers.

### icrc7_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc7_supported_standards : () -> (vec record { name : text; url : text }) query;
```

The result of the call should always have at least one entry:

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7"; }
```

## ICRC-7 Block Schema

ICRC-7 builds on the ICRC-3 specification for defining the format for storing transactions in blocks of the log of the ledger. ICRC-3 defines a generic, extensible, block schema that can be further instantiated in standards implementing ICRC-3. We next define the concrete block schema for ICRC-7 as extension of the ICRC-3 block schema.

### Generic ICRC-7 Block Schema

1. it MUST contain a field `ts: Nat` which is the timestamp of when the block was added to the Ledger
2. its field `tx`
    1. CAN contain the `memo: Blob` field if specified by the user
    2. CAN contain the `ts: Nat` field if the user sets the `created_at_time` field in the request.

### Mint Block Schema

1. the `tx.op` field MUST be `"7mint"`
2. it MUST contain a field `tx.tid: Nat`
3. it CAN contain a field `tx.from: Account`
4. it MUST contain a field `tx.to: Account`
5. it MUST contain a field `tx.meta: Value`

Note that `tid` refers to the token id. The size of the `meta` field expressing the token metadata must be less than the maximum size permitted for inter-canister calls. If the metadata is sufficiently small, it is recommended to put the full metadata into the `tx.meta` field, if the metadata is too large, it is recommended to add a hash of the metadata to the `meta` field.

### Burn Block Schema

1. the `tx.op` field MUST be `"7burn"`
2. it MUST contain a field `tx.tid: Nat`
3. it MUST contain a field `tx.from: Account`
4. it CAN contain a field `tx.to: Account`

### icrc7_transfer Block Schema

1. the `tx.op` field MUST be `"7xfer"`
2. it MUST contain a field `tx.tid: Nat`
3. it MUST contain a field `tx.from: Account`
4. it MUST contain a field `tx.to: Account`

As `icrc7_transfer` is a batch method, it results in one block per `token_id` in the batch. The blocks MUST appear in the block log in the same sequence as the token ids are listed in the `token_ids` vector. Blocks from one batch transfer invocation can be interleaved from other such method calls.

## Migration Path for Ledgers Using ICP AccountId

For historical reasons, multiple NFT standards, such as the EXT standard, use the ICP `AccountIdentifier` or `AccountId` (a hash of the principal and subaccount) instead of the ICRC-1 `Account` (a pair of principal and subaccount) to store the owners. Since the ICP `AccountId` can be calculated from an ICRC-1 `Account`, but computability does not hold in the inverse direction, there is no way for a ledger implementing ICP `AccountId` to display `icrc7_owner_of` data.

This standard does not mandate any provisions regarding the handling of tokens managed through the `AccountId` regime and leaves this open to a future ICRC standard that ledgers may opt to implement or ledger-specific implementations.

Ledgers using the ICP `AccountId` can provide a `null` response for a token that has not yet been migrated to an ICRC-1 account for the `icrc7_owner_of` query. The ledger implementation may offer an additional method to allow clients to obtain further information on this token, e.g., whether it is a token based on the ICP `AccountId`. `AccountId`-based ledgers that want to support ICRC-7 need to implement a strategy to become ICRC-7 compliant, e.g., by requiring all users to call a migration endpoint to migrate their tokens to an ICRC-1-based representation. It is acceptable behaviour to not consider not-yet-migrated tokens of such ledgers in responses as they conceptually don't count against the total ICRC-7 supply of the ledger before being migrated.

Different approaches for migration are feasible and the choice of migration approach is left to the ledger implementation and not mandated in this standard. ICRC standards may emerge in the future for addressing the migration from previous NFT standards to ICRC-7.

## Extensions

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  * Reliable transaction notifications for smart contracts.
  * The block structure and the interface for fetching blocks.
  * Pre-signed transactions.

The standard defines the `icrc7_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-42"` or `"DIP-20"`) implemented by the ledger as well as URLs.

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

The ledger SHOULD use the following algorithm for transaction deduplication if the client has set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client has not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

## Security Considerations

This section highlights some selected areas crucial for security regarding the implementation of ledgers following this standard and Web applications using ledgers following this standard. Note that this is not exhaustive by any means, but rather points out a few selected important areas.

### Protection Against Denial of Service Attacks

It is strongly recommended that implementations of this standard take steps towards protecting against Denial of Service (DoS) attacks. Some non-exhaustive list of examples for recommended mitigations are given next:
  * Enforcing limits, such as the number of active approvals per token for token-level approvals or per principal for collection-level approvals, to constrain the state size of the ledger. Examples of such limits are given in this standard through various metadata attributes.
  * Enforcing rate limits, such as the number of transactions such as approvals or approval revocations, can be performed on a per-token and per-principal basis to constrain the size of the transaction log for the ledger.
  * The execution of operations such as approving collections and revoking such approvals could be constrained to parties who own at least one token on a ledger. This helps prevent DoS by attackers who create a large number of principals and perform such operations without holding tokens.

### Protection Against Attacks Against Web Applications

We strongly advise developers who display untrusted user-generated data like images (e.g., the token logo or images referenced from NFT metadata) or strings in a Web application to follow Web application security best practices to avoid attacks such as XSS and CSRF resulting from malicious content provided by a ledger. As one particular example, images in the SVG format provide potential for attacks if used improperly. See, for example, the OWASP guidelines for protecting against [XSS](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) or [CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html). The above examples are only selected examples and by no means provide an exhaustive view of security issues.


<!--
```candid ICRC-1.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->
