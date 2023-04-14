### Uniques Attributes metadata format

##### JSON Schema
```json
{
    "title": "uniques-attributes",
    "type": "object",
    "properties": {
        "attributes": {
            "type": "array",
            "items": {
                "type": "array",
                "items": {
                    "type": "string",
                },
                "minItems": 2,
                "maxItems": 2,
            },
            "uniqueItems": true,
            "description": "key-value pairs describing the NFT attributes",
        }
    }
}
```

###### JSON Schema notes

* The first element of the nested array is an attribute's `key`, and the second one is the `value`.

##### Rust representation
```rust
/// Note: Any other representation with the same SCALE encoding is compatible. 

#[derive(Encode, Decode)]
struct KeyValue {
    key: frame_support::BoundedVec<u8, /* chain-specific key limit */>,
    value: frame_support::BoundedVec<u8, /* chain-specific value limit */>
}

#[derive(Encode, Decode)]
struct UniquesAttributes {
    /// key-value pairs describing the NFT attributes
    attributes: frame_support::BoundedVec<KeyValue, /* chain-specific key-value pairs limit */>
}
```

##### Chain-specifics
 * The maximum size of a `key`.
 * The maximum size of a `value`.
 * The maximum size of the `attributes`. 

##### Example

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
