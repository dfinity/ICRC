| ICRC |                   Title                    |                                      Author                                      |                Discussions                | Status |      Type       | Category |  Created   |
|:----:|:------------------------------------------:|:--------------------------------------------------------------------------------:|:-----------------------------------------:|:------:|:---------------:|:--------:|:----------:|
|  97  | Non-Fungible Token (NFT) Metadata Standard | Thomas (@sea-snake), Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer) | https://github.com/dfinity/ICRC/issues/97 | Draft  | Standards Track |          | 2024-08-13 |

# ICRC-97: Non-Fungible Token (NFT) Metadata Standard

ICRC-97 is a metadata standard for the implementation of Non-Fungible Tokens (NFTs) on
the [Internet Computer](https://internetcomputer.org).

The standard is designed to allow for multiple assets with different data types with various purposes. Where additional
asset purposes and attribute display types can be standardized in ICRC-97 extensions.

A list of asset purposes and attribute display types are defined in ICRC-97 to the minimum needed for wallets and marketplaces to display NFTs.

## Extending the ICRC-7 Standard with Token Metadata

The [`icrc7_token_metadata`](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#icrc7_token_metadata)
method returns the token metadata for `token_ids`, a list of token ids. Each tuple in
the response vector comprises an optional `metadata` element with the metadata expressed as vector of `text` and `Value`
pairs.

ICRC-7 does not specify the representation of token metadata any further than that it is represented in a generic manner
as a vector of (text, Value)-pairs. The ICRC-97 standard defines a token metadata
standard for ICRC-7 and other NFT standards.

### Entrypoint

The following metadata property MUST be defined in the root of the token metadata if we want to return on-chain
metadata.

| Root Metadata Property | ICRC-3 Type     | Description                            |
|------------------------|-----------------|----------------------------------------|
| icrc97:metadata        | variant { Map } | Contains on-chain metadata properties. |

Alternatively the following metadata property MUST be defined in the root of the token metadata if we want to return
external
off-chain metadata instead of on-chain metadata.

| Root Metadata Property   | ICRC-3 Type     | Description                                       |
|--------------------------|-----------------|---------------------------------------------------|
| icrc97:external_metadata | variant { Map } | Contains below external JSON metadata properties. |

| External Metadata Property | Optional | ICRC-3 Type      | Description                                                                    |
|----------------------------|----------|------------------|--------------------------------------------------------------------------------|
| url                        | No       | variant { Text } | URL that returns the metadata in JSON format (protocol is not limited to HTTP) |
| sha256_hash                | Yes      | variant { Blob } | SHA-256 hash of HTTP response body bytes returned from above url.              |

### Metadata properties

Below table list all (optional) metadata properties defined in this standard. Each property is defined in both ICRC-3
and JSON type to support both on-chain `Value` and off-chain JSON metadata respectively.

| Metadata Property | Optional | ICRC-3 Type                             | JSON Type | Description                                             |
|-------------------|----------|-----------------------------------------|-----------|---------------------------------------------------------|
| external_url      | Yes      | variant { Text }                        | string    | URL that allows the user to view the item on your site. |
| name              | Yes      | variant { Text }                        | string    | Plain text.                                             |
| description       | Yes      | variant { Text }                        | string    | Markdown.                                               |
| assets            | Yes      | variant { Array = vec variant { Map } } | object[]  | List of assets ordered by priority descending.          |
| attributes        | Yes      | variant { Array = vec variant { Map } } | object[]  | List of attributes ordered by priority descending.      |

The following table list all (optional) asset properties.

| Asset Property | Optional | ICRC-3 Type      | JSON Type       | Description                                                                    |
|----------------|----------|------------------|-----------------|--------------------------------------------------------------------------------|
| url            | No       | variant { Text } | string          | URL that returns the asset e.g. a PNG image (protocol is not limited to HTTP). |
| mime           | No       | variant { Text } | string          | Mime type as defined in RFC 6838.                                              |
| sha256_hash    | Yes      | variant { Blob } | string (base64) | SHA-256 hash of HTTP response body bytes returned from above url.              |
| purpose        | Yes      | variant { Text } | string          | Indicate purpose of asset.                                                     |

Below list of purpose values are part of the ICRC-97 standard, this list could be extended by other standards.
Purpose values can define additional asset properties.

| Purpose Value  | Description                                                                                                                                                 |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| icrc97:image   | Original size image that is shown on e.g. item details page. Additional optional `width` and `height` properties define the dimensions in number of pixels. |
| icrc97:preview | Small image meant as preview within e.g. a list of items. Additional optional `width` and `height` properties define the dimensions in number of pixels.    |

The following table list all (optional) attribute properties.

| Attribute Property | Optional | ICRC-3 Type                | JSON Type        | Description                                 |
|--------------------|----------|----------------------------|------------------|---------------------------------------------|
| value              | No       | variant { Text; Nat; Int } | string \| number | Value of the trait.                         |
| trait_type         | No       | variant { Text }           | string           | Name of the trait.                          |
| display_type       | Yes      | variant { Text }           | string           | Indicate how attribute should be displayed. |

Below list of display types are part of the ICRC-97 standard, this list could be extended by other standards.
Display types can define additional attribute properties.

| Display Type Value      | Description                                                                                                                                                   |
|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| icrc97:property         | Shows attribute as property with e.g. rarity, this is the default display type.                                                                               |
| icrc97:date             | Shows attribute as date, expects epoch timestamp number in milliseconds.                                                                                      |
| icrc97:time             | Shows attribute as date with time, expects epoch timestamp number in milliseconds.                                                                            |
| icrc97:rank             | Shows attribute as progress rank e.g. 4 of 10, expects number and additional `max_value` property with a number.                                              |
| icrc97:stat             | Show attribute as stat e.g. 1 out of 2, expects number and additional `max_value` property with a number.                                                     |
| icrc97:boost            | Shows attribute as boost e.g. +10, expects number and optional additional `min_value` and `max_value` properties. Numbers can be either positive or negative. |
| icrc97:boost_percentage | Same as `icrc97:boost` but shows values with %.                                                                                                               |
