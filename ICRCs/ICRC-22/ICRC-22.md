# ICRC-22 — URI Format for Payment Requests

|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|22|URI Format for Payment Requests|Dieter Sommer (@dietersommer), David Dal Busco (@peterpeterparker), Thomas (@sea-snake), Austin Fatheree (@skilesare)|[Issue 10](https://github.com/dfinity/ICRC/issues/22)|Draft|Standards Track||2024-08-18|

## Introduction

ICRC-22 defines a standard way of representing transaction requests for payments or approvals of token transactions on ICP as URIs. A variety of transport mechanisms can be used to communicate URIs to their recipients, e.g., they can be embedded in QR codes, hyperlinks on Web pages, emails, or chat communication to obtain robust signalling between loosely-coupled applications. Such either pre-defined or on-the-fly generated payment requests can be immediately used by the user's wallet application, where relevant arguments are provided through the URI and further arguments may be filled in by the wallet application or the user themselves. This makes such URI encoding an indispensible tool in a larger blockchain ecosystem to allow for convenient communication of payment- or approval-related information over a variety of different communication channels. Generic smart contract method calls are left to a future ICRC standard.

Relevant prior work in other blockchain ecosystems comprises the standards on payment URLs for Bitcoin \[Bit1, Bit2\] as well as Ethereum's [ERC-681 \[Nag17\]](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-681.md) for expressing payment requests and, more generally, transactions using EVM ABI encoding, for the Ethereum blockchain. The specification put forth in ICRC-22 is an adaptation of the ideas in the above listed references to establish a standard way of sharing URIs that represent payment and other requests on the Internet Computer.

## Specification

The below specification defines the syntax and informal semantics for ICRC-22 token transfer and approval requests. The specification takes the approaches of \[Bit1, Bit2, Nag17, WLGH19\] into consideration. Particularly, an authority-less URI format has been chosen following this prior art in the domain.

Next, we show the structure of an ICRC1 transfer request URI with some of the concrete arguments represented through `<>`-placeholders. `737ba355e855bd4b61279056603e0550` is the network id for ICP mainnet. Note that the URI does not contain an authority, but encodes the `network`, the `icrc22` standard specifier, and `canister_principal` as part of the URI path, directly following the URI scheme.
```
icp:737ba355e855bd4b61279056603e0550:icrc22:<canister_principal>/icrc1_transfer?to=<to_account>&amount=<amount>&memo=<memo>
```

Next, we present the ABNF syntax for the prefix of ICRC-22 URIs. The details of the full ICRC-22 URI, i.e., encoding of the method name to call and the query string follow from the Candid method name and parameter syntax defined already in the respective specification and thus are not repeated here. Note that certain characters may require URI encoding, e.g., UTF8 characters outside of the character set allowed in URIs that are used in method names need to be encoded with URI encoding.
```
request = protocol ":" network ":" "icrc22" ":" canister_principal "/" method_name [ "?" parameters ]
protocol = "icp" ; always "icp", referring to the current version of the Internet Computer Protocol
network = (1..32)[0..9a..fA..F] ; specifies the network through a prefix of its public key hash
```

The grammar productions not further specified above can be easily derived from the Candid method name and parameter syntax. Note that `method_name` is part of the path and `parameters` part of the query string and the according constraints of RFC 3986 apply.

The `protocol` is always the string `icp` and refers to the namespace for the Internet Computer Protocol.

The `network` uniquely identifies the ICP network to make the transaction on. Following the approach of the Chain Agnostic Standards Alliance \[Cha24\] of having a standardized way of referring to any blockchain network, ICP uses the following mechanism for referring to ICP mainnet or any other ICP-based network that may be available in the future, such as testnets or private networks based on the ICP protocol stack ("UTOPIA" private networks):
The network is identified by a prefix of the the base16 (i.e., hexadecimal) encoding of the SHA256 hash of the binary representation of the current DER-encoded public key of the network.
// In case of a *well-known public network*, an 8-character prefix of the key hash can be used as network identifier, for private networks, a 32-character prefix is used to represent the network. Currently, only ICP mainnet qualifies as a well-known public network, in the future public testnets might be eligible also to be well-known public networks.
For ICP mainnet, the public key of the network is the NNS public key:
```
308182301d060d2b0601040182dc7c0503010201060c2b0601040182dc7c05030201036100814c0e6ec71fab583b08bd81373c255c3c371b2e84863c98a4f1e08b74235d14fb5d9c0cd546d9685f913a0c0b2cc5341583bf4b4392e467db96d65b9bb4cb717112f8472e0d5a4d14505ffd7484b01291091c5f87b98883463f98091a0baaae
```
Its 32-character prefix of the HEX-encoded SHA256 hash is `737ba355e855bd4b61279056603e0550`.
In case of a private ICP deployment, the 32-character prefix of the SHA256 hash of the network's DER-encoded public key is used as network identifier.
// The approach of expressing network identifiers through the 32-character prefix of the SHA256 hash of their public key is based on the approach proposed by the CAIP initiative, the shorter prefix for well-known networks or the default is a simplification for the most used network(s).

The `network` argument defaults to the identifier `737ba355e855bd4b61279056603e0550` for ICP mainnet if left out. In this default case, the prefix of an ICRC-22 URI thus is `icp::icrc22:<contract_address>/`. The interpretation of what "current mainnet" is is left to the client. Semantically, it refers to the currently active public key for mainnet. This public key can only change in the event of an NNS recovery after a large desaster event that destroys the NNS private threshold key, and thus the key is considered very stable and very unlikely to change.

The `icrc22` constant provides a sub-namespace for ICRC-22. The URI namespace following this specifier is controlled by ICRC-22.

The `canister_principal` is the canister smart contract principal identifier of the canister smart contract the transaction is to be performed on. This is represented in the standardized format for expressing principals in textual representation. The address, together with the network identifier, unambiguously determines the token that is to be operated on.

The `method_name` parameter specifies the canister method to be called on the canister indicated through `canister_principal`. This is the string-based name as it appears in the canister's Candid specification. This standard requires support of all of ICP ledger's `transfer`, ICRC-1's `icrc1_transfer` and ICRC-2's `icrc2_approve` methods.

The `parameters` are the parameters of the method to be called. They can be encoded in the URI in any order. A subset of the parameters defind in the Candid specification of the method can be present, the remaining non-optional parameters MUST be filled in by the party executing the method call in order to obtain the valid method call paramters.

### Supported Methods

Next, we define the parameters allowed in the various smart contract methods this standard supports. A subset of the parameters can be specified in the URI, meaning that the remaining paramters need to be filled in by the client (e.g., wallet) in case they are mandatory and can be filled in in case they are optional in the method's specification. Note also that the initiator of the transaction may be specified through the `from` or just the `from_subaccount`. If omitted, they need to be provided by the client application. We give the following example use case scenarios:
* A payment use case where a shop provides a payment link with `to`, `amount`, and `memo` and the wallet provides the source account through the caller principal and filling in the `from_subaccount` according to user's preferences and sets the `fee` and `created_at_time`. A simple shop use case can also comprise a similar set up with the `amount` being provided by the user and being communicated to the user by the salesperson. This may be a proper setup for a market where the URI is provided as a printed QR code.
* A token transfer use case triggered by a dapp where the dapp knows the principal and subaccount of the transferring user. In this case, all of `from`, `to`, `amount`,`fee`, and `memo` may be provided in the request and the client application only needs to fill in `created_at_time`. The client application extracts the `from_subaccount` from `from` and the caller principal of the method invocation must match the `principal` of the `from` parameter.

The arguments `from` and `from_subaccount` are always mutually exclusive and both optional.

ICRC-22 focuses on the transmission of the information and does not standardize any application-level flows. And discussions of such are meant as illustrating examples and are not normative.

// FIX text for arguments

#### ICP's transfer

* `from`: The initiator of the transfer, i.e., the spender. Encoded using the [size-reduced human readable textual representation](## Size-reduced-ICRC-1-textual-account-representation) of an ICRC-1 account as specified in this document. Used when the spender is specified by the URI creator already. Must match the initiator of the transfer and the inteded subaccount to transfer the funds from.
* `from_subaccount` The spender subaccount expressed as 32-byte subaccount in base64 representation. Subaccounts are 32-byte byte arrays and encoded in hexadecimal representation for use in ICRC-22. Leading zeroes on the left of a subaccount SHOULD be omitted in the encoding. Used when the subaccount from which the funds should be transferred is known by the creator of the URI. Must match the subaccount intended by the initiator of the transfer.
* `to`: The recipient of the token transfer. Encoded as an ICP account identifier.
* `amount` The amount of tokens to be transferred. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation, e.g., 4.042E8 and 804200000 express the same integer.
* `fee`: The fee for the transfer. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation.
* `memo`: A decimal non-negative integer representing the memo.
* `created_at_time`: The creation time of the transaction expressed as integer timestamp.

#### ICRC1 icrc1_transfer

* `from` The initiator of the transfer, i.e., the spender. Encoded using the [size-reduced human readable textual representation](## Size-reduced-ICRC-1-textual-account-representation) of an ICRC-1 account as specified in this document. Used when the spender is specified by the URI creator already. Must match the initiator of the transfer and the inteded subaccount to transfer the funds from.
* `from_subaccount` The spender subaccount expressed as 32-byte subaccount in base64 representation. Subaccounts are 32-byte byte arrays and encoded in hexadecimal representation for use in ICRC-22. Leading zeroes on the left of a subaccount SHOULD be omitted in the encoding. Used when the subaccount from which the funds should be transferred is known by the creator of the URI. Must match the subaccount intended by the initiator of the transfer.
* `to` The recipient of the token transfer. Encoded using the [size-reduced human readable textual representation](## Size-reduced-ICRC-1-textual-account-representation) of an ICRC-1 account as specified in this document.
* `amount` The amount of tokens to be transferred. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation, e.g., 4.042E8 and 804200000 express the same integer.
* `fee` The fee for the transfer. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation.
* `memo` A byte array expressed as hexadecimal string representing the memo.
* `created_at_time` The creation time of the transaction expressed as integer timestamp.

#### ICRC2 icrc2_approve

* `from` The initiator of the approval. Encoded using the [size-reduced human readable textual representation](## Size-reduced-ICRC-1-textual-account-representation) of an ICRC-1 account as specified in this document. Used when the spender is specified by the URI creator already. Must match the initiator of the transfer and the inteded subaccount to transfer the funds from.
* `from_subaccount` The source subaccount expressed as 32-byte subaccount in base64 representation. Subaccounts are 32-byte byte arrays and encoded in hexadecimal representation for use in ICRC-22. Leading zeroes on the left of a subaccount SHOULD be omitted in the encoding. Used when the subaccount from which the funds should be transferred is known by the creator of the URI. Must match the subaccount intended by the initiator of the transfer.
* `spender` (ICRC-2): The spender of the approved funds. Encoded using the [size-reduced human readable textual representation](## Size-reduced-ICRC-1-textual-account-representation) of an ICRC-1 account as specified in this document.
* `amount` The amount of tokens to be transferred. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation, e.g., 4.042E8 and 804200000 express the same integer.
* `expected_allowance` (ICRC-2): The amount to be approved, , specified as integer or in scientific representation, e.g., 4.042E8 or 804200000. The number refers to the base units of the ledger the request is targeted at.
* `expires_at` (ICRC-2): The expiration date of the approval expressed as integer timestamp.
* `fee` The fee for the transfer. The number refers to the number of tokens in the base units of the ledger the request is targeted at. Specified as decimal integer or in scientific representation.
* `memo` A byte array expressed as hexadecimal string representing the memo.
* `created_at_time` The creation time of the transaction expressed as integer timestamp.

Accounts are always represented through the size-reduced textual encoding as specified in [Size-Reduced ICRC-1 Textual Account Representationbelow](## Size-reduced-ICRC-1-textual-account-representation) because of the human readability and the built-in checksum over both the principal and subaccount components of the account.

The `amount` should be provided as a nonnegative integer number. The amount represents the amount of tokens in the base unit used by the ledger, i.e., 4 ICP tokens would amount to 4 * 10^8 = 400000000 base units as managed by the ICP token ledger. It is strongly recommended to use scientific notation for the amount. Decimal representation can be combined with scientific representation, e.g., 4.042E8 ICP means a count of 404200000 base units as managed by the ICP ledger. As only integer numbers are allowed as amount, the exponent MUST be greater than or equal to the decimal places of the number in scientific representation.

The arguments `expires_at`, `fee`, and `created_at_time` will in most use cases likely be set by the client application and not be contained in the URI.

Implementors of ICRC-22 should take note of the different encoding of the `memo` for the ICP token standard on the one hand and ICRC-1 and ICRC-2 on the other hand.

## Size-Reduced ICRC-1 Textual Account Representation

The Chain Agnostic Standards Alliance \[Cha24\] has specified account identifiers to be of a maximum length of 128 characters \[Gom22\]. There seems to be no strong reason behind this, but it had been considered sufficient for the foreseeable future by the CAIP working group and is thus a limitation for representing accounts. This limitation cannot be changed at the current time.

ICRC-1 account identifiers on ICP may exceed this limitation of 128 bytes by a few bytes when using the human-readable representation of ICRC-1 accounts, i.e., the textual encoding of ICRC-1 accounts \[Dfi22\]. For this reason, we define a size-reduced textual ICRC-1 account representation as part of this standard in order to fit the CAIP limits of 128 characters.

The size-reduced representation is easily computed by removing all the dashes ("-") between the character groups of the representation of the principal and checksum, except for the rightmost dash (the one preceding the CRC32 checksum) for non-default accounts (i.e., accounts with a non-zero subaccount). For default accounts (i.e., accounts with the all-zero subaccount), all dashes are removed. It is easy to reconstruct the standard textual encoding from a size-reduced textual encoding by adding dash separators, starting from the left of the representation, for every 5-character group.

It is guaranteed that any ICRC-1 account in size-reduced representation fits into the 128-character limit. The Internet Computer Interface Specification \[Dfi\] specifies the maximum lenght of a textual representation of a principal to be 63 bytes, which includes 10 dashes. The additional CRC32 checksum in Base32 encoding takes another 7 characters, its preceding dash separator one. This makes the maximum length of the size-reduced representation 126 characters: 63 for the maximum-length principal in textual encoding - 10 for separators + 1 for the checksum "-"-separator + 7 for the CRC32 checksum + 1 for the subaccount "."-separator + 64 for the subaccount in hexadecimal representation.

The following is an example of an almost maximum-length textual representation that exceeds the size limits imposed by CAIP with its length of 135 characters.
```
k2t6j-2nvnp-4zjm3-25dtz-6xhaa-c7boj-5gayf-oj3xs-i43lp-teztq-6ae-dfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```
The following is the above example in size-reduced form and fits within the 128-character limit with its length of 125 characters. Note that the final dash in the first part (the part left of the `.`) is retained as it separates the principal from the checksum of the account representation.
```
k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6ae-dfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```
This size-reduced representation can easily be transformed back to the original textual representation as defined in \[Dfi22\] by adding dashes every group of 5 characters from the left. Note that the rightmost character group left of the checksum can have less than 5 characters.
```
k2t6j-2nvnp-4zjm3-25dtz-6xhaa-c7boj-5gayf-oj3xs-i43lp-teztq-6ae-dfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```

## Data Compression

Depending on the transport mechanism intended to be used, data compression of the URI resulting from the ICRC-22 encoding MAY be used to reduce the size of the message to be sent over the transport. This is particularly important and recommended for QR codes used as transport mechanism as they have limited bandwidth and larger QR codes are harder to scan.

When compression of URIs for transfer in QR codes is used, it MUST be performed using the gzip algorithm. This algorithm is reasonably simple and native implementations are available in many languages and have a small code size, thus it is easy to implement this feature in both tools and canister smart contracts without too much bloat. Future standards may add additional compression algorithms with better compression ratios such as zstd or Brotli, with the drawbacks of much larger code size and not as widespread open source implementations being available.

## Examples

The following examples show the application of ICRC-22 for ICP transfer, ICRC-1 transfer, and ICRC-2 approval use cases.

### ICP Token Transfer

```
icp:<network>:ryjl3-tyaaa-aaaaa-aaaba-cai/exec/transfer?from_subaccount=<subaccount>&to=<icp_account>&amount=4.042E8&memo=<memo>
```

The following shows an example of a transfer request of `4.042` ICP using on the ICP token ledger to `61342bfbd397f0c36c5da1f9661b802db60a20e973012c99903fc8637b5bf32b` with `memo` being the natural number `9812345670123456`, where `to` (the recipient account id), `amount`, and `memo` are specified, while the `from_subaccount`, `fee`, and `created_at_time` are provided by the client application (e.g., a wallet).
```
icp:737ba355e855bd4b61279056603e0550:icrc22:ryjl3-tyaaa-aaaaa-aaaba-cai/transfer?to=61342bfbd397f0c36c5da1f9661b802db60a20e973012c99903fc8637b5bf32b&amount=4.042E8&memo=9812345670123456
```
The string `ryjl3-tyaaa-aaaaa-aaaba-cai` is the principal of the ICP ledger.

This is a typical example of requesting a payment in ICP tokens from an off-chain service.

### ICRC-1 Token Transfer

The following is a payment request for `0.04042` ckBTC on the ckBTC ledger of the caller to the account `k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6aedfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20` with `memo` being the hex-encoded byte array `1b69b4ba630f34e15fd40af1`. The wallet reading and executing this request must insert the `from_subaccount`, `fee`, and `created_at_time` accordingly. For the representation of the amount, note that `1` ckBTC equals `1E8` (100 million) base units of the ckBTC ledger.
```
icp:737ba355e855bd4b61279056603e0550:icrc22:mxzaz-hqaaa-aaaar-qaada-cai/icrc1_transfer?to=k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6ae-dfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20&amount=4.042E6&memo=1b69b4ba630f34e15fd40af1
```
The string `mxzaz-hqaaa-aaaar-qaada-cai` is the principal of the ckBTC ledger.

### ICRC-2 Approval

An ICRC-2 approval can be made on any ICRC-2-compliant ledger, e.g., the ICP ledger or any ICRC-1-compliant token ledger like the ckBTC legder. The following example shows an ICRC-2 approval of `0.04042` ckBTC to account `h2oto-yhzgh-fdd7o-ucdym-dnwul-ihnjb-67dlq-ed3x2-mxzf2-des4t-xqe`. The `fee` and `created_at_time` arguments need to be provided by the wallet. The arguments `from_subaccount`, `expected_allowance` (optional), `fee`, and `created_at_time` need to be provided by the client application accordingly.

```
icp:737ba355e855bd4b61279056603e0550:icrc22:mxzaz-hqaaa-aaaar-qaada-cai/icrc2_approve?spender=h2otoyhzghfdd7oucdymdnwulihnjb67dlqed3x2mxzf2des4txqe&amount=4.042E6&memo=1b69b4ba630f34e15fd40ab8
```

## Future Work

The current specification is focussed on expressing token transfer and approval requests for the ICP, ICRC-1, and ICRC-2 token standards. The idea can be generalized, similar to ERC-681 \[Nag17\], to express any canister smart contract method invocation. ERC-681 is handling this generalization with embedded EVM ABI encoded arguments in the URI. For the Internet Computer, embedding Candid- or otherwise encoded arguments into the request is the natural way to accomplish this. The remaining issue is that mandatory parameters such as timestamps may need to be filled in by the client, but must be encoded already in the Candid structure to result in valid Candid. This can be handled by encoding a default value, e.g., 0 for a number type, in such cases and providing additional metadata that such element must be replaced by the client. Due to those complexities that still need to be worked out, we do not handle this extension in ICRC-22, but defer it to future work when to be done when use cases demand such functionality.

## References

* [Nag17] Daniel A. Nagy, URL Format for Transaction Requests. Ethereum EIP-681, https://github.com/ethereum/EIPs/blob/master/EIPS/eip-681.md, 2017
* [Bit1] Unified payment requests. https://bitcoin.design/guide/how-it-works/payment-request-formats/#unified-payment-requests
* [Bit2] Unified QRs for Bitcoin. https://bitcoinqr.dev/
* [Dfi22] DFINITY, ICRC-1 Token Standard -- Textual encoding of ICRC-1 accounts. 2024, https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md, 2022
* [WLGH19] Simon Warta, ligi <ligi@ligi.de>, Pedro Gomes, Antoine Herzog, Blockchain ID Specification, https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md, 2019
* [Cha24] Chain Agnostic Standards Alliance. https://github.com/ChainAgnostic
* [Gom22] Pedro Gomes, Account ID Specification. CAIP-10, https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md, 2022
* [Dfi] DFINITY Foundation, The Internet Computer Interface Specification: Principals. https://internetcomputer.org/docs/current/references/ic-interface-spec#principal
* [BFM05] T. Berners-Lee, R. Fielding, L. Masinter: Uniform Resource Identifier (URI): Generic Syntax. 2005, https://datatracker.ietf.org/doc/html/rfc3986
