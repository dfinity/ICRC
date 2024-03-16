# ICRC-4: Batch Transfers for ICRC-1 Fungible Tokens

This document specifies the ICRC-4 standard for batch processing transfer transactions for fungible tokens compliant with the ICRC-1 standard. It is designed to optimize and reduce the cost of multiple transfers originating from a single principal and increase the throughput of ledger implementation.


|   Status   |
|:----------:|
| Draft (Pending Review and Acceptance) |

## Abstract

ICRC-4 extends the ICRC-1 standard to enable the transfer of tokens from multiple subaccounts owned by a principal to multiple recipient accounts in a single ledger call. This method significantly reduces the latency and cost traditionally associated with multi-account token transfers. It can also increase the throughput of a ledger that is otherwise constrained by the message limit of the subnet and the quota thereof of the ledger.

## Motivation

The primary motivation is to facilitate optimized batch transactions in DeFi applications where multi-party settlements or token distributions are common. Implementing a batch transaction method reduces the overhead of individual call cycle charges and minimizes latency associated with separate transaction submissions. This standard aims to align closely with the ICRC-7 batch patterns to ensure consistency across token standards on the Internet Computer.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Methods

### Generally-Applicable Specification

We next outline general aspects of the specification and behaviour of query and update calls defined in this standard. Those general aspects are not repeated with the specification of every method, but specified once for all query and update calls in this section.

#### Batch Update Methods

The methods that have at most one result per input value are modeled as *batch methods*, i.e., they operate on a vector of inputs and return a vector of outputs. The elements of the output are sorted in the same order as the elements of the input, meaning that the `i`-the element in the result is the response to the `i`-th element in the request. We call this property of the arguments "positional". The response may have fewer elements than the request in case processing has stopped through a batch processing error that prevents it from moving forward. In case of such batch processing error, the element which caused the batch processing to terminate receives an error response with the batch processing error. This element can not have a specific per-element error as it expresses the batch error. This element need not be the element with the highest index in the response as processing of requests can be concurrent in an implementation and any element earlier in the request may cause the batch processing failure.

The response of a batch method may be shorter than the request and then contain only responses to a prefix of the request vector. This happens when the processing is terminated due to an error. However, due to the ordering requirements of the response elements w.r.t. the request elements (positional arguments), the response must be contiguous, possibly containing `null` elements, i.e., it contains response elements to a contiguous prefix of the request vector.

The standard does not impose any constraints on aspects such as no duplicate token ids being contained in the elements of a request batch. Rather, each element is independent of the others and its execution may succeed or lead to an error. We do not impose any constraints on the sequence of processing of the elements of a batch in order to not have undue constraints on the implementation in terms of performance. A client SHOULD not assume any specific sequence of processing of batch elements. I.e., if a client intends to make dependent transactions, e.g., to move a token from its current subaccount to a specific "vendor subaccount" and from there transfer it to a customer, those two operations should be part of subsequent batches to assure both transfers to complete without assumptions on the implementation of the ledger.

Note that the items in a batch are processed independently of each other and processing can independently succeed or fail. This choice does not impose relevant constraints on the ledger implementation. The only constraint resulting from this is that the response must contain response items up to the largest element index processing of which has been initiated by the ledger, regardless of its result. The response items following this highest-index processed request item can be left out.

The API style we employ for batch APIs is simple, does not repeat request information in the response, and does not unnecessarily constrain the implementation, i.e., permits highly-concurrent implementations. On the client side it has no major drawbacks as it is straightforward to associate the corresponding request data with the responses by using positional alignment of the request and response vectors.

The guiding principle is to have all suitable update methods be batch methods, i.e., all update calls that have at most one response per request.

For batch update calls, each element in the request for which processing has been attempted, i.e., started, regardless of the success thereof, needs to contain a non-null response at the corresponding index. Elements for which processing has not been attempted, may contain a `null` response. The response vector may contain responses only to a prefix of the request vector, with further non-processed elements being omitted from the response.

Update calls, i.e., methods that modify the state of the ledger, always have responses that comprise transaction indices in the success case. Such a transaction index is an index into the chain of blocks containing the transaction history of this ledger. The details of how to access the transaction history of ICRC-7 ledgers is not part of the ICRC-7 standard, but will use the separately published [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) standard that specifies access to the block log of ICRC ledgers.

*Example*

Consider an input comprising the batch of transactions `A, B, C, D, E, F, G, H`, each of the items being one operation to perform and letters abstract the request items in the example.
```
Input : [A, B, C, D, E, F, G, H];
```
Depending on the concurrent execution of the individual operations, there may be different outcomes:

1. Assume an error that prevents further processing has occurred while processing `D`, with successfully processed `A`. Processing of `B` and `C` has not been initiated yet, therefore they receive `null` responses. Responses for `E..H` can be omitted as processing for `E..H` has not been initiated.
  * Output: `vec{opt variant{Ok=5}, null, null, opt variant{Err = variant {GenericBatchError =(...)}}};`
2. Assume an error that prevents further processing has occurred while processing `B` with successfully processed `A` and `D`, but processing for `C` has not been initiated. `A` and `D` receive a success response with their transaction id, `B` the batch error, and `C` is filled with a `null`. Responses for `E..H` can be omitted as processing for `E..H` has not been initiated.
  * Output: `vec{opt variant{Ok = 5}, opt variant{Err = variant {GenericBatchError =(...)}}, null , opt variant {Ok= 6}};`
3. Assume an error that prevents further processing has occurred while processing `A`, but processing of `B` and `H` has already been initiated and succeeded. The not-processed elements are filled up with `null` elements up to the rightmost processed element `H`.
  * Output: `vec {opt variant{Err = variant{GenericBatchError = (...)}}, opt variant{Ok =5}, null, null, null, null, null, opt variant{Ok=6}};`

#### Batch Query Methods

There are two different classes of query methods in terms of their API styles defined in this standard:
1. Query methods that have one response per request in the batch. For example, `icrc7_balance_of`, which receives a vector of token ids as input and each output element is the balance of the corresponding input. Those methods perfectly lend themselves for implementation with a batch API. Those queries have an analogous API style to batch update calls, with a difference in the meaning of `null` responses.
1. Query methods that may have multiple responses for an input element. An example is `icrc7_tokens_of`, which may have many response elements for an account. Those methods require pagination. Pagination is hard to combine with the batch API style and positional responses and they complicate both the API and the implementation. Thus, the guiding principle is that such methods be non-batch paginated methods, unless there is a strong reason for a deviation from this.

The class 1 of query calls above is handled with an API style that is *almost identical* to that of batch update calls as outlined above. The main and only difference is the meaning of `null` values. For update calls, a `null` response always means that processing of the corresponding request has not been initiated, e.g., after a batch error has occurred. For query calls, errors that prevent further processing of queries are not expected as queries are read operations that should not fail. For queries, `null` may be defined to have a specific meaning per query method and do not have the default semantics that the corresponding request has not been processed. Queries must process the complete contiguous request sequence from index 0 up to a given request element index and may not have further response elements after that index, but must, unlike update calls, not skip processing of some elements in the returned sequence. As queries are read-only operations that don't have the numerous failure modes of updates, this should not impose any undue constraints on an implementation.

#### Error Handling

It is recommended that neither query nor update calls trap unless completely unavoidable. The API is designed such that many error cases do not need to cause a trap, but can be communicated back to the client and the processing of large batches may be short-circuited to processing only a prefix thereof in case of an error.

For example, if a limit expressed through an `icrc1:max_...` metadata attribute is violated, e.g., the maximum batch size is exceeded and the response size would exceed the system's permitted maximum, the ledger should process only a prefix of the input and return a corresponding response vector with elements corresponding to this prefix. Only a prefix of the request being responded to means that the suffix of the request has not been processed and the processing of its elements has not even been attempted to be initiated.

#### Other Aspects

The size of responses to messages sent to a canister smart contract on the IC is constrained to a fixed constant size. For requests that could potentially result in larger response messages that breach this limit, the caller SHOULD ensure to constrain the input of the methods accordingly so that the response remains below the maximum allowed size, e.g., the caller should not query too many token ids in one batch call. To avoid hitting the size limit, the ledger may process only a prefix of the request. The ledger MAY make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

All update methods take `memo` parameters as input. An implementation of this standard SHOULD allow memos of at least 32 bytes in length for all methods.

Each used Candid type is only specified once in the standard text upon its first use and subsequent uses refer to this first use. Likewise, error responses may not be explained repeatedly for all methods after having been explained already upon their first use, so the reader may need to refer back to a previous use.

### icrc4_transfer_batch

Executes a batch of transfer operations from the sender's subaccounts to various recipient accounts.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner: principal; subaccount: opt Subaccount; };

type TransferArg = record {
    from_subaccount: opt Subaccount;
    to: Account;
    amount: nat;
    memo: opt blob;
    created_at_time: opt nat64;
    fee: opt nat
};

type TransferError = variant {
    BadBurn : record {min_burn_amount : nat};
    BadFee : record { expected_fee : nat };
    InsufficientFunds : record { balance : nat;};
    TooOld;
    GenericBatchError : record { error_code : nat; message : text };
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    TooManyRequests : record { limit: nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};

type TransferBatchResult = variant {
    Ok : nat; // Transaction index for successful transfers
    Err : TransferError;
};

/*
   input : [A, B, C, D, E, F, G];
   possible output: [opt #Ok(5), null,null,opt #Err(#GenericBatchError(...)]; 
                    [opt #Err(#GenericBatchError(...), opt #Ok(5), null, null,, null, null, opt #Ok(6)];
*/

// icrc4_transfer_batch method definition
icrc4_transfer_batch: (vec TransferArg) -> async vec (opt TransferBatchResult) query;
```

#### Preconditions for Transfer Batch

- The sender MUST have sufficient tokens in each specified subaccount to complete the corresponding transfers, including the fee per transfer.
- The number of transactions in the batch SHOULD NOT exceed the maximum batch size specified by the ledger.

#### Postconditions for Transfer Batch

- Upon successful processing, each transfer in the batch will be associated with a transaction index corresponding to its inclusion in the ledger.
- Transaction indices are returned in an `Ok` variant, while errors are captured in an `Err` variant. If an error occurs, the batch processing MAY stop.
- The Implementation MAY return unprocessed entries with a null item in the TransferBatchResult.
- The Implementation MUST maintain index integrity. use positional indexing, i.e., the `i`-th response element is the response to the `i`-th request element.
- Indexes with a null response MAY be truncated on the right side.

#### Fee Structure

- While individual fees for each transaction MAY be lower than a standard single transfer to reflect cost savings in batch processing, the practical aspects of how fees are collected, applied, and recorded results in charging the same fee as for ICRC-1 transfers SHOULD be followed.
- Since the duration of storage in the ledger contributes significantly to associated costs, the fee per transfer reflects the requirement for eternal ledger history.

### icrc4_balance_of_batch

Queries the balances of multiple accounts in a single call. This method is designed to optimize balance retrievals, aggregating multiple requests into one and reducing the overhead of individual queries.

```candid "Type definitions" +=
type BalanceQueryArgs = record {
    accounts: vec Account;
};

type BalanceQueryResult = vec nat; 

// icrc4_balance_of_batch method definition
icrc4_balance_of_batch: (BalanceQueryArgs) -> async (BalanceQueryResult) query;
```

#### Preconditions for the Balance Query Batch

- The request MUST include a list of accounts with at least one item, each specified by their owner principal and optionally a subaccount.
- The number of accounts in the query SHOULD NOT exceed the maximum batch size for balance queries specified by the ledger.

#### Postconditions for the Balance Query Batch

- The method returns a list of balances in order of request if the query succeeds.
- If too many requests are provided, the ledger MAY truncate the responses at the limit.
- Every balance included in the result corresponds directly to an account in the input query, maintaining the same order to facilitate mapping between requested accounts and their balances.

### icrc4_maximum_update_batch_size

Queries the metadata entry for the maximum number of batch transactions that can be included in one batch call.

```candid "Type definitions" +=
icrc4_maximum_batch_size: () -> async opt Nat query;
```

#### Query Assumptions and Prerequisites

- The canister implementation has a maximum batch threshold

#### Expected Response

- If the canister does not have a maximum batch threshold set, the response should be null.
- If the canister does have a maximum batch threshold it should be returned as a nat.

### icrc4_maximum_query_batch_size

Queries the metadata entry for the maximum number of balance queries that can be included in one batch call.

```candid "Type definitions" +=
icrc4_maximum_batch_size: () -> async opt Nat query;
```

#### Query Assumptions and Prerequisites

- The canister implementation has a maximum balance threshold.

#### Expected Response

- If the canister does not have a maximum balance threshold set, the response should be `null`.
- If the canister does have a maximum balance threshold, it should be returned as a `nat`.

## Batch Transfer Metadata

Ledgers supporting ICRC-4 SHOULD provide metadata via the `icrc1_metadata()` function including maximum batch and balance query sizes.

### Standard metadata entries
| Key | Semantics | Example value
| --- | ------------- | --------- |
| `icrc4:maximum_batch_size` | The IC has a ~2MB limit on ingress and responses, therefore an implementor must provide guidance on the maximum number of transactions that can be handled in one call. | `variant { Nat = 200 }` | 
| `icrc4:maximum_balance_size` | The IC has a ~2MB limit on ingress and responses, therefore an implementor must provide guidance on the maximum number of balance inquiries that can be handled in one call. | `variant { Nat = 200 }` | 

## Updates to Transfer Block Schema

ICRC-4 does not make updates to the ICRC-1 block schema for transfer, mint, or burn. Each successful transaction in a transfer request MUST get its own entry in the transaction log.

## Transaction deduplication <span id="transaction_deduplication"></span>

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-4 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-4 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer_batch`](#transfer_batch_method) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client did not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

### Supported Standards

All ledgers implementing ICRC-4 MUST include the standard in the list returned by the `icrc1_supported_standards` method. Furthermore, ICRC-4 aims to adhere to the patterns established by ICRC-7 for batch processing, providing a consistent interface for both fungible and non-fungible token transfers within the Internet Computer ecosystem.

## Rationale and Backwards Compatibility

ICRC-4 introduces no known backward compatibility issues with existing standards. It is designed as an extension of ICRC-1 and preserves the principles of individual transfer transactions.

## Future Considerations

As the Internet Computer ecosystem evolves, there may be further optimizations or adjustments to the batch transfer process. These changes will be proposed, reviewed, and integrated into subsequent ICRC proposals as necessary.

## Implementing ICRC-4

Developers and ledger maintainers are encouraged to adopt the ICRC-4 standard and provide feedback for future revisions. A reference implementation and developer guide will be made available once the standard is accepted and ratified.

