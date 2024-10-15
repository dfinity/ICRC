# ICRC-22 — URI Format for Payment Requests

|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|22|URI Format for Payment Requests|Dieter Sommer (@dietersommer), David Dal Busco (@peterpeterparker), Thomas (@sea-snake), Austin Fatheree (@skilesare)|[Issue 10](https://github.com/dfinity/ICRC/issues/22)|Draft|Standards Track||2024-08-18|

## Introduction

ICRC-22 defines a standard way of representing transaction requests, e.g., for payments or approvals of token transactions, on ICP as URIs. A variety of transport mechanisms can be used to communicate URIs to their recipients, e.g., they can be embedded in QR codes, hyperlinks on Web pages, emails, or chat communication, to obtain robust signalling between loosely-coupled applications. Such pre-defined or on-the-fly generated payment requests can be immediately used by the user's wallet application, where relevant parameters are provided through the URI and further parameters may be filled in by the wallet application or the user themselves. This makes such URI encoding an indispensible tool in a larger blockchain ecosystem to allow for convenient communication of payment-related information over many different communication channels. ICRC-22 clearly focusses on payment-related transactions, but supports a wider range of transactions to be expressed as URIs.

Relevant prior work in other blockchain ecosystems comprises the standards on payment URLs for Bitcoin \[Bit1, Bit2\] as well as Ethereum's [ERC-681 \[Nag17\]](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-681.md) for expressing payment requests and, more generally, transactions using EVM ABI encoding, for the Ethereum blockchain. The specification put forth in ICRC-22 is an adaptation of the ideas in the above listed references to establish a standard way of sharing URIs that represent payment and other requests on the Internet Computer.

## Specification for Payments

The below specification defines the syntax and (non-formal) semantics for an ICRC-22 payment request and extension to further requests. The specification takes the approaches of \[Bit1, Bit2, Nag17, WLGH19\] into consideration. Particularly, an authority-less URI format has been chosen following this prior art in the domain.

Next, we show the structure of a request URI with the concrete parameters represented through `<>`-placeholders. Note that the URI does not contain an authority, but encodes the `network` and `contract_address` as part of the URI path, directly following the URI scheme.
```
icp:737ba355e855bd4b61279056603e0550:<canister_principal>/exec/icrc1_transfer?enc=plain&from_subaccount=<from_subaccount>&to=<to_account>&amount=<amount>&memo=<memo>
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
The `network` defaults to the identifier `737ba355e855bd4b61279056603e0550` for ICP mainnet. When the network identifier is left out, the prefix of a URI would thus be `icp::<contract_address>/exec/<transaction>?` for the current ICP mainnet. The interpretation of what "current mainnet" is is left to the client. It should refer to the currently active key for mainnet, which can only change in the event of an NNS recovery after a large desaster event that destroys the NNS private threshold key, and thus is considered very stable.

An encoding specifier as initial element of query string defines the kind of encoding used for the parameters. ICRC-22 defines only the encoding `plain`. We envision a future ICRC that uses Candid for encoding any list of parameters, indicated through the `candid` constant. If omitted, the encoding defaults to the `plain` encoding.
// Do we need to distinguish between payment and generic method defined here?

The grammar productions not further specified above can be looked up in the syntax specification for URIs in RFC 3986 \[BFM05\]. Note that the production `transaction` above is part of the path and `parameters` part of the query string and the according constraints of RFC 3986 apply. // FIX

The `network_identifier` uniquely identifies the ICP network to make the transaction on. Following the ideas of the Chain Agnostic Standards Alliance \[Cha24\] of having a standardized way of referring to any blockchain network, ICP uses the following mechanism for referring to ICP mainnet or any other ICP-based network that may be available in the future, such as testnets or private networks based on the ICP protocol stack ("UTOPIA" private networks).
The network is identified by a prefix of the the Base16 (HEX) encoding of the SHA256 hash of the binary representation of the DER-encoded public key of the network. In case of a *well-known public network*, an 8-character prefix of the key hash can be used as network identifier, for private networks, a 32-character prefix is used to represent the network. Currently, only ICP mainnet qualifies as a well-known public network, in the future public testnets might be eligible also to be well-known public networks. For ICP mainnet, the public key of the network is the NNS public key:
```
308182301d060d2b0601040182dc7c0503010201060c2b0601040182dc7c05030201036100814c0e6ec71fab583b08bd81373c255c3c371b2e84863c98a4f1e08b74235d14fb5d9c0cd546d9685f913a0c0b2cc5341583bf4b4392e467db96d65b9bb4cb717112f8472e0d5a4d14505ffd7484b01291091c5f87b98883463f98091a0baaae
```
Its 32-character prefix of the HEX-encoded SHA256 hash is
`737ba355e855bd4b61279056603e0550`.
In case of a private ICP deployment, the 32-character prefix of the SHA256 hash of the network's DER-encoded public key is used as network identifier.
The approach of expressing network identifiers through the 32-character prefix of the SHA256 hash of their public key is based on the approach proposed by the CAIP initiative, the shorter prefix for well-known networks or the default is a simplification for the most used network(s).

The `contract_address` is the canister smart contract principal identifier of the contract the transaction is to be performed on. This is represented in the standardized format for principals. The address, together with the network identifier, unambiguously determines the token that is to be transacted.

The constant `exec` means that the forthcoming part in the path component specifies which transaction (smart contract method) should be invoked with the information in the URI. In the future, other ICRC standards may emerge that can have a different string with different semantics in the place of this.

The `transaction` parameter specifies the canister method to be called on the canister indicated through `contract_address`. This is the string-based name as it appears in the canister's Candid specification. This standard requires support of all of ICP ledger's `transfer`, ICRC-1's `transfer` and ICRC-2's `approve` methods. The method names to handle payment-related use cases currently envisioned are `transfer`, `icrc1_transfer`, and `icrc2_approve`.

The `parameters` are the parameters of the method to be called. They SHOULD be given in the order in which they appear in the Candid specification of the method to be called. A subset of the parameters defind in the Candid specification of the method can be present, the remaining non-optional parameters MUST be filled in by the party executing the transaction.

Next, we define the parameters required for realizing the supported transactions of the supported ledger standards and their encoding. For each parameter we indicate the standards for which they are required:
* `from_subaccount` (ICRC-1, ICRC-2): 32-byte subaccount in Base64 representation.
* `to` (ICRC-1, ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `spender` (ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `spender_subaccount` (ICRC-2): 32-byte subaccount in Base64 representation.
* `from` (ICRC-2): The size-reduced human readable [textual representation](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md) of an ICRC-1 account as specified in this document.
* `amount` (ICP, ICRC-1, ICRC-2): The transaction amount, specified as integer or in scientific representation, e.g., 4.042E8 or 804200000. The number refers to the base units of the ledger the request is targeted at.
* `expected_allowance` (ICRC-2): The amount to be approved, , specified as integer or in scientific representation, e.g., 4.042E8 or 804200000. The number refers to the base units of the ledger the request is targeted at.
* `expires_at` (ICRC-2): The expiration date of the approval expressed as integer timestamp.
* `fee` (ICP, ICRC-1, ICRC-2): Fee expressed in base units of the addressed ledger, using scientific notation if appropriate.
* `memo` (ICP, ICRC-1, ICRC-2): An integer (for ICP transactions) or byte array expressed as hexadecimal string (for ICRC-1 transactions) representing the memo.
* `created_at_time` (ICP, ICRC-1, ICRC-2): The creation time of the transaction expressed as integer timestamp.
* ...
Accounts are always represented through the size-reduced textual encoding as specified in [Size-Reduced ICRC-1 Textual Account Representationbelow](## Size-reduced-ICRC-1-textual-account-representation) because of the built-in checksum over both the principal and subaccount components of the account and the human readability.
// FIX complete the list, define the respective encoding, maybe restructure per method

Subaccounts are 32-byte byte arrays and encoded in hexadecimal representation for ICRC-22. Leading zeroes on the left of a subaccount SHOULD be omitted in the encoding.

The `amount` should be provided as a nonnegative integer number. The amount represents the amount of tokens in the base unit used by the ledger, i.e., 4 ICP tokens would amount to 4 * 10^8 = 400000000 base units as managed by the ICP token ledger. It is strongly recommended to use scientific notation for the amount. Decimal representation can be combined with scientific representation, e.g., 4.042E8 ICP means a count of 404200000 base units as managed by the ICP ledger. As only integer numbers are allowed as amount, the exponent MUST be greater than or equal to the decimal places of the number in scientific representation.

## Generalization Towards Handling a Larger Class of Method Calls

ICRC-22 does not handle generic calls of smart contract methods on ICP, but does allow for handling further requests that are sufficiently similar to the ones captured explicitly. This comprises requests that have parameters that can be represented as query parameters in a URI in a straightforward way and a canonical encoding exists for the parameters.

The generic representation of call parameters is done as follows through a canonical encoding of the method parameters as URI query parameters:
* a `bool` parameter is represented as either `true` or `false`
* a `text` parameter is represented as a string representing the text
* a `number` parameter is encoded as the string representation of the decimal number
* a binary `blob` parameter is hexadecimal encoded
* a tuple is encoded by having an opening element `*tuple`, followed by the tuple values, followed by a closing element `-tuple`
* a record is encoding is started with the query parameter `*record=fieldName`, where `fieldName` is the name of the field containing the record; this is followed by a sequence of encodings of its fields according to the encoding rules; the termination of the record is encoded with `-record=fieldName`
* an array is started with `*array=fieldName`, followed by an encoding of its entries the names of which are their index using 0-based indexing (e.g., `0=8` for the 0-th field with value `8`), followed by a termination `-array=fieldName`
* an enumeration is encoded by a start element `*enum=fieldName` and a variant element encoding the variant of the enumeration; for variants that have an associated value, the value is encoded after the variant using the encoding rules

Fields that are mandatory according to the Candid specification may be left out from the encoding in case that the canister needs to fill in those missing values. Note that those values MUST be provided by the canister in order to obtain a valid Candid value to use for the method invocation.

A large class of method interfaces — every method the encoding of which can be uniquely decoded back to Candid — can be handled with this method of canonical flattening of nested data types expressible in Candid. However, for complex data structures an approach of mapping the parameters to Candid and encoding the Candid value in the URI may be preferable and part of a future ICRC as outlined in the [Future Work](##Future-Work) section.

The specification of payment-related transaction requests as defined for ICP and ICRC-1 payments and ICRC-2 approvals is a special case of the generalized encoding introduced in this section, with the following differences:
* the parameters are provided as a flat list, with the enclosing `TransferArgs` record being made explicit; the rationale behind this is that payment-related use cases are the primary target of this specification;
* ICRC-1 accounts are encoded using the size-reduced textual representation of ICRC-1 accounts ([Dfi22]) as defined in this document.

The generalization introduced in this section is optional for an implementation of ICRC-22 to keep the minimum requirements for an implementation simple and help widespread adoption of ICRC-22 for payment-related use cases.

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

Depending on the transport mechanism used, data compression of the URI resulting from the encoding MAY be used to reduce the size of the message to be sent over the transport. This is particularly important for QR codes used as transport mechanism as they have limited bandwidth and larger QR codes are harder to scan.

Compression of URIs for transfer in QR codes, if used, MUST be performed using the gzip algorithm. This algorithm is reasonably simple and native implementations are available in many languages and have a small code size, thus it is easy to implement this feature in both tools and canister smart contracts without too much bloat. Future standards may add additional compression algorithms with better compression ratios such as zstd or Brotli.

## Examples

The following examples show the application of ICRC-22 for ICP transfer, ICRC-1 transfer, and ICRC-2 approval use cases.

### ICP Token Transfer

```
icp:<network>:ryjl3-tyaaa-aaaaa-aaaba-cai/exec/transfer?from_subaccount=<subaccount>&to=<icp_account>&amount=4.042E8&memo=<memo>
```

```
icp:737ba355e855bd4b61279056603e0550:ryjl3-tyaaa-aaaaa-aaaba-cai/exec/transfer?from_subaccount=0&to=61342bfbd397f0c36c5da1f9661b802db60a20e973012c99903fc8637b5bf32b&amount=4.042E8&memo=<memo>
```

Note that `ryjl3-tyaaa-aaaaa-aaaba-cai` is the principal of the ICP ledger.

### ICRC-1 Token Transfer

// FIX replace 1234abcd90 with a prefix of the SHA256 hash of the mainnet NNS public key in binary

// FIX fill in subaccounts etc. with accounts in the proper format

The following is a payment request for `0.04042` ckBTC on the ckBTC ledger from subaccount `ef` of the caller to the account `k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6aedfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20` with memo `1b69b4ba630f34e15fd40af`. The wallet reading and executing this request must insert the `created_at_time` field based on the current time to create a valid ledger transaction. For the representation of the amount, note that 1 ckBTC equals `1E8` (100 million) base units of the ckBTC ledger.
```
icp:737ba355e855bd4b61279056603e0550:mxzaz-hqaaa-aaaar-qaada-cai/exec/icrc1_transfer?from_subaccount=ef&to=k2t6j2nvnp4zjm325dtz6xhaac7boj5gayfoj3xsi43lpteztq6aedfxgiyy.102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20&amount=4.042E6&memo=1b69b4ba630f34e15fd40af
```

### ICRC-2 Approval

An ICRC-2 approval can be made on any ICRC-2-compliant ledger, e.g., the ICP ledger or any ICRC-1-compliant token ledger like the ckBTC legder. The following example shows an ICRC-2 approval of `0.04042` ckBTC to account `h2oto-yhzgh-fdd7o-ucdym-dnwul-ihnjb-67dlq-ed3x2-mxzf2-des4t-xqe`. The `fee` and `created_at_time` parameters need to be provided by the wallet.

```
icp:737ba355e855bd4b61279056603e0550:mxzaz-hqaaa-aaaar-qaada-cai/exec/icrc2_approve?from_subaccount=3&spender=h2otoyhzghfdd7oucdymdnwulihnjb67dlqed3x2mxzf2des4txqe&amount=4.042E6&expected_allowance=0&...
```

## Future Work

The current specification is focussed on expressing payment-related requests, concretely payments and approvals related to ICP, ICRC-1, and ICRC-2 standards. The idea can be generalized, like in ERC-681 \[Nag17\] to expressing any smart contract invocation. ERC-681 is handling this generalization with embedded EVM ABI encoded parameters in the URI.

For the Internet Computer, embedding Candid-encoded parameters are the natural way to accomplish this. The remaining issue is that mandatory parameters such as timestamps may need to be filled in by the client, but must be encoded already in the Candid structure to result in valid Candid. This can be handled by encoding a default value, e.g., 0 for a number type, in such cases and providing additional metadata that such element must be replaced by the client. Due to those complexities that still need to be worked out, we do not handle this extension in ICRC-22, but defer it to future work when to be done when use cases demand such functionality.

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
