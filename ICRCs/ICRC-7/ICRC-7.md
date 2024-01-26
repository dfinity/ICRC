|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|7|Minimal Non-Fungible Token (NFT) Standard|Ben Zhai (@benjizhai), Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer), Thomas (@sea-snake), Moritz Fuller (@letmejustputthishere), Matthew Harmon|https://github.com/dfinity/ICRC/issues/7|Draft|Standards Track||2023-01-31|



# ICRC-7: Minimal Non-Fungible Token (NFT) Standard

ICRC-7 is the minimal standard for the implementation of Non-Fungible Tokens (NFTs) on the [Internet Computer](https://internetcomputer.org).

A token ledger implementation following this standard hosts an *NFT collection* (*collection*), which is a set of NFTs.

ICRC-7 does not handle approval-related operations such as `approve` and `transfer_from` itself. Those operations are specified by ICRC-37 which extends ICRC-7 with approval semantics.

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

ICRC-7 views the ICRC-1 `Account` as a primary concept, meaning that operations like transfers (or approvals as defined in ICRC-37) always refer to the full account and not only the principal part thereof. Thus, some methods comprise an extra optional `from_subaccount` or `spender_subaccount` parameter that together with the caller form an account to perform the respective operation on. Leaving such subaccount parameter `null` always has the semantics of referring to the default subaccount comprised of all zeroes.

### Token Identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value and thus unbounded in length. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids, which is, for example, important for making other NFT standards that use strings as token identifiers compatible with ICRC-7.

## Methods

### Generally-Applicable Specification

We next outline general aspects of the specification and behaviour of query and update calls defined in this standard. Those general aspects are not repeated with the specification of every method, but specified once for all query and update calls in this section.

#### Batch Update Methods

The methods that have at most one result per input value are modeled as *batch methods*, i.e., they operate on a vector of inputs and return a vector of outputs. The elements of the output are sorted in the same order as the elements of the input, meaning that the `i`-the element in the result is the reponse to the `i`-th element in the request. We call this property of the arguments "positional". The response may have fewer elements than the request in case processing has stopped through a batch processing error that prevents it from moving forward. In case of such batch processing error, the element which caused the batch processing to terminate receives an error response with the batch processing error. This element can not have a specific per-element error as it expresses the batch error. This element need not be the element with the highest index in the response as processing of requests can be concurrent in an implementation and any element earlier in the request may cause the batch processing failure.

The response of a batch method may be shorter than the request and then contain only responses to a prefix of the request vector. This happens when the processing is terminated due to an error. However, due to the ordering requirements of the response elements w.r.t. the request elements (positional arguments), the response must be contiguous, possibly containing `null` elements, i.e., it contains response elements to a contiguous prefix of the request vector.

The standard does not impose any constraints on aspects such as no duplicate token ids being contained in the elements of a request batch. Rather, each element is independent of the others and its execution may succeed or lead to an error. We do not impose any constraints on the sequence of processing of the elements of a batch in order to not have undue constraints on the implementation in terms of performance. A client SHOULD not assume any specific sequence of processing of batch elements. I.e., if a client intends to make dependent transactions, e.g., to move a token from its current subaccount to a specific "vendor subaccount" and from there transfer it to a customer, those two operations should be part of subsequent batches to assure both transfers to complete without assumptions on the implementation of the ledger.

Note that the items in a batch are processed independently of each other and processing can independently succeed or fail. This choice does not impose relevant constraints on the ledger implementation. The only constraint resulting from this is that the response must contain response items up to the largest element index processing of which has been initiated by the ledger, regardless of its result. The response items following this highest-index processed request item can be left out.

The API style we employ for batch APIs is simple, does not repeat request information in the response, and does not unnecessarily constrain the implementation, i.e., permits highly-concurrent implementations. On the client side it has no major drawbacks as it is straightforward to associated the corresponding request data with the responses by using positional alignment of the request and response vectors.

The guiding principle is to have all suitable update methods be batch methods, i.e., all update calls that have at most one response per request.

For batch update calls, each element in the request for which processing has been attempted, i.e., started, regardless of the success thereof, needs to contain a non-null response at the corresponding index. Elements for which processing has not been attempted, may contain a `null` response. The response vector may contain responses only to a prefix of the request vector, with further non-processed elements being omitted from the response.

Update calls, i.e., methods that modify the state of the ledger, always have responses that comprise transaction indices in the success case. Such a transaction index is an index into the chain of blocks containing the transaction history of this ledger. The details of how to access the transaction history of ICRC-7 ledgers is not part of the ICRC-7 standard, but will use the separately published [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) standard that specifies access to the block log of ICRC ledgers.

*Example*

Consider an input comprising the batch of transactions `A, B, C, D, E, F, G, H`, each of the items being one operation to perform and letters abstract the request items in the example.
```
Input : [A, B, C, D, E, F, G, H];
```
Depending on the concurrent execution of the individual operations, there may be different outcomes:

1. Assume an error that prevents further processing has occurred while processing `D`, with successfully processed `A`. Processing of `B` and `C` has not been initated yet, therefore they receive `null` responses.
  * Output: `[opt #Ok(5), null ,null, opt #Err(#GenericBatchError(...)];`
2. Assume an error that prevents further processing has occurred while processing `B` with successfully processed `A` and `D`, but processing for `C` has not been initiated. `A` and `D` receive a success response with their transaction id, `B` the batch error, and `C` is filled with a `null`:
  * Output: `[opt #Ok(5), opt #Err(#GenericBatchError(...), null , opt #Ok(6)];`
3. Assume an error that prevents further processing has occurred while processing `A`, but processing of `B` and `H` has already been initiated and succeeds. The not-processed elements are filled up with `null` elements up to the rightmost processed element `H`.
  * Output: `[opt #Err(#GenericBatchError(...), opt #Ok(5), null, null, null, null, null, opt #Ok(6)];`

#### Batch Query Methods

There are two different classes of query methods in terms of their API styles defined in this standard:
1. Query methods that have (at most) one response per request in the batch. For example, `icrc7_balance_of`, which receives a vector of token ids as input and each output element is the balance of the corresponding input. Those methods perfectly lend themselves for implementation with a batch API. Those queries have an analogous API style as batch update calls.
1. Query methods that may have multiple responses for an input element. An example is `icrc7_tokens_of`, which may have many response elements for an account. Those methods require pagination. Pagination is hard to combine with the batch API style of the reponses corresponding to the requests by index of the respective vectors. Thus, the guiding principle is that such methods be non-batch paginated methods, unless there is a strong reason for a deviation from this.

The class 1 of query calls above is handled with an API style that is *almost identical* to that of batch update calls. The main and only difference is the meaning of `null` values. For update calls, a `null` response means that processing has not been initiated, e.g., after a batch error has occurred. For query calls, errors that prevent further processing of queries are not expected as queries are read operations that should not fail. For queries, `null` may be defined to have a specific meaning per query. Queries must process the complete contiguous request sequence from index 0 up to a given request element index and may not have further response elements, but must, unlike queries, not skip processing of some elements. As queries are read-only operations that don't have the numerous failure modes of updates, this should not impose any undue constraints on an implementation.

#### Error Handling

It is recommended that neither query nor update calls trap unless completely unavoidable. The API is designed such that errors do not need to cause a trap, but can be communicated back to the client and the processing of large batches may be short-circuited to processing only a prefix thereof.

For example, if a limit expressed through an `icrc7:max_...` metadata attribute is violated, e.g., the maximum batch size is exceeded and the response size would exceed the system's permitted maximum, the ledger should process only a prefix of the input and return a corresponding response vector with elements corresponding to this prefix. Only a prefix of the request being responded to means that the suffix of the request has not been processed and the processing of its elements has not even been attempted to be initiated.

#### Other Aspects

The size of responses to messages sent to a canister smart contract on the IC is constrained to a fixed constant size. For requests that could potentially result in larger response messages that breach this limit, the caller SHOULD ensure to constrain the input of the methods accordingly so that the response remains below the maximum allowed size, e.g., the caller should not query too many token ids in one batch call. To avoid hitting the size limit, the ledger may process only a prefix of the request. The ledger MAY make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

All update methods take `memo` parameters as input. An implementation of this standard SHOULD allow memos of at least 32 bytes in length for all methods.

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

Note that if `icrc7_max...` limits specified through metadata are violated in a query call by providing larger argument lists or resulting in larger responses than permitted, the canister SHOULD return a response only to a prefix of the request items.

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
    -> (vec opt metadata: vec record { text; Value }) query;
```

### icrc7_owner_of

Returns the owner `Account` of each token in a list `token_ids` of token ids. The ordering of the response elements corresponds to that of the request elements.

Note that tokens for which an ICRC-1 account cannot be found have a `null` response. This can for example be the case for a ledger that has originally used a different token standard, e.g., based on the ICP account model, and tokens of this ledger have not been fully migrated yet to ICRC-7.

```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat)
    -> (vec opt account : Account) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument, i.e., the number of tokens held by the account. For a non-existing account, the value `0` is returned.

// FIX Do we need to allow a response to be `null` with the new batch API semantics? E.g., processing needs to hold and responses need to be filled up with `null` values; if so, this clashes with `null` semantics for various queries; not allowing `null` values seems to be cleaner, but deviates from ghow update calls are handled

```candid "Methods" +=
icrc7_balance_of : (vec account : Account) -> (nat) query;
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

Performs a batch of transfers of tokens. Each transfer transfers a token `token_id` from the account defined by the caller principal and the specified `subaccount` to the `to` account. A `memo` and `created_at_time` can be given optionally. The transfer can only be initiated by the holder of the tokens.

The response is a vector of `TransferResult` records, whose sequence correspond to the sequence of `TransferArg`s in the request, i.e., the `i`-th `TransferResult` in the response is the response to the `i`-th `TransferArg` in the request. Each `TransferResult` comprises an `Ok` variant with the transaction index of the transfer in the success case or an `Err` variant with a `TransferError` in the error case.

A transfer clears all active token-level approvals for a successfully transferred token. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals. This clearing does not create an ICRC-3 block in the transaction log, but it is implied by the transfer block in the log.

```candid "Type definitions" +=
TransferArg = record {
    subaccount: opt blob; // the subaccount of the caller (used to identify the spender), null means the default all-zero subaccount
    to : Account;
    token_id : vec nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferResult = variant {
    Ok : nat; // Transaction indices for successful transfers
    Err : TransferError;
};

type TransferError = variant {
    // per-transfer errors
    NonExistingTokenId;
    InvalidRecipient;
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
    // batch errors
    BatchTermination;
    GenericBatchError : record { error_code : nat; message : text };
};

```

```candid "Methods" +=
icrc7_transfer : (vec TransferArg) -> (vec opt TransferResult);
```

The ledger returns an `InvalidRecipient` error in case `to` equals `from` for a `TransferArg`.

If the caller principal is not permitted to act on a token id, then the corresponding request item receives the `Unauthorized` error response. This may be the case if the token is not held in the subaccount specified in the `from` account.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

> [!NOTE] Note that multiple concurrently executing batch transfers triggered by method invocations can lead to an interleaved execution of the corresponding sequences of token transfers.

> [!NOTE] Note further that deduplication is performed independently on the different items of the batch.

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
    1. CAN contain a field `memo: Blob` if specified by the user
    2. CAN contain a field `ts: Nat` if the user sets the `created_at_time` field in the request.

### Mint Block Schema

1. the `tx.op` field MUST be `"7mint"`
2. it MUST contain a field `tx.tid: Nat`
3. it CAN contain a field `tx.from: Account`
4. it MUST contain a field `tx.to: Account`
5. it MUST contain a field `tx.meta: Value`

Note that `tid` refers to the token id. The size of the `meta` field expressing the token metadata must be less than the maximum size permitted for inter-canister calls. If the metadata is sufficiently small, it is recommended to add the full metadata into the `tx.meta` field, if the metadata is too large, it is recommended to add a hash of the metadata to the `meta` field.

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

As `icrc7_transfer` is a batch method, it results in one block per `token_id` in the batch. The blocks MUST appear in the block log in the same sequence as the token ids are listed in the `token_ids` vector. Blocks from one batch transfer invocation can be interleaved from other such method calls. // t.b.d. for the new API

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
