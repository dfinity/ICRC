# ICRC-4: Batch Transfers for ICRC-1 Fungible Tokens

This document specifies the ICRC-4 standard for batch processing transfer transactions for fungible tokens compliant with the ICRC-1 standard. It is designed to optimize and reduce the cost of multiple transfers originating from a single principal.

|   Status   |
|:----------:|
| Draft (Pending Review and Acceptance) |

## Abstract

ICRC-4 extends the ICRC-1 standard to enable the transfer of tokens from multiple subaccounts owned by a principal to multiple recipient accounts in a single ledger call. This method significantly reduces the latency and cost traditionally associated with multi-account token transfers.

## Motivation

The primary motivation is to facilitate optimized batch transactions in DeFi applications where multi-party settlements or token distributions are common. Implementing a batch transaction method reduces the overhead of individual call cycle charges and minimizes latency associated with separate transaction submissions. This standard aims to align closely with the ICRC-7 batch patterns to ensure consistency across token standards on the Internet Computer.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Methods

### icrc4_transfer_batch

Executes a batch of transfer operations from the sender's subaccounts to various recipient accounts.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner: principal; subaccount: opt Subaccount; };

type BatchTransferArg = record {
    from_subaccount: opt Subaccount;
    to: Account;
    amount: nat;
    fee: ?nat
};

type TransferError = variant {
    BadFee : record { expected_fee : nat };
    InsufficientFunds : record { balance : nat;};
    GenericError : record { error_code : nat; message : text };
};

type TransferBatchArgs = record {
    transfers: vec BatchTransferArg;
    memo: opt blob;      // A single memo for batch-level deduplication
    created_at_time: opt nat64;
};

type TransferBatchError = variant {
    TemporarilyUnavailable;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat }; //todo: should this be different for batch since the items can go into many transactions
    GenericError : record { error_code : nat; message : text };
};

type TransferBatchResult = variant {
    Ok : vec record {
      transfer : BatchTransferArg; //todo: do we need this?  Can we leave out memo? or is it helpful
      transfer_result : variant {
        Ok : nat; // Transaction indices for successful transfers
        Err : TransferError
      };   
    Err : TransferBatchError;
};

// icrc4_transfer_batch method definition
icrc4_transfer_batch: (TransferBatchArgs) -> (TransferBatchResult);
```

#### Preconditions for Transfer Batch

- The sender MUST have sufficient tokens in each specified subaccount to complete the corresponding transfer, including the fee per transfer.
- The number of transactions in the batch SHOULD NOT exceed the maximum batch size specified by the ledger.

#### Postconditions for Transfer Batch

- Upon successful processing, each transfer in the batch will be associated with a transaction index corresponding to its inclusion in the ledger.
- Transaction indices are returned in an `Ok` variant, while errors are captured in an `Err` variant. If an error occurs, the batch processing MAY stop.

#### Fee Structure

- While individual fees for each transaction could be lower than a standard single transfer to reflect cost savings in batch processing, the practical aspects of how fees are collected, applied, and recorded in the ledger must be straightforward.
- Each transfer within the batch MAY incur an individual fee as specified in the ledger's batch metadata. There is NO separate batch-level fee.
- Since the length of storage in the ledger contributes significantly to associated costs, the fee per transfer reflects the requirement for immortal ledger history.

### icrc4_balance_of_batch

Queries the balances of multiple accounts in a single call. This method is designed to optimize balance retrievals, aggregating multiple requests into one and reducing the overhead of individual queries.

```candid "Type definitions" +=
type BalanceQueryArgs = record {
    accounts: vec Account;
};


type BalanceQueryResult = vec (Account, nat); 

// icrc4_balance_of_batch method definition
icrc4_balance_of_batch: (BalanceQueryArgs) -> (BalanceQueryResult) query;
```

#### Preconditions for the Balance Query Batch

- The request MUST include a list of accounts, each specified by their owner principal and optionally a subaccount.
- The number of accounts in the query MUST NOT exceed the maximum batch size for balance queries specified by the ledger.

#### Postconditions for the Balance Query Batch

- The method returns a list of accounts paired with their respective balances if the query succeeds.
- If an error occurs due to too many requests, a `TooManyRequests` trap will occur.
- Every balance included in the result corresponds directly to an account in the input query, maintaining the same order to facilitate mapping between requested accounts and their balances. Todo: do we want to require ordering?


## Batch Transfer Metadata

Ledgers supporting ICRC-4 SHOULD provide metadata via the icrc1_metadata() function including maximum batch size and MAY provide a fee per transfer to enable principals to structure batch transactions appropriately. If the fee per transfer is not included the user of the ledger should assume the value for icrc1:fee.

### Standard metadata entries
| Key | Semantics | Example value
| --- | ------------- | --------- |
| `icrc4:maximum_batch_size` | The IC has a ~2MB limit on ingress and responses there for an implementor must provide guidance on the maximum number of transactions that can be handled in one call | `variant { Nat = 10_000 }` | 
| `icrc4:batch_fee` | A replacement fee that can be used as an alternative to the icrc1:fee value. | `variant { Nat = 5_000 }` | 
| `icrc4:maximum_balance_size` | The IC has a ~2MB limit on ingress and responses there for an implementor must provide guidance on the maximum number of balance inquiries that can be handled in one call | `variant { Nat = 10_000 }` | 

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

If the client did not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction. //todo: is this still true?

todo: I assume that we need to not dedupe existing transactions, but we should discuss.

### Supported Standards

All ledgers implementing ICRC-4 MUST include the standard in the list returned by the `icrc1_supported_standards` method. Furthermore, ICRC-4 aims to adhere to the patterns established by ICRC-7 for batch processing, providing a consistent interface for both fungible and non-fungible token transfers within the Internet Computer ecosystem.

## Rationale and Backwards Compatibility

The introduction of the `memo` and `created_at_time` fields at the batch level, rather than on individual transactions, aligns ICRC-4 with the batch processing patterns of ICRC-7 and improves consistency across token standards. This structure simplifies deduplication processes and reduces the complexity of transaction verification for users and implementers.

Transactions within a batch SHALL share the same `memo` and `created_at_time` to allow for efficient deduplication. This approach eliminates the need for per-transaction memos, which can otherwise complicate the standard and its implementation.

ICRC-4 introduces no known backward compatibility issues with existing standards. It is designed as an extension of ICRC-1 and preserves the principles of individual transfer transactions.

## Future Considerations

As the Internet Computer ecosystem evolves, there may be further optimizations or adjustments to the batch transfer process. These changes will be proposed, reviewed, and integrated into subsequent ICRC proposals as necessary.

## Implementing ICRC-4

Developers and ledger maintainers are encouraged to adopt the ICRC-4 standard and provide feedback for future revisions. A reference implementation and developer guide will be made available once the standard is accepted and ratified.

