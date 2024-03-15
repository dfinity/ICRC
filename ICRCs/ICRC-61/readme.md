# ICRC-61 Supported Standards Generalization

|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|61|Supported Standards Generalization|Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer), Mario Pastorelli (@MarioDfinity)|https://github.com/dfinity/ICRC/issues/61|Draft|Standards Track||2024-03-05|

The ICRC-61 is a standard aimed at simplifying the discovery of supported standards by canisters on the Internet Computer. By providing a unified method, `icrc61_supported_standards`, canisters can easily expose the standards they implement, enhancing interoperability and easing integration efforts across the ecosystem.

## Data

The `icrc61_supported_standards` method will return a list of standardized records, each corresponding to a supported standard by the canister. The response record format of this method is structured as follows:

```candid "Type definitions" +=
type SupportedStandard = record { name : text; url : text; };
type SupportedStandardsResponse = vec SupportedStandard;
```

### text field

For ICRC standards the text SHOULD be in the form `ICRC-X` where X is the official ICRC number assigned at https://https://github.com/dfinity/ICRC/issues. New ICRC numbers can be procured by filing an issue at https://github.com/dfinity/ICRC/issues/new or via pull request(Issues are recommended for discoverability).

For non-ICRC standards, the text SHOULD be a human-readable namespace with a small likelihood of collision. The Internet Computer shares a global namespace. Please be polite to other developers and make namespaces specific. See https://forum.dfinity.org/t/prefix-all-the-methods-of-the-icrc-1-token-standard-with-icrc1/13865 for more discussion.

Suggestions:

- com.search.searchable
- org.icdevs.donatable
- eth.ecrc20.addressable

### url field

The url SHOULD point to a reliable URL that explains the standard.

## Methods

### icrc61_supported_standards <span id="supported_standards_method"></span>

Standard developers MUST implement the icrc61_supported_standards query endpoint.

Returns a list of standards supported by the canister.

This method is for discovering what APIs or protocols a given canister complies with, making it easier for developers and applications to interact with a broad range of services on the Internet Computer without prior, detailed knowledge of each service's capabilities. 

```candid "Methods" +=
icrc61_supported_standards : () -> (SupportedStandardsResponse) query;
```

The result MUST include entries for `ICRC-1` if the canister supports these standards. This mandate ensures that clients querying for supported standards can upgrade to ICRC-61 for discoverability. 

The result MUST include a self-reference for `record{name="ICRC-61"; url="https://github.com/dfinity/ICRCs/ICRC-61"}`;

Additionally, the result SHOULD include entries for other supported standards, offering a flexible and extensible way to advertise capabilities.

## Reasoning

The rationale behind the `icrc61_supported_standards` method over implementing similar endpoints within ICRC-1 individually is that it alleviates the need for developers to adhere to the full specifications of either standard to disclose supported features. This approach streamlines the integration process, broadens the compatibility across different protocols, and fosters a more interconnected and versatile ecosystem.

In effect, `icrc61_supported_standards` serves as a universal discovery mechanism, enabling canisters to succinctly communicate their supported interfaces. This capability is particularly valuable in a decentralized environment like the Internet Computer, where discovering and leveraging the functionalities of various services can significantly enhance application development and user experiences.

Through standardization of the `icrc61_supported_standards` method, the ICRC-61 aims to simplify interoperability, reduce implementation complexity, and foster a more vibrant and accessible ecosystem of services on the Internet Computer.

## Migration Path for Ledgers Using ICRC-1

Ledgers and other services that already exist, to the extent that they are capable, SHOULD add the ICRC-61 to increase interoperability on the Internet Computer.

For historical interoperability, existing ledgers MUST maintain their implementation of  `icrc1_supported_standards` .  The new `icrc61_supported_standards` should return at least same data, but MAY be augmented with additional ICRC features or supported standards.

## Versioning

Generally, new versions and updates of existing standards SHOULD be assigned a new ICRC number as opposed to attempting to support versioning through this interface.

## Future Work

A  potential area for improvement is the establishment of a registry or directory service on the Internet Computer that indexes canisters by the standards they support, using the ICRC-61 mechanism. Such a service could dramatically improve discoverability and foster a more interconnected ecosystem of canisters.
