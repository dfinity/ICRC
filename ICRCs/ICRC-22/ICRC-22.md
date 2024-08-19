# ICRC-22 — URI Format for Payment Requests

|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|22|URI Format for Payment Requests|Dieter Sommer (@dietersommer), David Dal Busco (@peterpeterparker), Thomas (@sea-snake), Austin Fatheree (@skilesare)|[Issue 10](https://github.com/dfinity/ICRC/issues/22)|Draft|Standards Track||2024-08-18|

## Introduction

ICRC-22 defines a standard way of representing payment requests on ICP as URIs. URIs can be embedded in QR codes, hyperlinks on Web pages, emails, or chat communication and serve the purpose of robust signalling between loosely-coupled applications. Such pre-canned or on-the-fly generated payment requests can be immediately used by the user's wallet application, where relevant parameters are provided by the URI and some other parameters may be filled in by the wallet application. This makes such URI encoding an indispensible tool in a larger blockchain ecosystem to allow for convenient communication of payment-related information over many different communication channels.

Relevant prior work in other blockchain ecosystems comprises the standards on payment URLs for Bitcoin \[Bit1, Bit2\] as well as Ethereum's [ERC-681 \[Nag17\]](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-681.md) for expressing payment requests and, more generally, transactions using EVM ABI encoding, for the Ethereum blockchain. The specification put forth in ICRC-22 is an adaptation of the ideas in the abovelisted references to establish a standard way of sharing URIs that represent payment requests on the Internet Computer.

## Specification

The below specification defines the syntax and (non-formal) semantics for an ICRC-22 payment request. The specification takes the approaches of \[Bit1, Bit2, Nag17, WLGH19\] into consideration. Particularly, an authority-less URI format has been chosen following this prior art in the domain.

The following shows the structure of a URI for an example with the concrete parameters represented through `<>`-placeholders. Note that the URI does not contain an authority, but encodes the `network` and `contract_address` as part of the URI path, directly following the URI scheme.
```
icp:<network>:<contract_address>/exec/<transaction>?from_subaccount=<subaccount>&to=<account>&amount=4.042E8&fee=<fee>&memo=<memo>
```

// FIX syntax details
Next, we present the ABNF syntax for ICRC-22 URIs.
```
request = protocol ":" network ":" contract_address "/" "exec" transaction [ "?" parameters ]
protocol = "icp" ; always "icp" referring to the current version of the Internet Computer Protocol
network = (1..10)[0..9a..fA..F] / ; specifies the network through a prefix of its public key hash
contract_address = ICP principal FIX: canister principal ABNF ; canister address to which to make the call
transaction = ... ; canister method to call FIX: Candid method syntax
parameters = parameter [ "&" parameter ]
parameter = key "=" value
key = ... ; FIX: Candid parameter name syntax
value = ... ; FIX Candid encoded value syntax
```

// FIX: We should default the network to ICP's mainnet current value when omitted
// Would result in `icp::<contract_address>/exec/<transaction>?` for current mainnet; the interpretation of what "current mainnet" is would be left to the client (the currently active key for mainnet, which can only change in the event of an NNS recovery after a large desaster event that destroys the NNS private threshold key)

// FIX: Should we keep it open to other methods and supported ICRCs as long as their API can be canonically expressed using a sequential list of parameters encoded as query parameters

// maybe add an encoding specifier in the beginning of query to keep it open for future work; this could look like `exec/enc_plain` for ICRC-22 and `exec/enc_candid` for the extensions in the future work; it could default to the further

The grammar productions not further specified can be looked up in the syntax specification for URIs in RFC 3986 \[BFM05\]. Note that the production `transaction` above is part of the path and `parameters` part of the query string and the according constraints of RFC 3986 apply.

The `network_identifier` uniquely identifies the network to make the transaction on. Following the ideas of the Chain Agnostic Standards Alliance \[Cha24\] of having a standardized way of referring to any blockchain network, ICP uses the following mechanism for referring to ICP mainnet or any other ICP-based network that may be available in the future, such as testnets or private networks based on the ICP protocol stack.
Base16 encoding of the SHA256 hash of the binary representation of the DER-encoded public key of the network. The prefix length used ensures that the probability of collision is negligible for all practical purposes. For ICP mainnet the public key of the network is the NNS public key.
// FIX using Base64 encoding for the network allows for using a shorter prefix, which is crucial for usability and human readibility; prefix length t.b.d.
// FIX Is it a security risk to use a prefix of the hash?

The `contract_address` is the canister smart contract principal identifier of the contract the transaction is to be performed on. This is represented in the standardized format for principals. The address, together with the network identifier, unambiguously determines the token that is to be transacted.

The constant `exec` means that the forthcoming part in the path component specifies which transaction (smart contract method) should be invoked with the information in the URI. In the future, other ICRC standards may emerge that can have a different string with different semantics in the place of this. // FIX: do we want to call this `tx`?

The `transaction` parameter specifies the canister method to be called on the canister indicated through `contract_address`. This is the string-based name as it appears in the canister's Candid specification. This standard requires support of all of ICP ledger's `transfer`, ICRC-1's `transfer` and ICRC-2's `approve` methods. The method names to handle payment-related use cases currently envisioned are `transfer`, `icrc1_transfer`, and `icrc2_approve`.

The `parameters` are the parameters of the method to be called. They SHOULD be given in the order in which they appear in the Candid specification of the method to be called. A subset of the parameters defind in the Candid specification of the method can be present, the remaining non-optional parameters MUST be filled in by the party executing the transaction.

// FIX: Details need to be worked out to make it possible to call different methods we want to cover in the base standard (at least ICP and ICRC-1 transfer and ICRC-2 approve). The result may not be so far away from a generic standard covering the call of any method of canisters. Specifically: Should types be decomposed into their constituents or the compound values be provided using a specific or their Candid encoding?

Next, we define the parameters required for realizing the supported transactions of the supported ledger standards:
* `from_subaccount` (ICRC-1, ICRC-2): 32-byte subaccount in Base64 representation
* `to` (ICRC-1, ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `spender` (ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `spender_subaccount` (ICRC-2): 
* `from` (ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `amount` (ICP, ICRC-1, ICRC-2): The transaction amount, specified in scientific representation, e.g., 4.042E8 means 804200000 base units of the addressed ledger.
* `expected_allowance` (ICRC-2): The amount to be approved, specified in scientific representation, e.g., 4.042E8 means 804200000 base units of the addressed ledger.
* `expires_at` (ICRC-2): The expiration date of the approval.
* `fee` (ICP, ICRC-1, ICRC-2): Fee expressed in base units of the addressed ledger, using scientific notation if appropriate.
* `memo` (ICP, ICRC-1, ICRC-2): A hexadecimal string representing the memo.
* `created_at_time` (ICP, ICRC-1, ICRC-2): The creation time of the transaction.
* ...
Accounts are always represented through the size-reduced textual encoding as specified in [the section below](## Size-reduced-ICRC-1-textual-account-representation).
// FIX complete the list, define the respective encoding, maybe restructure per method

The `amount` should be provided as a nonnegative integer number. The amount represents the amount of tokens in the base unit used by the ledger, i.e., 4 ICP tokens would amount to 4 * 10^8 = 400000000 base units as managed by the ICP token ledger. It is strongly recommended to use scientific notation for the amount. Decimal representation can be combined with scientific representation, e.g., 4.042E8 ICP means a count of 404200000 base units as managed by the ICP ledger. As only integer numbers are allowed as amount, the exponent MUST be greater than or equal the decimal places of the number in scientific representation.

// FIX How exactly can the mapping from the parameters to the Candid method signature be done? Is it sufficient to enumerate all named parameters, with the type inferable by the recipient using the canister's Candid specification which can be obtained from the canister?

// FIX How to handle optional fields in the Candid specification of the method? As long as the input can be constructed unambiguously, they can be left out, otherwise be included as `key=null`.

## Generalization Towards Handling a Larger Class of Method Calls

ICRC-22 does not handle generic calls of smart contract methods on ICP, but does allow for handling further requests that are sufficiently similar to the ones captured explicitly. This comprises requests that have parameters that can be represented as query parameters in a URI in a straightforward way and a canonical encoding exists for the parameters. Canonical encoding means that `text` parameters are represented as a strings, `number` paramters as the string of the number representation, binary `blob` parameters are hexadecimal encoded, records are "inlined", i.e., their fields are encoded sequentially in the query string while encoding the beginning of the record with the record name as key and the string `record` as value, and arrays are "inlined" as well so that every of its entries receives the array's name and the array starts with the array name as key and `array` as value. A large class of method interfaces can be handled with this method of canonical flattening of any nested Candid structure, however, this simple Candid-to-URI mapping may have its limits when parameters get nested in a too complex manner, contain the same names on different levels, or otherwise make it hard to re-assemble the Candid request in unambiguous form given the Candid specification of the method and the query string. Those cases may need a more elaborate approach of handling as outlined in the [Future Work](##Future-Work) section.
// FIX too vague, work out and explain in more detail

## Size-Reduced ICRC-1 Textual Account Representation

// FIX maybe call "compressed" instead of "size-reduced"

The Chain Agnostic Standards Alliance \[Cha24\] has specified account identifiers to be of a maximum length of 128 characters \[Gom22\]. There seems to be no strong reason behind this, but it has been considered sufficient for the foreseeable future by the CAIP working group. This limitation cannot be changed at the current time.

ICRC-1 account identifiers on ICP may exceed this limitation by a few bytes when using the human-readable representation of ICRC-1 accounts, i.e., textual encoding of ICRC-1 accounts \[Dfi22\]. For this reason, we define a size-reduced textual ICRC-1 account representation as part of this standard in order to fit the CAIP limits of 128 characters. The size-reduced representation is easily computed by removing all the dashes ("-") between the character groups of the representation, except for the rightmost one (the one preceding the CRC32 checksum) for non-default accounts (non-zero subaccount). For default accounts (all-zero subaccount), all dashes are removed. It is easy to reconstruct the standard textual encoding from a size-reduced textual encoding by adding dash separators from the left of the representation for every 5-character group.

It is guaranteed that any ICRC-1 account in size-reduced representation fits into the 128-character limit. The Internet Computer Interface Specification \[Dfi\] specifies the maximum lenght of a textual representation of a principal to be 63 bytes, which includes 10 dashes. The additional CRC32 checksum in Base32 encoding takes another 7 characters, its preceding dash separator one. This makes the maximum length of the size-reduced representation 126 characters: 63 for the maximum-length principal in textual encoding - 10 for separators + 1 for the checksum "-"-separator + 7 for the CRC32 checksum + 1 for the subaccount "."-separator + 64 for the subaccount in hexadecimal representation.

The following is an example of an almost maximum-length textual representation that exceeds the size limits imposed by CAIP with its length of 135 characters.
```
k2t6j-2nvnp-4zjm3-25dtz-6xhaa-c7boj-5gayf-oj3xs-i43lp-teztq-6ae-dfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```
The following is the above example in size-reduced form and fits within the 128-character limit with its length of 124 characters.
```
k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6aedfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```

// FIX use maximum-length textual account representation as example; the current one is close

## Examples

The following examples show the application of ICRC-22 for ICP transfer, ICRC-1 transfer, and ICRC-2 approval use cases.

### ICRC-1 Token Transfer

// FIX replace 1234abcd90 with a prefix of the SHA256 hash of the mainnet NNS public key in binary

// FIX fill in subaccounts etc. with accounts in the proper format

The following is a payment request for 4.042 ckBTC on the ckBTC ledger. The wallet reading and executing this request would insert the `created_at_time` field based on the current time.
```
icp:1234abcd90:mxzaz-hqaaa-aaaar-qaada-cai/exec/icrc1_transfer?from_subaccount=10&to=k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6aedfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20&amount=4.042E8&memo=receipt%2012345678
```

### ICP Token Transfer

// FIX
```
icp:1234abcd90:<icp_contract_address>/exec/transfer?from_subaccount=<subaccount>&to=<icp_address>&amount=4.042E8&fee=<fee>&memo=<memo>&created_at_time=<timestamp>
```

### ICRC-2 Approval

An ICRC-2 approval can be made on any ICRC-2-compliant ledger, e.g., the ICP ledger or any ICRC-1-compliant token ledger.

```
icp:1234abcd90:<icrc1_contract_address>/exec/icrc2_approve?from_subaccount=<subaccount>&spender=<spender_account>&amount=<amount>&expected_allowance=<allowance>&...
```

## Future Work

The current specification is focussed on expressing payment-related requests, concretely payments and approvals related to ICP, ICRC-1, and ICRC-2 standards. The idea can be generalized, like in ERC-681 \[Nag17\] to expressing any smart contract invocation. ERC-681 is handling this generalization with embedded EVM ABI encoded parameters in the URI.

For the Internet Computer, embedding Candid-encoded parameters are the natural way to accomplish this. The remaining issue is that mandatory parameters such as timestamps may need to be filled in by the client, but must be encoded already in the Candid structure to result in valid Candid. This can be handled by encoding a default value, e.g., 0 for a number type, in such cases and providing additional metadata that such element must be replaced by the client. Due to those complexities that still need to be worked out, we do not handle this extension in ICRC-22, but defer it to future work when to be done when use cases demand such functionality.

## References

[Nag17] Daniel A. Nagy, URL Format for Transaction Requests. Ethereum EIP-681, https://github.com/ethereum/EIPs/blob/master/EIPS/eip-681.md, 2017
[Bit1] Unified payment requests. https://bitcoin.design/guide/how-it-works/payment-request-formats/#unified-payment-requests
[Bit2] Unified QRs for Bitcoin. https://bitcoinqr.dev/
[Dfi22] DFINITY, ICRC-1 Token Standard -- Textual encoding of ICRC-1 accounts. 2024, https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md, 2022
[WLGH19] Simon Warta, ligi <ligi@ligi.de>, Pedro Gomes, Antoine Herzog, Blockchain ID Specification, https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md, 2019
[Cha24] Chain Agnostic Standards Alliance. https://github.com/ChainAgnostic
[Gom22] Pedro Gomes, Account ID Specification. CAIP-10, https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md, 2022
[Dfi] DFINITY Foundation, The Internet Computer Interface Specification: Principals. https://internetcomputer.org/docs/current/references/ic-interface-spec#principal
[BFM05] T. Berners-Lee, R. Fielding, L. Masinter: Uniform Resource Identifier (URI): Generic Syntax. 2005, https://datatracker.ietf.org/doc/html/rfc3986
