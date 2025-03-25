| ICRC |                   Title                    |                                      Author                                      |                Discussions                | Status |      Type       | Category |  Created   |
|:----:|:------------------------------------------:|:--------------------------------------------------------------------------------:|:-----------------------------------------:|:------:|:---------------:|:--------:|:----------:|
|  97  | Non-Fungible Token (NFT) Metadata Standard | Thomas (@sea-snake), Austin Fatheree (@skilesare), Dieter Sommer (@dietersommer) | https://github.com/dfinity/ICRC/issues/97 | Draft  | Standards Track |  Tokens  | 2024-08-13 |

# ICRC-97: Non-Fungible Token (NFT) Metadata Standard

ICRC-97 defines a **user-facing metadata** standard for Non-Fungible Tokens (NFTs) on the Internet Computer. This standard aligns with ERC-721 and Metaplex JSON metadata formats to enhance interoperability across ecosystems.

## ICRC-7 Token Metadata

The [`icrc7_token_metadata`](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#icrc7_token_metadata)
method returns the token metadata for `token_ids`, represented as a list of (text, Value) pairs. Here, Value is recursive as defined in the ICRC-3 standard.

ICRC-97 specifies a metadata property pointing to one or more URIs serving the metadata JSON.

| Property          | ICRC-3 Type                            |
|-------------------|----------------------------------------|
| `icrc97:metadata` | `variant { Array = variant { Text } }` |

## JSON Metadata Properties

The following properties are defined for token metadata under the ICRC-97 standard. These fields are designed to be user-facing and provide essential information about the NFT.

| Property        | Required | Description                                                                                                                                                                                                                                                                                                                                        |
|-----------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `name`          | Yes      | The name of the NFT e.g. "CoolToken #1"                                                                                                                                                                                                                                                                                                            |
| `description`   | Yes      | A detailed description of the NFT or token. It can describe its history, rarity, use case, or special properties.                                                                                                                                                                                                                                  |
| `image`         | Yes      | URI pointing to the primary image representing the NFT. Supported formats include **PNG**, **JPEG**, and **SVG**.                                                                                                                                                                                                                                  |
| `animation_url` | No       | URI pointing to interactive NFT. This property supports the following file extensions and functionalities:<ul><li>**GLB**: Formats for 3D models.</li><li>**WEBM**, **MP4**: Video formats.</li><li>**WAV**, **OGG**, **MPEG**: Audio Formats.</li><li>**HTML**: For creating interactive NFTs using JavaScript canvas, WebGL, and more.</li></ul> |
| `external_url`  | No       | URI pointing to an external URL defining the asset — e.g. the game's main site.                                                                                                                                                                                                                                                                    |
| `attributes`    | No       | Array of attributes defining the characteristics of the asset.<ul><li>`trait_type`: The type of attribute.</li><li>`value`: The value for that attribute.</li></ul>                                                                                                                                                                                |

The attribute fields supports an optional `display_type` to enhance the visual representation of certain traits.

| Display Type       | Description                                                                              | Example                    |
|--------------------|------------------------------------------------------------------------------------------|----------------------------|
| `number`           | Displays the attribute as a numeric value.                                               | `"Level": 5`               |
| `boost_percentage` | Displays the attribute as a percentage-based boost (common for gaming NFTs).             | `"Health Boost": +85%`     |
| `boost_number`     | Displays the attribute as a numeric boost (e.g., additional points in a game).           | `"Strength Boost": +10`    |
| `date`             | Displays the value as a human-readable date interpreted from a Unix timestamp (seconds). | `"Birthday": "2021-08-01"` |
| Default (no type)  | If `display_type` is not specified, the attribute is displayed as a text-based trait.    | `"Background": "Blue"`     |

Adding an optional `max_value` sets a ceiling for a numerical trait's possible values. If you set a `max_value`, make sure not to pass in a higher value.

While additional metadata properties and other file types (beyond those specified in this standard) are technically allowed, there is a risk that some platforms or services may not support them fully. This could lead to incompatibilities or issues with rendering/displaying the metadata across different systems.

## URI Storage Recommendations

To reduce risks associated with off-chain storage, developers can adopt one of the following decentralized approaches. Multiple URIs can be provided for resilience (clients can attempt the next one if the first fails).

### 1. Basic Canister Approach:

Embed JSON metadata directly within a URL using a DATA URI for fully on-chain storage.

```candid
vec {
    record {
         "icrc97:metadata"; 
         variant { Array = vec {
             variant { Text = "data:text/json;charset=utf-8;base64,ew0KICAgImtleSIgOiAidmFsdWUiDQp9" };
         }; };
    }
}
```

### 2. Multi-Canister Approach (Advanced)

Store JSON metadata on the same canister or another certified canister, served via HTTPS. The ICRC-91 standard improves decentralization by removing the `ic0.app`/`icp0.io` dependency.

```candid
vec {
    record {
         "icrc97:metadata"; 
         variant { Array = vec {
             variant { Text = "ic-http://2225w-rqaaa-aaaai-qtqca-cai/metadata/3456" };
             variant { Text = "https://2225w-rqaaa-aaaai-qtqca-cai.icp0.io/metadata/3456" };
         }; };
    }
}
```

### 3. Bridging NFTs from Other Chains

Point to their metadata on **IPFS**, a common decentralized file system.

```candid
vec {
    record {
         "icrc97:metadata"; 
         variant { Array = vec {
             variant { Text = "ipfs://bafybeihkoviema7g3gxyt6la7vd5ho32ictqbilu3wnlo3rs7ewhnp7lly" };
             variant { Text = "https://ipfs.io/ipfs/bafybeihkoviema7g3gxyt6la7vd5ho32ictqbilu3wnlo3rs7ewhnp7lly" };
         }; };
    }
}
```