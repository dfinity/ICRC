# Approaches to ICRC-7 NFT User Facing Metadata

We identified three approaches for handling ICRC-7 NFT user facing metadata, each with distinct trade-offs in terms of interoperability, simplicity, and on-chain accessibility.

---

## 1. On-Chain ICRC-3 Metadata (Structured Fields)

This approach involves defining metadata directly within ICRC-3 fields, allowing canisters to access key information without additional fetching or parsing.

### Pros:

- **Direct Access**: Simplifies on-chain metadata retrieval (e.g., image URLs), as canisters can access the data directly.
- **Efficient Processing**: Avoids the overhead and complexity of fetching and parsing JSON.

### Cons:

- **Limited Interoperability**: Incompatible with NFT metadata standards from other blockchains.
- **Cross-Chain Maintenance**: Cross-chain collections would need metadata mapping, leading to additional storage and maintenance overhead.
- **Tooling Gaps**: Existing NFT tools won’t work out-of-the-box, and new IC-specific tools would need to be developed.
- **Wallet Complexity**: Cross-chain wallets would need to implement custom IC logic to handle the metadata format.

---

## 2. URL-Based JSON Metadata

In this approach, metadata consists of a URL pointing to a JSON file, which follows common NFT metadata standards used across multiple ecosystems.

### Pros:

- **Cross-Chain Compatibility**: Aligns with existing NFT metadata standards, enabling seamless integration with other blockchains.
- **Tooling Reuse**: Leverages existing NFT tools, wallets, and marketplaces, simplifying NFT creation on the IC.
- **Metadata Reusability**: Cross-chain collections can reuse the same metadata across different chains.
- **Simpler Wallet Integration**: Wallets can adopt IC-based NFTs without needing special implementations.

### Cons:

- **Indirect Access**: Canisters must fetch and parse the JSON file to access metadata, adding some overhead and complexity.
- **Potential Off-Chain Risks**: If not implemented properly, metadata may be stored off-chain (e.g., via centralized servers), unless decentralized alternatives like DATA URLs, ICRC-91 URLs, or IPFS are used.

---

## 3. Hybrid Approach (Supporting Both Formats)

This approach combines on-chain structured metadata with URL-based JSON, giving developers the flexibility to choose the format that best suits their needs.

### Pros:

- **Flexible Metadata Options**: Developers can optimize metadata for either on-chain access or cross-chain compatibility, depending on their use case.

### Cons:

- **Increased Complexity**: Wallets, marketplaces, and dapps must support and manage two metadata formats.
- **Canister Overhead**: Canisters that process metadata may need to handle both formats or risk being incompatible with certain collections.

---

## Summary

The right metadata approach depends on how the data is primarily consumed:

- **On-Chain Metadata (ICRC-3 Structured Fields)**: Best for collections where canisters need direct, efficient access to metadata.
- **URL-Based JSON Metadata**: More practical for most cases, offering better compatibility with external ecosystems and existing tools.

## Current Working Group Recommendation

After evaluating the pros and cons, we believe that **URL-based JSON metadata** offers the best balance of flexibility, cross-chain compatibility, and tooling reuse for most NFT use cases.

To minimize the risks of off-chain storage, developers can adopt the following decentralized solutions:

- **Recommended Basic Canister Approach:**  
  Embed JSON metadata directly within the URL using a **DATA URI** to keep it fully on-chain and easily accessible.

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

  > **Notes**:
  > - This approach is only recommended for limited metadata, large data would limit the number of NFTs that can be queried at once.
  > - For larger amounts, dynamic or other advanced types of metadata, it's recommended to use below approach instead.


- **Recommended Advanced Multi-Canister Approach:**  
  Store JSON metadata on the same canister or another canister, certified and served over HTTPS. The ICRC-91 standard basically removes the `ic0.app`/`icp0.io` part from the URL, improving decentralization and flexibility.

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

  > **Notes**:
  > - The URL path is not a part of this standard and could be any provided value.
  > - ICRC-91 is still work in progress and is likely to change, the value here should be seen as a placeholder.
  > - Efforts are currently being made to align this standard with the ICRC namespacing and payment URI standards.
  > - Multiple entries can be provided for resiliency. If the first one is not compatible with a client, the next one can be attempted.
  > - HTTP cache headers can be used to optimize the number of incoming requests, particularly if metadata is immutable.


- **Recommended for Bringing NFTs from Other Chains**  
  Store metadata on IPFS, a common decentralized file system, to facilitate interoperability with NFTs from other blockchains.

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

  > **Notes:**
  > - Multiple entries can be provided for resiliency. If the first one is not compatible with a client, the next one can be attempted.
  > - In IPFS, the URI itself is a content identifier (CID), derived from the hash of the file's content, ensuring immutability and authenticity.

### Handling Canister-Specific Metadata:

If canisters need direct access to collection-specific metadata (e.g., a game character's level), developers can implement alternative entries with that data. These items should be namespaced to keep the global namespace clean and MAY be defined as alternative ICRCs.

```candid
vec {
    record {
        "icrc97:metadata"; 
        variant { Array = vec {
            variant { Text = "data:text/json;charset=utf-8;base64,ew0KICAgImtleSIgOiAidmFsdWUiDQp9" };
        }; };
    };
    record {
        "com.mygame.namespace";
        variant { Map = vec {
            vec {
                "level";
                variant { Nat = 6; };
            };
            vec {
                "name";
                variant { Text = "Rathgar the Wise" };
            };
            vec {
                "inventory";
                variant { Array = vec {
                    variant { Text = "Cooking Pot" };
                    variant { Text = "Asparagus" };
                    variant { Text = "Hot Chocolate" };
                }; };
            };
        }; };
    }
}
```

## Notes

Please see below notes that further details regarding our current viewpoint and approach:

- Initially we considered the hybrid approach to cover all use cases. Looking further into the consequences of such an approach, we discovered it would make implementation and adoption a significant challenge. Two approaches in a single standard would need result in developers need to implement and maintain things twice everywhere.
- Trying to define a metadata standards that cover all use cases is a big goal but is likely to result in a standard that falls short for each individual use case. So instead we decided to focus on a single use case **user facing metadata**, this use case covers the data shown to end users within wallets and marketplaces.
- Interoperability with other chains and cross-chain wallets was an important requirement to make sure NFTs on the IC would gain more and wider adoption. Therefore, we think the JSON approach, which indeed seems less "IC" than candid, aligns better with this goal.
- Immutability, dynamic and other metadata topics are interesting and should definitely be covered. But from our viewpoint, these topics require further discussion with developers within the WG and the community. Meanwhile, these topics are not directly affecting the use case of **user facing metadata** shown by wallet and marketplaces.

## Join the discussion

Respond in this thread to join the discussion, additionally there's the Tokenization WG meeting every other Tuesday.

- Are there other better approaches we've missed?
- Disagree with our viewpoint? Awesome, share your thoughts!
- What metadata fields and values do we need? Do we follow existing standards? Which one?
- Input and discussions regarding technical details is highly appreciated.
  > **Example**: Are multiple URLs (fallbacks) needed and what is the risk of data being different between these URLs?