
# ICP Namespace

This ICRC standard defines the global ICP namespace. It is intended as the main namespace standard that defines and structures the ICP namespace at the highest level. Further ICRCs may define sub-namespaces in the future and may augment this namespace standard to define additional top-level namespacing elements.

# Specification

We define the `icp` URI scheme as the identifier for the ICP namespace. The scheme SHOULD be registered with IANA in the future, once eligible for registration.

We next define two classes of URIs used in the context of ICP:
1 URIs without an authority component
1 URIs with an authority component

## URIs without an authority component

```
icp:1:icrcY:<icrcY-defined-ns>
```

This kind of URIs starts with the prefix `icp` representing the root namespace, followed by the separator `:`, followed by the chain identifier, e.g., `1` standing for ICP mainnet, followed by the separator `:`. The following `icrcY` defines the sub-namespace of the icrc that is used to define the sub namespace, structured as the string `icrc` followed by the number `Y` of the ICRC standard, e.g., `icrc20` for the ICRC-20 standard. The ICRC standard defines the structure of the namespace behind the namespace identifier. `<icrcY-defined-ns>` in the example would expand to a concrete valid identifier within the `icrcY` sub-namespace.

`icp:1:icrc1:account:abcd`

The above example refers to the hypothetical ICRC-1-defined `account` sub-namespace that defines accounts independent of the ledger. This way, one can refer to an ICRC account in a globally unique manner. This example is conceptually compliant with the CAIP initiative to represent accounts in a globally-unique manner.

FIX: `icrcY | reverse domain name`

Identifiers other than `icrcY` referring to a specific ICRC standard SHOULD NOT be following the chain id without being defined in an ICRC. It is left to future standards to define additional use of the top-level namespace besides `icrc`s.

Authority-less URIs are applied in use cases that do not require an authority, such as a ledger-agnostic account identifier, or a protocol and chain identifier used as part of another URI. Such URIs can also be composed, e.g., a URI representing a token can be used as part of another URI to refer to this token. This kind of identifiers is particularly crucial for creating ICRC standards compliant with the Chain Agnostic standards (CAIP). FIX references;

An authority-less URI cannot be generically interpreted by a generic client such as a (modified) browser as is the case for authority-ful URLs. Interpretation of authority-less URIs is always dependent on the ICRC that defines the sub-namespace.

FIX elaborate on using slash (`/`) as separator and differences to using `:`; in common usage, `/` is used when referring to resources, while `:` is used in identifiers

Depending on how the chain id is expressed, the following URIs are equivalent to the above URI. The first one uses the hash of the NNS public key of ICP mainnet, the second one leaves out the identifier and thus defaults to mainnet.
```
icp:737ba355e855bd4b61279056603e0550:icrcY:<icrcY-defined-ns>
icp::icrcY:<icrcY-defined-ns>
```

## URIs with an authority component

An authority in a URI is the hierarchical component following the `://` following the scheme. It specifies a hierachical name managed by a hierarchy of naming authorities. A name resolves to a network address using systems like the well-known Domain Name System (DNS), or the decentralized alternative Canister Name System (CNS).

```
icp://application.app/icrcy/<icrcY-defined-ns>
```

The above shows the basic structure of a URI with an authority component: It starts with the `icp` scheme, followed by `://`, then followed by the authority component, the CNS name, e.g., `application.app` in the example. This is followed by the ICRC sub-namespace specifier `icrcY` for some ICRC-Y, followed by a slash `/`. This is then followed by the substring as defined by the ICRC-Y standard, with `Y` the number of the respective ICRC standard.

*Rationale on the absence of a network identifier:* ICP networks such as ICP mainnet, future testnets, and UTOPIA private networks are conceptually similar to networks comprising the larger Internet. Modelling ICP networks analogous to intranets in the DNS world implies to have a single naming system akin to DNS for all ICP networks. As in the world of the Internet and DNS, a single naming system for all ICP networks makes most sense in terms of how to model names. Thus, identifiers with authority component do, much like URLs on the Internet, not need a network identifier.

There should be a global CNS name registration system applicable to all ICP-based networks with its core managed on ICP mainnet and sub-authorities being hosted on any ICP network, analogous to how DNS decentralizes the authority over different parts of the namespace. This is well aligned to the thinking of people w.r.t. names. CNS names should not overlap with DNS names, i.e., someone claiming a CNS name should reserve the same name on DNS if the TLD exists in DNS.

Other identifier than valid `icrcY` Other identifiers as root of the resource path may be used at the discretion of the owner of the namespace defined by the authority, unless reserved through an ICRC standard existing at the time of use. Future ICRC standards may define additional uses of the namespace outside of ICRC standards.

The authority component MUST be resolved with CNS. FIX: Is this too restrictive? We only have DNS right now. CNS cannot be used without the browser (or an extension) interpreting it. This requires further discussion. There are many use cases using DNS (all use cases currently), so we may want to support it as well. The problem is that if we support both, it is unclear which one to use in a given context (or we use DNS only for http URLs). Idea: If the client is CNS aware, try resolving a URI with CNS, if this fails, try resolving with DNS. Which attacks might this result in? E.g., someone could register CNS names in the DNS system and get the user to load exploit web sites. Reserving a CNS address also in the DNS system (partially) resolves this issue as then the same party controls both addresses. Means that DNS addresses could not be used in CNS (makes some sense also, as this would be very confusing).
The question whether and how to also support DNS requires substantial further discussion.

We can refer to a canister by using the canister principal as authority:
```
icp://ormnc-tiaaa-aaaaq-aadyq-cai/icrcY/<icrcY-defined-ns>
```

Referring to a canister through only its principal, e.g., `ormnc-tiaaa-aaaaq-aadyq-cai`, is much more powerful as global identifier for a canister than the current use of a boundary node domain name along with the canister id, such as ```ormnc-tiaaa-aaaaq-aadyq-cai.ic0.app```. The latter constrains the validity of the URI only to the specific specified DNS name of boudary nodes, although it should semantically not be dependent on this, while the former removes this limitation.

Note that this does not work out of the box in a standard browser as it interprets the string as DNS name. // FIX We need to discuss the implications of this for this ICRC.

Authority-based URIs are useful to express concepts that are related to an authority, e.g., a ledger canister. For example, a token of a specific ledger can be referred to with an authority-ful URI by using the ledger canister principal as authority in the URI. A substantial subset of authority-based URIs (the ones that are URLs) can be resolved (using DNS or CNS) to a network address on which the URI has a well-defined meaning, e.g., performing a transaction on a canister or loading a resource. Not all URIs need to be resolvable like this, though, and can simply act as well-defined identifier as is typical for URIs.


# References

* [INR25] ICP Network Registry
* [INI25] ICP Network Identifiers
* [CAIP] CAIP
