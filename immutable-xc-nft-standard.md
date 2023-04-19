# Immutable XC-NFT Standard

## Table of Contents
* [Introduction](#introduction)
* [Metadata transfer](#metadata-transfer)
    * [The transfer of the authentic metadata blob](#the-transfer-of-the-authentic-metadata-blob)
    * [Different metadata formats](#different-metadata-formats)
    * [The metadata hashing](#the-metadata-hashing)
* [Transfer process](#receiving-an-nft-on-a-non-reserve-chain)
    * [The Staging Area](#the-staging-area)
    * [The Clearance Procedure](#the-clearance-procedure)
    * [The XC-Transfer flow](#the-xc-transfer-flow)
* [Implementation considerations using Substrate and Polkadot XCM](#implementation-considerations-using-substrate-and-polkadot-xcm)
    * [Cross-chain NFT identification](#cross-chain-nft-identification)
    * [The Staging Area pallet](#the-staging-area-pallet)
    * [The XC-Transfer flow using the Staging Area pallet](#the-xc-transfer-flow-using-the-staging-area-pallet)
    * [The Staging Area pallet's pseudocode](#the-staging-area-pallets-pseudocode)

## Introduction

This document describes the standard of cross-chain transfers of immutable (unmodifiable) NFTs.

The standard assumes that any NFT is governed by its *originating chain*.
The *originating chain* is where the given NFT was originally minted, i.e., the NFT is *native* to the *originating chain*.

Only the *originating chain* has the authority on its NFTs; it is the only source of truth.
Therefore, the *originating chain* is the *reserve chain (R)* in terms of the reserve-based transfer model:

![image|100x100](https://cms.polkadot.network/content/images/2021/09/image-5.png)

>You can read about the reserve-based transfer model [here](https://polkadot.network/blog/xcm-the-cross-consensus-message-format#-some-initial-use-cases).

Even if an NFT was modifiable on the *originating chain* before the cross-chain transfer, the NFT **must** remain immutable when it is transferred to another chain since the derivative asset located on a non-reserve chain **must always** correspond to the real asset located on the reserve chain.

However, if the NFT is cross-transferred back to the reserve chain (meaning that the given NFT no longer backs a derivative asset), then the reserve chain **may** modify the NFT.

>Note: The teleportation of NFTs where two chains participating in the cross-chain transfer mutually trust each other is out of the scope of this document.

NFTs are identified between chains using some sort of *cross-chain NFT ids* (such as Polkadot XCM `AssetId`).

## Metadata transfer

NFTs are unique objects with associated data (the NFT metadata).
Unlike fungible tokens, transferring the non-fungible token itself is not enough because the valuable data is not the number of tokens (which is always 1 in the case of NFTs) but the metadata associated with them.

Since the NFTs participating in cross-transfers are immutable, the correct metadata is always located on the reserve chain.

However, there are still two remaining problems:
1. The transfer of the authentic metadata blob
2. Different metadata formats

### The transfer of the authentic metadata blob

Considering that on-chain mechanisms for cross-chain transfers (such as Polkadot XCM) could be limited in bandwidth, we can't send the potentially giant metadata blob via them. Hence we need to transfer the metadata blob off-chain.
However, we can still transfer some small-sized data on-chain to verify the authenticity of data provided off-chain. So, we should first transfer the *cross-chain NFT id* and data that compactly represent the NFT's metadata (the metadata hash) and only then transfer the metadata blob off-chain.

The *originating chain* should transfer the *cross-chain NFT id* and the hash representing the authentic metadata blob to the *destination non-reserve chain* via the cross-chain transfer mechanism such as Polkadot XCM.

Then, the destination chain should verify the data provided off-chain: compute the hash of the data and compare it to the metadata hash received from the *originating chain*.

If the computed hash is correct, the destination chain can mint the derivative NFT with the provided metadata.

### Different metadata formats

The *destination non-reserve chain* should be able to interpret the off-chain provided metadata blob correctly, so the destination chain should know the blob format ahead of time.

Nevertheless, several metadata formats exist, and more can appear in the future. Therefore, the metadata blob **must** be a correct *Metadata Container* that can contain various metadata formats inside.

For further details, see the [*Metadata Container* description](./metadata-container.md). 

### The metadata hashing

The hashing algorithm is defined by the [*Metadata Container's*](./metadata-container.md) `hashingAlgorithmId` field.
The hash **must** be computed based on the SCALE-encoded *Metadata Container*.

## Transfer Process

Since the metadata blob is transferred *after* the *cross-chain NFT id* and the metadata hash, the *destination non-reserve chain* doesn't know if the metadata's format is supported. Also, even if the format is supported, the destination chain can have restrictive rules regarding the NFT metadata contents.

Because of the above, there is a need to introduce the *Staging Area* and the *Clearance Procedure*.

### The Staging Area

The *Staging Area* is a dedicated place for *cross-chain NFT ids* and metadata hashes to be sent to another chain or for those received from the *originating chain*.

Although the *Staging Area* seems to be optional on the *originating chain*, it should have the *Staging Area* to separate concerns: when an NFT is cross-transferred to another chain, the "real" NFT on the *originating chain* should be moved to the *Staging Area* account; thus the NFT is "taken" from the business logic of the *originating chain*, and the *Staging Area* guarantees that NFT won't change while the NFT is backing a derivative on another chain (i.e., while it is not cross-transferred back to the *originating chain*).

Also, a chain can be the *originating chain* only for its native NFTs. Therefore, to accept foreign NFTs, such a chain also **must** have the Staging Area.

#### On a non-reserve chain

Derivative NFTs should be initially minted inside the *Staging Area* of the *destination non-reserve chain* to the *Beneficiary* of the cross-chain transfer. The *Staging Area* **must** contain information about who is the owner of the NFT (the owner is the *Beneficiary*).

While the derivative NFT is located inside the *Staging Area*, the owner can do exactly two mutually exclusive actions:
* Pass the derivative NFT through the *Clearance Procedure*.
* Cross-transfer the NFT somewhere else.

The derivative NFT **must** remain in the *Staging Area* until the owner either pass it through the *Clearance Procedure* or cross-transfers it to another chain.

Either of the two actions above **must** lead to removing the derivative NFT from the *Staging Area*.

#### On the *originating chain*

Before an NFT can be transferred to another chain, it should be moved to *Staging Area* account to give the *Staging Area* control of the NFT. The *Staging Area* guarantees that the NFT won't change while the NFT is backing a derivative on another chain.

The *Staging Area* **must** contain information about who is the owner of the NFT.

While the NFT is located inside the *Staging Area*, the owner can do exactly two mutually exclusive actions:
* Return the NFT to the regular NFT logic of the *originating chain*.
* Cross-transfer the NFT somewhere else.

The "return" operation shouldn't require the metadata checking since the NFT is native to the *originating chain*, and the *Staging Area* guarantees that the metadata didn't change (i.e., the *Clearance Procedure* **must not** be required).

The "return" action **must** lead to removing the derivative NFT from the *Staging Area*.
However, when the owner cross-transfers the NFT to another chain, the "real" NFT **must** remain in the *Staging Area* of the reserve chain until the NFT is cross-transferred back.

### The Clearance Procedure

When the derivative NFT is located in the *Staging Area*, the *destination non-reserve chain* can execute the *Clearance Precedure* of the given metadata blob for the given NFT on the *Beneficiery's* demand.

During the *Clearance Procedure*, the destination chain **must** compute the hash of the metadata blob using the algorithm defined by the `hashingAlgorithmId` field of the blob. Recall that the metadata blob **must** be a valid *Metadata Container*; otherwise, the *Clearance Procedure* **must** fail with an error and leave the derivative NFT in the *Staging Area* untouched.

If the metadata blob is a valid *Metadata Container* and the hash is computed, the destination chain **must** compare the computed hash against the hash stored inside the *Staging Area*.

If the hashes are not equal, the *Clearance Procedure* **must** fail with an error and leave the derivative NFT in the *Staging Area* untouched.

Otherwise, additional checks **may** be performed by the destination chain during the *Clearance Procedure*, such as payload content checks.

If all the checks are passed, the derivative NFT **must** be removed from the *Staging Area* and minted as a native NFT with the provided metadata.

Note 1: If the *Clearance Procedure* has failed, the derivative NFT is still in the *Staging Area*, meaning that the *Beneficiary* **can** send the NFT to another chain. Also, the *Beneficiary* **can** try to execute the *Clearance Procedure* again, hoping it will succeed this time.

Note 2: The exact mechanism of providing the metadata blob to the chain (to be checked during the *Clearance Procedure*) is arbitrary and is out of the scope of this document.

### The XC-Transfer flow

For a better description, let's provide examples of using the *Staging Area* and the *Clearance Procedure* combination in various scenarios.

Let's describe a journey of a single NFT, beginning with the *originating chain* (the reserve chain):
* A cross-transfer from the reserve chain to the non-reserve Chain A
* A cross-transfer from the non-reserve Chain A to the non-reserve Chain B
* A cross-transfer from the non-reserve Chain B back to the reserve chain

#### Reserve Chain → Non-reserve Chain A

##### On the reserve chain

1. Move the NFT to the *Staging Area* and compute the hash of the NFT's metadata container on-chain
2. Cross-transfer the NFT to the non-reserve Chain A:
    * make Chain A's sovereign account the owner of the NFT inside the *Staging Area* of the reserve chain
    * send the *cross-chain NFT id* and the computed metadata hash to Chain A

Note: After the cross-transfer, the NFT **must** remain in the *Staging Area* **of the reserve chain.**

##### On the non-reserve Chain A

1. Mint the NFT inside the *Staging Area* upon receiving the *cross-chain NFT id* and the metadata hash.
2. The derivative NFT remains in the *Staging Area* until one of the following actions is performed by the owner of the NFT:
    * The owner passes the NFT and the off-chain provided metadata blob through the *Clearance Procedure*. If the NFT has passed the *Clearance Procedure*, it is minted on Chain A in the native representation and removed from the *Staging Area*; thus, the cross-transfer is complete. Otherwise, if the *Clearance Procedure* has failed, the NFT remains in the *Staging Area*.  

    * The owner cross-transfers the NFT to another chain. The NFT is removed from the *Staging Area*.

#### Non-reserve Chain A → Another non-reserve Chain B

##### On Chain A

1. If the owner passed the NFT through the *Clearance Procedure*, they should first move the NFT to the *Staging Area*.
2. When the NFT is located inside the *Staging Area*, the owner can cross-transfer it to another chain. For instance, they can transfer the NFT to the non-reserve Chain B. Chain A asks the reserve chain to transfer the "real" NFT to Chain B. The derivative NFT **must** be burned on Chain A since the "real" NFT will no longer back it on the reserve chain.

##### On the reserve chain

Upon receiving the transfer request from Chain A, the reserve chain will update the owner of the NFT inside the *Staging Area* (Chain B's sovereign account will be the new owner) and send the *cross-chain NFT id* and the metadata hash to Chain B.

##### On Chain B

1. Mint the NFT inside the *Staging Area* upon receiving the *cross-chain NFT id* and the metadata hash.
2. The second step is the same as with Chain A when receiving an NFT.

#### Non-reserve Chain B → Reserve Chain

##### On Chain B

Both steps are the same as with Chain A when cross-transferring an NFT.

##### On the reserve chain

1. Upon receiving the transfer request from Chain B, the reserve chain will update the owner of the NFT inside the *Staging Area*.
2. The owner of the NFT can then either:
    * *Return* the NFT into the regular NFT logic of the reserve chain (hence, removing the NFT from the *Staging Area*).
    * Or cross-transfer the NFT to another chain.

## Implementation considerations using Substrate and Polkadot XCM

This section describes the considerations for implementing the standard using the `Substrate` and the `Polkadot XCM`.

It is **only an example**, not the standard itself. The standard is described in the above sections.

### Cross-chain NFT identification

To identify an asset between different chains, the Polkadot XCM uses the `AssetId` type. 

```rust
pub enum AssetId {
	/// A specific location identifying an asset.
	Concrete(MultiLocation),
	/// An abstract location; this is a name which may mean different specific locations on
	/// different chains at different times.
	Abstract([u8; 32]),
}
```

We can use the `Concrete` variant of the `AssetId` to identify an NFT.

For example, if an NFT could be identified on *some chain* using two numbers (a collection id and an item id), we could identify that NFT between chains like the following:
```rust
AssetId::Concrete(MultiLocation {
    parents: 1,
    interior: Junctions::X3(
        Parachain(SOME_CHAIN_PARA_ID),
        GeneralIndex(COLLECTION_ID),
        GeneralIndex(ITEM_ID),
    ),
})
```

Or if an NFT could be identified by an ERC-721 contract address and an item id:
```rust
AssetId::Concrete(MultiLocation {
    parents: 1,
    interior: Junctions::X3(
        Parachain(SOME_CHAIN_PARA_ID),
        AccountKey20 {
            network: OPTIONAL_NETWORK_ID,
            key: CONTRACT_ADDRESS,
        },
        GeneralIndex(ITEM_ID),
    ),
})
```

Thus, the structure of the `AssetId` for NFTs should consist of two parts:
* The reserve chain part: the first component of the `interior`
* the NFT id identifying the NFT on the reserve chain: the rest of the components of the `interior`

However, the XCM uses not only the `AssetId` during the cross-transfer. The XCM commands for transfers such as `WithdrawAsset`, `InitiateReserveWithdraw`, and `TransferReserveAsset` use the `MultiAsset` type.

```rust
pub struct MultiAsset {
	/// The overall asset identity (aka *class*, in the case of a non-fungible).
	pub id: AssetId,
	/// The fungibility of the asset, which contains either the amount (in the case of a fungible
	/// asset) or the *insance ID`, the secondary asset identifier.
	pub fun: Fungibility,
}

pub enum Fungibility {
	/// A fungible asset; we record a number of units, as a `u128` in the inner item.
	Fungible(#[codec(compact)] u128),
	/// A non-fungible asset. We record the instance identifier in the inner item. Only one asset
	/// of each instance identifier may ever be in existence at once.
	NonFungible(AssetInstance),
}
```

As stated in the doc-comment of the `fun` field, it is the secondary asset identifier in the case of non-fungibles. Such a secondary identifier should be the metadata hash for two reasons:
* We can easily send the metadata hash of the NFT simultaneously with its asset id.
* The metadata hash represents the state of the NFT. The NFT's state could be modified when the NFT is located on its *originating chain* (on the reserve chain). Having the metadata hash as a part of the cross-chain identifier, we can identify a particular NFT in a specific state between chains.

Since we should use the `AssetInstance` type for storing the NFT's metadata container hash, we can call the metadata hash the "instance hash".

### The Staging Area pallet

The *Staging Area* can be implemented as a dedicated pallet.

This pallet should have the following:
* Storage for NFTs identified by the `AssetId`. The storage should contain all the required information about NFTs, such as the owner and the instance hash
* An extrinsic that represents the *Clearance Procedure*

The *Clearance Procedure* extrinsic should compare the hash stored in the *Staging Area's* storage against the computed hash of the metadata blob provided via the extrinsic's argument.

Also, the pallet should have two additional extrinsics:
* one for moving the NFT to the *Staging Area* and computing the instance hash
* and another one for returning the NFT to a user if the *Clearance Procedure* is not required

The *Staging Area* pallet should use the NFT pallet (via an associated type from the pallet's `Config`) that is responsible for the standard NFT operations:
* mint NFTs
* transfer NFTs between different sovereign accounts
* transfer NFTs from a user's account to the *Staging Area*'s account and vice versa

Thus, the *Staging Area* pallet could be used with different NFT pallets, e.g., with Parity's `Nfts` pallet.

Moreover, there should be an implementation of the `TransactAsset` trait so that the XCM executor can act appropriately when dealing with NFT assets.
The *Staging Area* pallet can implement the `TransactAsset` trait itself to act like an XCM adapter for NFTs.  

The commented pseudocode of such a pallet is provided [below](#the-staging-area-pallets-pseudocode). It uses the Substrate's `nonfungibles_v2` traits to communicate with the NFT pallet and other standard XCM traits (such as `MatchesNonFungibles`).

Any non-standard trait or type is defined in the pseudocode itself.

### The XC-Transfer flow using the Staging Area pallet

To illustrate the usage of the *Staging Area* pallet, let's map each action described in [The XC-Transfer flow](#the-xc-transfer-flow) section to extrinsic calls of the *Staging Area* pallet and XCM Programs execution.

#### Reserve Chain → Non-reserve Chain A

##### On the reserve chain

1. The owner calls `prepare_for_xc_transfer`. This extrinsic will move the NFT into the *Staging Area* and compute the instance hash.
   It should be called with the following parameters:
    * `collection_id`: the collection id of the NFT
    * `item_id`: the item id of the NFT

2. Execute the cross-chain transfer XCM program. The program could look like this:
   ```rust
    // The `WithdrawAsset` will check if the NFT is located in the Staging Area.
    // And it will also check if the `instance_hash` is correct.
    // (See the `TransactAsset` trait implementation in the pallet's pseudocode).
    WithdrawAsset(vec![
        // We need to send fungible tokens to pay for execution
        MultiAsset {
            id: FEE_ASSET_ID,
            fun: FEE_AMOUNT,
        },
        MultiAsset {
            id: NFT_ASSET_ID,
            fun: Fungibility::NonFungible(AssetInstance::Array32(instance_hash)),
        },
    ]),

    // The `TransferReserveAsset` will modify the `owner` of the NFT in the Staging Area
    // and send the message to Chain A.
    TransferReserveAsset {
        assets: vec![
            MultiAsset {
                id: FEE_ASSET_ID,
                fun: FEE_AMOUNT,
            },
            MultiAsset {
                id: NFT_ASSET_ID,
                fun: Fungibility::NonFungible(AssetInstance::Array32(instance_hash)),
            },
        ],
        dest: MultiLocation {
            parents: 1,
            interior: X1(Parachain(CHAIN_A_PARA_ID)),
        },
        xcm: vec![
            // Buy execution on Chain A. 
            BuyExecution {
                fees: MultiAsset {
                    id: FEE_ASSET_ID,
                    fun: FEE_AMOUNT,
                },
                weight_limit: WeightLimit::Unlimited,
            },

            // Deposit the NFT (alongside of the remaining fungible tokens).
            //
            // The `DepositAsset` will create the NFT inside the *Staging Area*
            // and set its owner to the `BENEFICIARY_ACCOUNT`. 
            DepositAsset {
                assets: MultiAssetFilter::Wild(All),
                beneficiary: MultiLocation {
                    parents: 0,
                    interior: X1(AccountId32 {
                        network: None,
                        id: BENEFICIARY_ACCOUNT,
                    }),
                },
            },
        ].into(),
    }
   ```

##### On Chain A

1. After executing the `DepositAsset` command from the above, the NFT will be minted inside the *Staging Area*.
2. The derivative NFT remains in the *Staging Area* until one of the following actions is performed by the owner of the NFT:
    * The owner passes the NFT and the off-chain provided metadata blob through the *Clearance Procedure*. To do so, the owner should call the `check_in` extrinsic with the following parameters:
        * `xc_asset_id`: the `AssetId` of the NFT
        * `metadata_blob`: the SCALE-encoded metadata container of the NFT
    
    * The owner cross-transfers the NFT to another chain. See the 2nd step of the "Non-reserve Chain A → Another non-reserve Chain B" section.

#### Non-reserve Chain A → Another non-reserve Chain B

##### On Chain A

1. If the owner passed the NFT through the *Clearance Procedure*, they should first move the NFT to the *Staging Area*. To do so, the owner should call the `prepare_for_xc_transfer` extrinsic with the following parameters:
    * `collection_id`: the collection id of the NFT (the collection id corresponds to Chain A's native collection where the derivative NFT is located)
    * `item_id`: the item id of the NFT (the item id corresponds to Chain A's native id for the derivative NFT)

2. When the NFT is located inside the *Staging Area*, the owner can cross-transfer it to another chain. For instance, they can transfer the NFT to the non-reserve Chain B.
An XCM program like the following should be executed:
    ```rust
    // The `WithdrawAsset` will check if the NFT is located in the Staging Area.
    // And it will also check if the `instance_hash` is correct.
    //
    // Since Chain A is NOT the reserve chain, this command will burn the derivative NFT.
    // (See the `TransactAsset` trait implementation in the pallet's pseudocode).
    WithdrawAsset(vec![
        MultiAsset {
            id: FEE_ASSET_ID,
            fun: TOTAL_FEE_AMOUNT,
        },
        MultiAsset {
            id: NFT_ASSET_ID,
            fun: Fungibility::NonFungible(AssetInstance::Array32(instance_hash)),
        },
    ]),

    // The `InitiateReserveWithdraw` asks the reserve chain to transfer it to Chain B's user. 
    InitiateReserveWithdraw {
        assets: MultiAssetFilter::Wild(All),
        reserve: MultiLocation {
            parents: 1,
            interior: X1(Parachain(RESERVE_CHAIN_PARA_ID)),
        },
        xcm: vec![
            // Buy execution on the reserve chain. 
            BuyExecution {
                fees: MultiAsset {
                    id: FEE_ASSET_ID,
                    fun: RESERVE_CHAIN_FEE_AMOUNT,
                },
                weight_limit: WeightLimit::Unlimited,
            },

            // The `TransferReserveAsset` will modify the `owner` of the NFT in the Staging Area of the reserve chain
            // and send the message to Chain B.
            TransferReserveAsset {
                assets: vec![
                    MultiAsset {
                        id: FEE_ASSET_ID,
                        fun: CHAIN_B_FEE_AMOUNT,
                    },
                    MultiAsset {
                        id: NFT_ASSET_ID,
                        fun: Fungibility::NonFungible(AssetInstance::Array32(instance_hash)),
                    },
                ],
                dest: MultiLocation {
                    parents: 1,
                    interior: X1(Parachain(CHAIN_B_PARA_ID)),
                },
                xcm: vec![
                    // Buy execution on Chain B. 
                    BuyExecution {
                        fees: MultiAsset {
                            id: FEE_ASSET_ID,
                            fun: CHAIN_B_FEE_AMOUNT,
                        },
                        weight_limit: WeightLimit::Unlimited,
                    },

                    // Deposit the NFT (alongside of the remaining fungible tokens).
                    //
                    // The `DepositAsset` will create the NFT inside the *Staging Area*
                    // and set its owner to the `BENEFICIARY_ACCOUNT`. 
                    DepositAsset {
                        assets: MultiAssetFilter::Wild(All),
                        beneficiary: MultiLocation {
                            parents: 0,
                            interior: X1(AccountId32 {
                                network: None,
                                id: BENEFICIARY_ACCOUNT,
                            }),
                        },
                    },
                ].into(),
            }
        ].into(),
    }
    ```

##### On the reserve chain

During the execution of the `TransferReserveAsset` XCM command, the reserve chain will update the NFT owner inside the *Staging Area* (see the `TransactAsset::transfer_asset` fn). Then, the XCM executor will send the "deposit asset" XCM message to Chain B.

##### On Chain B

1. After executing the `DepositAsset` command from the above, the NFT will be minted inside the *Staging Area*.
2. The second step is the same as with Chain A when receiving an NFT.

#### Non-reserve Chain B → Reserve Chain

##### On Chain B

1. The first step is the same as with Chain A when cross-transferring an NFT.
2. The second step is the same as with Chain A by its semantics (we are cross-transferring the NFT), but the XCM program is different because we are sending the NFT back to the reserve chain.
    ```rust
    // The `WithdrawAsset` will check if the NFT is located in the Staging Area.
    // And it will also check if the `instance_hash` is correct.
    //
    // Since Chain B is NOT the reserve chain, this command will burn the derivative NFT.
    // (See the `TransactAsset` trait  implementation in the pallet's pseudocode).
    WithdrawAsset(vec![
        MultiAsset {
            id: FEE_ASSET_ID,
            fun: FEE_AMOUNT,
        },
        MultiAsset {
            id: NFT_ASSET_ID,
            fun: Fungibility::NonFungible(AssetInstance::Array32(instance_hash)),
        },
    ]),

    // Ask the reserve chain to transfer the NFT to the beneficiary's account
    InitiateReserveWithdraw {
        assets: MultiAssetFilter::Wild(All),
        reserve: MultiLocation {
            parents: 1,
            interior: X1(Parachain(RESERVE_CHAIN_PARA_ID)),
        },
        xcm: vec![
            // Buy execution on the reserve chain. 
            BuyExecution {
                fees: MultiAsset {
                    id: FEE_ASSET_ID,
                    fun: FEE_AMOUNT,
                },
                weight_limit: WeightLimit::Unlimited,
            },
            // Deposit the NFT (alongside of the remaining fungible tokens).
            //
            // The `DepositAsset` will set the NFT's owner to the `BENEFICIARY_ACCOUNT`
            // inside the Staging Area of the reserve chain. 
            DepositAsset {
                assets: MultiAssetFilter::Wild(All),
                beneficiary: MultiLocation {
                    parents: 0,
                    interior: X1(AccountId32 {
                        network: None,
                        id: BENEFICIARY_ACCOUNT,
                    }),
                },
            },
        ].into(),
    }
    ```

##### On the reserve chain

1. After executing the `DepositAsset` command from the above, the NFT will be granted to the *Beneficiary* inside the *Staging Area*.

2. The owner of the NFT can do one of the two following actions:
    * *Return* the NFT into the regular NFT logic of the reserve chain: the owner just needs to call the `return_nft` extrinsic with the only parameter `xc_asset_id`: the XCM `AssetId` of the NFT. 
    * Cross-transfer the NFT to another chain. This step is exactly the same as in the "Reserve Chain → Non-reserve Chain A" section.

### The Staging Area pallet's pseudocode

```rust
// The pseudocode of the Staging Area pallet.

// ...snip... ...imports...

/// This enum is part of the standard.
#[derive(Encode, Decode)]
enum HashingAlgorithm {
    Blake2_256,
}


/// These lengths are defined as part of the standard.
pub const HASHING_ALGORITHM_ID_LEN: u32 = 1;
pub const STANDARD_NAME_MAX_LEN: u32 = 32;
pub const VERSION_MAX_LEN: u32 = 8;

/// The metadata blob (encoded metadata container) size.
/// It is implementation-specific: max payload size could vary on different chains.
pub struct MetadataBlobMaxSize<T: Config>(PhantomData(T));
impl<T: Config> Get<u32> for MetadataBlobMaxSize<T> {
    fn get() -> u32 {
        HASHING_ALGORITHM_ID_LEN
        + STANDARD_NAME_MAX_LEN
        + VERSION_MAX_LEN
        + T::MaxPayloadSize::get()
    }
}

pub type MetadataBlob<T> = BoundedVec<u8, MetadataBlobMaxSize<T>>;

#[derive(Encode, Decode)]
pub struct MetadataStandard {
    standard_name: BoundedVec<u8, ConstU32<STANDARD_NAME_MAX_LEN>>,
    version: BoundedVec<u8, ConstU32<VERSION_MAX_LEN>>,
}

/// This struct is part of the standard.
#[derive(Encode, Decode)]
struct MetadataContainer<MaxPayloadSize: Get<u32>> {
    hashing_algorithm_id: HashingAlgorithm,
    standard: MetadataStandard,
    payload: frame_support::BoundedVec<u8, MaxPayloadSize>,
}

/// Needed for `nonfungibles_v2::Mutate::mint_into`.
/// The `mint_into` fn should interpret the `metadata`
/// and save the `asset_id` and the instance hash stored in the `multiasset` field.
struct ItemConfig<MaxPayloadSize: Get<u32>> {
    metadata: MetadataContainer<MaxPayloadSize>,
    multiasset: MultiAsset,
}

/// Helpers to retrieve the CollectionId/ItemId types from `nonfungibles_v2::Inspect`. 
type CollectionId<T> = <<T as Config>::NftPallet as nonfungibles_v2::Inspect<T::AccountId>>::CollectionId;
type ItemId<T> = <<T as Config>::NftPallet as nonfungibles_v2::Inspect<T::AccountId>>::ItemId;

/// A trait for retrieving the instance hash of an NFT's metadata container.
pub trait InstanceHashProvider<T: Config> {
    /// Retrieves (computes or returns previously saved) the instance hash of the item's metadata container.
    fn request_instance_hash(
        collection_id: &CollectionId<T>,
        item_id: &ItemId<T>,
    ) -> Result<AssetInstance, DispatchError>
};

pub trait WeightInfo<T: Config> {
    /// This function computes the weight of the `prepare_for_xc_transfer` extrinsic. 
    fn prepare_for_xc_transfer() -> Weight;

    /// This function computes the weight of the `check_in` extrinsic. 
    /// It decodes the metadata container from the `metadata_blob` and analyze how much weight
    /// its checking could take.
    fn check_in(metadata_blob: &MetadataBlob<T::MaxPayloadSize>) -> Weight;
    
    /// This function computes the weight of the `return_nft` extrinsic. 
    fn return_nft() -> Weight
};

pub trait XcNftAssetId<T: Config> {
    /// Maps the `collection_id` and the `item_id` to the cross-chain `AssetId`.
    fn cross_chain_asset_id(
        collection_id: &CollectionId<T>,
        item_id: &ItemId<T>,
    ) -> Option<AssetId>
};

#[derive(Encode, Decode)]
pub enum StagingStatus<CollectionId, ItemId> {
    /// The NFT is freshly deposited (most likely, it is from another chain).
    /// Therefore, we must check-in its metadata.
    Deposited,

    /// The NFT is already known, there is no need for metadata checking.
    Known(CollectionId, ItemId),
}

/// The information of an NFT located inside the Staging Area.
#[derive(Encode, Decode)]
pub struct StagingNftInfo<AccountId, CollectionId, ItemId> {
    /// Represents the NFT state -- this is the hash of its metadata container.
    instance_hash: AssetInstance,

    /// The owner of the NFT.
    owner: AccountId,

    /// Is it `Known` or `Deposited`?
    status: StagingStatus<CollectionId, ItemId>,
}

#[frame_support::pallet]
pub mod pallet {
    // ...snip...

    #[pallet::config]
    pub trait Config: frame_system::Config {
        /// The maximum size of metadata container's payload.
        type MaxPayloadSize: Get<u32>;

        /// The pallet that implements the NFT logic on this particular chain.
        type NftPallet: nonfungibles_v2::Mutate<T::AccountId, ItemConfig<Self::MaxPayloadSize>>
                        + nonfungibles_v2::Transfer<T::AccountId>
                        + InstanceHashProvider;

        /// Mapping: XCM AssetId -> (CollectionId, ItemId)
        type NftMatcher: MatchesNonFungibles<CollectionId<Self>, ItemId<Self>>;

        /// Mapping: (CollectionId, ItemId) -> XCM AssetId
        type XcNftAssetId: XcNftAssetId<Self>;

        /// Needed to implement `TransactAsset`: converts MultiLocation to AccountId.
        type AccountIdConverter: Convert<MultiLocation, T::AccountId>;

        // Is the NFT native to this particular chain?
        type IsNative: Contains<AssetId>;

        type WeightInfo: WeightInfo<Self>
    };

    /// XCM AssetId -> StagingNftInfo.
    #[pallet::storage]
    pub type StagingNfts<T: Config> =
        StorageMap<
            _,
            Blake2_128Concat,
            AssetId,
            StagingNftInfo<T::AccountId, CollectionId<Self>, ItemId<Self>>,
            OptionQuery
        >;

    #[pallet::error]
    pub enum Error<T> {
        UnexpectedNftMetadata,
        UnexpectedNftReturn,
        InvalidMetadata,
        NoPermission,
        UnknownItem,
    }
    

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        NftPreparedForXcTransfer {
            xc_asset_id: AssetId,
            local_id: (CollectionId<T>, ItemId<T>),
            instance_hash: AssetInstance,
        },
        NftReturned {
            xc_asset_id: AssetId,
            local_id: (CollectionId<T>, ItemId<T>),
        },
        NftWithdrawn {
            xc_asset_id: AssetId,
            instance_hash: AssetInstance,
        },
        NftDeposited {
            xc_asset_id: AssetId,
            instance_hash: AssetInstance,
            beneficiary: T::AccountId,
        },
        NftTransferred {
            xc_asset_id: AssetId,
            instance_hash: AssetInstance,
            beneficiary: T::AccountId,
        },
        NftAccepted {
            xc_asset_id: AssetId,
            local_id: (CollectionId<T>, ItemId<T>),
        },
    }
    

    #[pallet::call]
    impl<T: Config> Pallet<T> {
        /// Move the NFT into the Staging Area and compute the instance hash.
        #[pallet::call_index(0)]
        #[pallet::weight(T::WeightInfo::prepare_for_xc_transfer())]
        pub fn prepare_for_xc_transfer(
            origin: OriginFor<T>,
            collection_id: CollectionId<T>,
            item_id: ItemId<T>,
        ) -> DispatchResult {
            let who = ensure_signed!(origin)?;

            let owner = <T::NftPallet as nonfungibles_v2::Inspect<T::AccountId>>::owner(&collection_id, &item_id)
                .ok_or(<Error<T>>::UnknownItem)?;

            ensure!(who == owner, <Error<T>>::NoPermission);

            // Map the local id `(collection_id, item_id)` to the XCM `AssetId`.
            //
            // Note: if it is happening on the reserve chain,
            // the AssetId should reflect the local id closely.
            //
            // Examples:
            //  * Unique Network XCM AssetId: (Parent, Parachain(2037), GeneralIndex(<collection-id>), GeneralIndex(<item-id>)).
            //  * Moonbeam ERC-721 XCM AssetId: (Parent, Parachain(2004), AccountId20(<contract address>), GeneralIndex(<item-id>)).
            let xc_asset_id = T::XcNftAssetId::cross_chain_asset_id(&collection_id, &item_id)
                .ok_or(<Error<T>>::UnknownItem)?;

            // Compute the instance hash
            // or get previously saved instance hash provided by the originating chain.
            let instance_hash = <T::NftPallet as InstanceHashProvider>::request_instance_hash(
                &collection_id,
                &item_id,
            )?,

            // The NFT is now controlled by the Staging Area.
            T::NftPallet as nonfungibles_v2::Transfer<T::AccountId>>::transfer(
                &collection_id,
                &item_id,
                Self::staging_account(),
            )?;

            <StagingNfts<T>>::set(&xc_asset_id, Some(StagingNftInfo {
                instance_hash: instance_hash.clone(),

                // the owner can XC-transfer that NFT to another chain
                // or they can return that NFT back into the `NftPallet` if they want to.
                owner,

                // there is no need to check the metadata
                // if we just want to bring that NFT back into the `NftPallet`
                // after we prepared it for XC-transfer
                status: StagingStatus::Known(collection_id, item_id),
            }));

            Self::deposit_event(<Event<T>>::NftPreparedForXcTransfer {
                xc_asset_id,
                local_id: (collection_id, item_id),
                instance_hash,
            });

            Ok(())
        }
        

        /// The Clearance Procedure extrinsic
        #[pallet::call_index(1)]
        #[pallet::weight(T::WeightInfo::check_in(&metadata_blob))]
        pub fn check_in(
            origin: OriginFor<T>,
            xc_asset_id: AssetId,
            metadata_blob: MetadataBlob<T>,
        ) -> DispatchResult {
            let nft_info = <StagingNfts<T>>::get(&xc_asset_id)
                .ok_or(<Error<T>>::UnknownItem)?;

            let who = ensure_signed!(origin)?;

            // Only the owner can `check-in` the NFT.
            // Because the owner could want to transfer the NFT
            // to another chain right after receiving it on this chain.
            ensure!(who == nft_info.owner, <Error<T>>::NoPermission);

            // Do not "check-in" the known NFT. It would be a waste of computational resources.
            ensure!(
                matches![nft_info.status, StagingStatus::Deposited],
                <Error<T>>::UnexpectedNftMetadata
            );

            let mut input = metadata_blob.as_slice();
            let metadata: MetadataContainer = metadata_blob.decode()
                .map_err(|_| <Error<T>>::InvalidMetadata)?;

            let nft_multiasset = MultiAsset {
                asset_id: xc_asset_id.clone(),
                fun: Fungibility::NonFungible(nft_info.instance_hash.clone()),
            };

            // Try map the `MultiAsset` NFT representation to the native one.
            // If no such mapping could be established -- return an error.
            let (collection_id, item_id) = T::NftMatcher::matches_nonfungibles(&nft_multiasset)?;

            todo!("Compute the hash of the `metadata_blob` according to the `metadata.hashing_algorithm_id`");

            todo!(
                "Compare the computed hash against the `nft_info.instance_hash`.
                If they are not equal, return the `InvalidMetadata` error"
            );

            // Mint the NFT to the beneficiary.
            // The `ItemConfig` contains the metadata container and the NFT MultiAsset
            // (contaning the XCM AssetId and the instance hash).
            //
            // The `NftPallet` is responsible for saving metadata container in the native representation. 
            // Also, it should remember the XCM AssetId and the instance hash to facilitate the `XcNftAssetId` mapping implementation.
            //
            // It also could do additional checks if needed.
            <T::NftPallet as nonfungibles_v2::Mutate<T::AccountId, ItemConfig<T::MaxPayloadSize>>>::mint_into(
                &collection_id,
                &item_id,
                &nft_info.owner,
                &ItemConfig {
                    metadata,
                    multiasset: nft_multiasset,
                },
                false, // `deposit_collection_owner`.
            )?;

            <StagingNfts<T>>::remove(&xc_asset_id);

            Self::deposit_event(<Event<T>>::NftAccepted {
                xc_asset_id,
                local_id: (collection_id, item_id),
            });

            Ok(())
        }
        

        /// If the NFT owner changes their mind after the `prepare_for_xc_transfer` call
        /// and no more wants to XC-transfer the NFT to another chain,
        /// they can return the NFT into the NFT pallet without the "check-in" procedure.
        ///
        /// Also, this operation can be used to return an NFT into the Reserve Chain's NFT pallet.
        #[pallet::call_index(2)]
        #[pallet::weight(T::WeightInfo::return_nft())]
        pub fn return_nft(
            origin: OriginFor<T>,
            xc_asset_id: AssetId,
        ) -> DispatchResult {
            let who = ensure_signed!(origin)?;

            let nft_info = <StagingNfts<T>>::get(&xc_asset_id)
                .ok_or(<Error<T>>::UnknownItem)?;

            ensure!(who == nft_info.owner, <Error<T>>::NoPermission);

            /// The NFT metadata must be known.
            let (collection_id, item_id) = match nft_info.status {
                StagingStatus::Known(col_id, item_id) => (col_id, item_id),
                _ => return Err(<Error<T>>::UnexpectedNftReturn),
            };

            <T::NftPallet as nonfungibles_v2::Transfer<T::AccountId>>::transfer(
                &collection_id,
                &item_id,
                who,
            )?;

            <StagingNfts<T>>::remove(&xc_asset_id);

            Self::deposit_event(<Event<T>>::NftReturned {
                xc_asset_id,
                local_id: (collection_id, item_id),
            });

            Ok(())
        }
    }
}

impl<T: Config> Pallet<T> {
    /// The account of the Staging Area pallet.
    pub fn staging_account() -> T::AccountId {
        const ID: PalletId = PalletId(*b"nft/stgng");
        AccountIdConversion::<T::AccountId>::into_account_truncating(&ID)
    }
}

/// The `TransactAsset` implementation.
/// This trait is used by the XCM executor to execute several XCM commands
/// such as: `WithdrawAsset`, `TransferReserveAsset`, `DepositAsset`.
impl<T: Config> TransactAsset for Pallet<T> {
    /// `DepositAsset` command implementation.
    fn deposit_asset(what: &MultiAsset, who: &MultiLocation, _context: &XcmContext) -> XcmResult {
        let who = AccountIdConverter::convert_ref(who).map_err(|()| xcm_builder::Error::AccountIdConversionFailed)?;

        match what {
            MultiAsset {
                asset_id,
                fun: Fungibility::NonFungible(instance_hash),
            } => {
                if T::IsNative::contains(&asset_id) {
                    // on a reserve chain
                    <StagingNfts<T>>::try_mutate(&asset_id, |nft_info| {
                        let nft_info.as_mut().ok_or(XcmError::::AssetNotFound)?;

                        // Only the owner of the NFT is changed.
                        // The `status` id NOT changed, so we can simply `return`
                        // the NFT on the Reserve Chain if we want to.
                        nft_info.owner = who.clone();
                        
                        Ok(())
                    })?
                } else {
                    // on a non-reserve chain
                    <StagingNfts<T>>::set(&asset_id, Some(StagingNftInfo {
                        instance_hash: instance_hash.clone(),
                        
                        // the beneficiary can "check-in" the NFT or tranfer it to another chain.
                        owner: who.clone(),
                        
                        // The NFT is not Known, so we must "check-in" the NFT's metadata.
                        status: StagingStatus::Deposited,
                    }))
                };
                

                Self::deposit_event(<Event<T>>::NftDeposited {
                    xc_asset_id: asset_id,
                    instance_hash,
                    beneficiary: who,
                });

                Ok(())
            },
            _ => Err(XcmError::AssetNotFound),
        }
    }
        
    

    /// `WithdrawAsset` command implementation.
    fn withdraw_asset(
        what: &MultiAsset,
        who: &MultiLocation,
        _maybe_context: Option<&XcmContext>,
    ) -> Result<xcm_executor::Assets, XcmError> {
        let who = AccountIdConverter::convert_ref(who).map_err(|()| xcm_builder::Error::AccountIdConversionFailed)?;

        let (asset_id, instance_hash) = match what {
            MultiAsset {
                asset_id,
                fun: Fungibility::NonFungible(instance_hash),
            } => (asset_id, instance_hash),
            _ => return Err(XcmError::AssetNotFound)
        };

        let nft_info = <StagingNfts<T>>::get(&asset_id)
            .ok_or(XcmError::::AssetNotFound)?;

        // Only the owner can withdraw the NFT.
        ensure!(who == nft_info.owner, XcmError::::FailedToTransactAsset("not an owner"));

        // The instance hash is a part of the NFT MultiAsset.
        // It must be the same as the hash computed during the preparation for XC-transfer.
        ensure!(instance_hash == nft_info.instance_hash, XcmError::::FailedToTransactAsset("invalid instance hash"));

        if T::IsNative::contains(&asset_id) {
            // on a reserve chain we shouldn't burn the NFT.
            // The subsequent Transfer/Deposit command will set the `StagingNfts` storage into a proper state.
            // Nothing to do here.
        } else {
            // on a non-reserve chain: burn the derivative NFT.

            let (collection_id, item_id) = T::NftMatcher::matches_nonfungibles(&what)?;

            let maybe_check_owner = None;
            <T::NftPallet as nonfungibles_v2::Mutate<T::AccountId, ItemConfig<T::MaxPayloadSize>>>::burn(&collection_id, &item_id, maybe_check_owner)?;

            <StagingNfts<T>>::remove(&asset_id);
        }
        

        Self::deposit_event(<Event<T>>::NftWithdrawn {
            xc_asset_id: asset_id,
            instance_hash,
        });

        Ok(asset_id)
    }
    

    /// `TransferAsset` / `TransferReserveAsset` command implementation.
    fn transfer_asset(
        what: &MultiAsset,
        from: &MultiLocation,
        to: &MultiLocation,
        context: &XcmContext,
    ) -> Result<xcm_executor::Assets, XcmError> {
        let from = AccountIdConverter::convert_ref(from).map_err(|()| xcm_builder::Error::AccountIdConversionFailed)?;
        let to = AccountIdConverter::convert_ref(to).map_err(|()| xcm_builder::Error::AccountIdConversionFailed)?;

        let (asset_id, instance_hash) = match what {
            MultiAsset {
                asset_id,
                fun: Fungibility::NonFungible(instance_hash),
            } => (asset_id, instance_hash),
            _ => return Err(XcmError::AssetNotFound)
        };

        <StagingNfts<T>>::try_mutate(&asset_id, |nft_info| {
            let nft_info.as_mut().ok_or(XcmError::::AssetNotFound)?; 

            nft_info.owner = to.clone();

            Ok(())
        })?;

        Self::deposit_event(<Event<T>>::NftTransferred {
            xc_asset_id: asset_id,
            instance_hash,
            beneficiary: to,
        });

        Ok(())
    }
}    
```
