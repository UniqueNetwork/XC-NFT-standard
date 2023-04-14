### General Metadata Container Format

There are several NFT metadata formats, and there could be many new formats in the future.

To transfer an NFT to another chain and to be free in choosing a metadata format,
some meta-format is needed to "tunnel" different NFT metadata formats.

The possible meta-format a.k.a **metadata container** may be the following:

##### JSON Schema
```json
{
    "title": "Metadata Container",
    "type": "object",
    "properties": {
        "hashingAlgorithmId": {
            "type": "integer",
            "minimum": 0,
            "maximum": 255,
            "description": "the hashing algorithm identifier to use for verifying the authenticity of the metadata container",
        },
        "standard": {
            "type": "string",
            "maxLength": 32,
            "description": "the standard describing the payload format",
        },
        "version": {
            "type": "string",
            "maxLength": 8,
            "description": "the version of the standard",
        },
        "payload": {
            "type": "object",
            "description": "the actual metadata object defined in the standard's format",
        },
    },
}
```

###### JSON Schema notes

The hashing algorithm identifier's meaning is defined as follows:

* `0` represents the Blake2 hashing algorithm with 256-bit digest.
* if the `hashing algorithm identifier` has any other values, it is invalid.

Note: new hashing algorithms could be added later.

You can read about the hashing algorthm's purpose in the [Immutable XC-NFT Standard](./immutable-xc-nft-standard.md).

##### Rust representation
```rust
/// Note: Any other representation with the same SCALE encoding is compatible. 

#[derive(Encode, Decode)]
enum HashingAlgorithm {
    Blake2_256, // encodes as `0`
}

#[derive(Encode, Decode)]
struct MetadataContainer {
    hashing_algorithm: HashingAlgorithm,
    standard: frame_support::BoundedVec<u8, ConstU32<32>>,
    version: frame_support::BoundedVec<u8, ConstU32<8>>,
    payload: frame_support::BoundedVec<u8, /* chain-specific max payload size */>,
}
```

##### Chain-specifics

* The maximum size of the `payload`.

##### Encoding

The metadata container should be [SCALE](https://github.com/paritytech/parity-scale-codec)-encoded into a blob to be transferred across chains.
The destination chain should decode the metadata container blob to examine the `standard` and the `version` fields to verify if the `payload`'s content is supported on this particular chain.

If the payload is supported, the destination chain should decode bytes from the `payload` according to the given standard.

##### Examples

###### ERC-721 on-chain metadata format
See [the format's definition](./metadata-formats/erc721.md).

*A "Magic Sword" NFT*
```json
{
    "hashingAlgorithmId": 0,
    "standard": "erc721",
    "version": "1",
    "payload": {
        "tokenURI": "https://game.example/magic-sword.png",
    },
}
```

*Scale-encoded*: `0x018657263373231431949068747470733a2f2f67616d652e6578616d706c652f6d616769632d73776f72642e706e67`

###### Uniques Attributes metadata format
See [the format's definition](./metadata-formats/uniques-attributes.md).

*A "Magic Sword" NFT with 3 attributes: `name`, `kind`, and `damage`.*
```json
{
    "hashingAlgorithmId": 0,
    "standard": "uniques-attributes",
    "version": "1",
    "payload": {
        "attributes": [
            ["name", "Magic Sword"],
            ["kind", "rare"],
            ["damage", "100"],
        ],
    },
}
```

*Scale-encoded*: `0x048756e69717565732d617474726962757465734319cc106e616d652c4d616769632053776f7264106b696e6410726172651864616d616765c313030`
