# ic-http URI Scheme For Addressing Content of Canisters Exposed Via Their HTTP Interface
|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|91|ic-http URI Scheme For Addressing Content of Canisters Exposed Via Their HTTP Interface|Dieter Sommer (@dietersommer), Thomas (@sea-snake), Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/91|Draft|Standards Track||2024-08-09|


## Introduction

Canister smart contracts on ICP can expose an HTTP interface that can be accessed from an HTTP client like a regular browser via an HTTP gateway. The HTTP Gateway Protocol Specification mandates regular HTTP URIs to specify a resource of a canister. That is, the HTTP URI for a canister contains the boundary node as the authority, or host name. This is inflexible as the URI contains a specific boundary node host name that resolves to a set of boundary nodes and multiple different URIs with different boundary node hostnames refer to the same resource. 

The following example shows a regular https URL,, which includes the hostname `ico.app` referring to a set of boundary nodes, to access a resource hosted by a canister:
`https://dlbnd-beaaa-aaaaa-qaana-cai.ic0.app/tokens/12345678/image`

Boundary node host names should be transparent for the addressing of HTTP resources in order to be able to specify the resource independent of a specific boundary node host name. This standard introduces a new `ic-http` URI scheme which abstracts from the host name of the boundary node and thereby allows for uniquely referring to HTTP(S)-exposed resources on the Internet Computer without reference to the boundary node's host name. This is crucial in the light of the future architecture of ICP because with the opening up of the Internet Computer's edge infrastructure by allowing community-operated boundary nodes, there will be many host names available to access the same resource.

The above example would give the following URI using ICRC-91:
`ic-http://dlbnd-beaaa-aaaaa-qaana-cai/tokens/12345678/image`

> [!NOTE]
> FIX: We should discuss to add a network id, e.g., (a prefix of) the hash of the DER-encoded network public key as unique identifier; this is important when we have private ICP-based networks or public testnets; we can have a default of ICP mainnet when omitting it to keep it more convenient for the majority of use cases. The network could be added as top-level element in the authority hierarchy (`737ba355e855` is a prefix of the pub key hash) (we may want to add the full pubkey hash for reasons of security in order to not have collision potential with an adversarial's network)
`ic-http://dlbnd-beaaa-aaaaa-qaana-cai.737ba355e855/tokens/12345678/image`

For example, an NFT standard can use this mechanism to specify HTTP-based canister-hosted resources of NFTs as part of NFT metadata. Parties who read the metadata can transform the `ic-http` URIs to HTTP URIs with their preferred boundary node provider and load the resources with those URIs.

This standard does not specify how to resolve a `http-ic` URI to a specific `https` URI that can be accessed in a browser. This is left to future ICRC standards that are expected to evolve once the boundary node ecosystem is growing with further community participation. Such future standard would need to compose an `https` URI from an `http-ic` URI based on the boundary node provider preference, and based on the preference choose a hostname. Not all boundary node providers will provide access to each canister on ICP. Thus, this constrains the transformation function from `http-ic` URIs to `https` URIs. A simple way to realize such transformation is with a registry of boundary node hostnames and canisters accessible through each boundary node to be available.


## Background

[RFC-3986 \[BFM05\]](https://datatracker.ietf.org/doc/html/rfc3986) specifies the syntax and semantics of URIs. The following grammar defines valid URIs:

```
    URI = scheme ":" hier-part [ "?" query ] [ "#" fragment ]

    hier-part = "//" authority path-abempty
        / path-absolute
        / path-rootless
        / path-empty
```

For a new URI scheme, like also for the `ic-http` scheme, a subset of the language defined by this grammar needs to be defined.


## Specification

The `ic-http` scheme is used to locate blockchain network resources exposed via the `https` protocol by ICP canisters independently of the hostname of the boundary nodes via which the resources can be accessed.

We follow the specificiation of URI schemes being lowercase according to [RFC-3986, Section 3.1](https://datatracker.ietf.org/doc/html/rfc3986#section-3.1).

The `ic-http` scheme defined in this standard has the following grammar expressed following the formal language put forth in RFC-5234:

```
    IC-HTTP-URI = "ic-http" ":" ic-http-hier-part [ "?" query ] [ "#" fragment ]

    ic-http-hier-part   = "//" ic-canister-principal ic-http-path

    ic-canister-principal = 9*9( 5*5( base32char ) "-" ) 1*2( base32char )
        / 9*9( 5*5( base32char ) )
        / 1*8( 5*5( base32char ) "-" ) ( 1*5( base32char ) )
        / 1*5( base32char )

    ic-http-path = path-abempty    ; begins with "/" or is empty
        / path-absolute   ; begins with "/" but not "//"
        / path-empty   ; zero characters

    path-abempty  = *( "/" segment )
    path-absolute = "/" [ segment-nz *( "/" segment ) ]
    path-empty    = 0<pchar>
    segment       = *pchar
    segment-nz    = 1*pchar
    segment-nz-nc = 1*( unreserved / pct-encoded / sub-delims / "@" )
        ; non-zero-length segment without any colon ":"

    query         = *( pchar / "/" / "?" )

    fragment      = *( pchar / "/" / "?" )

    unreserved    = ALPHA / DIGIT / "-" / "." / "_" / "~"
    pct-encoded   = "%" HEXDIG HEXDIG
    sub-delims    = "!" / "$" / "&" / "'" / "(" / ")"
        / "*" / "+" / "," / ";" / "="
    pchar         = unreserved / pct-encoded / sub-delims / ":" / "@"
    base32char    = base32letterlowercase / base32letterlowercase / base32digit
    base32letterlowercase = a-z
    base32letteruppercase = A-Z
    base32digit   = 2-7
```

We refer the reader to [RFC-3986 \[BFM05\]]([https://datatracker.ietf.org/doc/html/rfc3986](https://datatracker.ietf.org/doc/html/rfc3986#section-3.4)) for details on the components of a URI and [RFC-5234 \[CO08\]](https://datatracker.ietf.org/doc/html/rfc2234) for the definition of the productions in the above grammar that are not defined explicitly, such as `ALPHA` or `DIGIT`.


## Examples

The following is an example for a regular `https` URI. The problem is that it includes the domain name for boundary nodes and thus is not generically referring to the resource on ICP, but referring to the resource through this specific boundary node domain:
`https://d6g4o-amaaa-aaaaa-qaaoq-cai.ic0.app/token/12345678/image?format=jpeg&res=high&hash=e18c9d041a3b66b794a37c52a49d1f4c9173c8aeadd6fc4cb1f3677d66873ddd`

The same URI expressed using the approach using an `ic-http` URI results in the following URI:
`ic-http://d6g4o-amaaa-aaaaa-qaaoq-cai/token/12345678/image?format=jpeg&res=high&hash=e18c9d041a3b66b794a37c52a49d1f4c9173c8aeadd6fc4cb1f3677d66873ddd`
This URI expresses the resource on ICP in a generic way independent of any boundary node domain.

FIX further examples to show different use cases we consider important


## Conclusions

With the boundary node edge infrastructure being opened up to community participants, additional sets of boundary nodes with their own host names are expected to emerge. This ICRC standard is intended to unify the addressing of `https`-exposed resources of canister smart contracts on the Internet Computer. It abstracts away the hostname of the boundary node used so far in `https` URIs to address resources. This is important for any use case that references `https`-exposed canister resources.


## Future Work

The current proposal has not yet been registered with [IANA's URI scheme registry](https://www.iana.org/assignments/uri-schemes) \[IANA\] to make it an official URI scheme. This is a future step that should be taken once the approach has been sufficiently validated in reference implementations and found to be fit for the intended purpose. The `ic-http` URI scheme can already be used before registration to prove its usefulness.


## References

* [BFM05] T. Berners-Lee, R. Fielding, L. Masinter: Uniform Resource Identifier (URI): Generic Syntax. 2005, https://datatracker.ietf.org/doc/html/rfc3986
* [CO08] D. Crocker, Ed., P. Overell: Augmented BNF for Syntax Specifications: ABNF. 2008, https://datatracker.ietf.org/doc/html/rfc5234
* [IANA] IANA URI scheme registry. https://www.iana.org/assignments/uri-schemes
* [DFI24] DFINITY Foundation, The HTTP Gateway Protocol Specification. Accessed July 2024, https://internetcomputer.org/docs/current/references/http-gateway-protocol-spec
