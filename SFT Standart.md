- **TEP**: 
- **title**: SFT (semi-fungible token) Standard
- **status**: Draft
- **type**: Contract Interface
- **authors**: [Ivan Klimov](https://github.com/ivklim-ton-play) 
- **created**: 03.09.2022

# Summary

A standard interface for semi-fungible tokens(SFT). 

# Motivation
The idea is simple and seeks to create standard that can represent and control any number of fungible and non-fungible token types.

- The way of ownership changing.
- The way of association of similar items into collections.
- The way of tonkens transfers.
- The way of retrieving common information (name, circulating supply, etc) about given semi-fungible tokens asset.

# Guide

This standard is needed in the case when you need to issue a large number of identical NFTs, but the serial number is not important. For example, a membership card to access the site or the same items for games (100 red t-shirts, 100 yellow shorts, etc.). 

SFT represents ownership of a copy of a unique asset in a collection.

## SFT Metadata
Each SFT minter and SFT Collection itself has its own metadata (TEP-64). It contains some info about SFT, such as title and associated image. Metadata can be stored offchain (smart contract will contain only a link to json) or onchain (all data will be stored in smart contract).

[SFT minter metadata](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#nft-collection-metadata-example-offchain) (it's like NFT item metadata)

[SFT collection metadata](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md#nft-item-metadata-example-offchain) (it's like NFT collection metadata)

Offchain metadata is published for example on web 

## Useful links
1. [Reference semi-fundable token implementation](https://github.com/ivklim-ton-play/ton-SFT)

# Specification

Here and following we use:
 - "SFT" - semi-fungible token. Almost the same as jetton from [Jetton](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md) standart. However, the decimal number is always 0 (we only need [token metadata](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md) for NFT). It leads to the logic that each token is undivided but fungible.
 - "SFT wallet" - wallet for semi-fungible tokens. Stores information about amount of SFTs owned by each user. Almost the same as jetton-wallets from [Jetton](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md). Smart-contracts called **"[sft-wallet](#sft-wallet)"**.
 - "SFT minter" - minter of semi-fungible tokens. It stores one NFT [token metadata](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md) for all SFTs that it minted. Ideologically very similar to Jetton master from [Jetton](https://github.com/ton-blockchain/TEPs/blob/master/text/0074-jettons-standard.md). Smart-contracts called **"sft-minter"**.
 - "SFT collection" - collection for SFT minters. Each SFT minter has its own unique id. Based on the idea of [nft-collection](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md). Smart-contracts called **"sft-collection"**.

### Example: 
You release a SFT-collection with circulating supply of 200 SFTs for id = 0, and circulating supply of 100 SFTs for id = 1.
- We have 2 owners.
  - Owner_1 has 100 SFTs by id = 0 and 100 by id = 1.
  - Owner_2 has 100 SFTs by id = 0.

We need to deploy 6 contracts: 
- **1** sft-collection smart-contract.
- **1** sft-minter smart-contract by id = 0 and **1** sft-minter smart-contract by id = 1.
- For owner_1 we need **1** sft-wallet smart-contract from sft-minter by id = 0 and **1** sft-wallet smart-contract from sft-minter by id = 1.
- For owner_2 we need **1** sft-wallet smart-contract from sft-minter by id = 0.

<a name="sft-wallet"></a>
## SFT wallet smart contract
Must implement:

### Internal message handlers

1. `transfer` 

**Request**

TL-B schema of inbound message:

```
transfer#0f8a7ea5 query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress
                 response_destination:MsgAddress custom_payload:(Maybe ^Cell)
                 forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell)
                 = InternalMsgBody;
```

`query_id` - arbitrary request number.

`amount` - amount of transferred SFTs in elementary units.

`destination` - address of the new owner of the SFTs.

`response_destination` - address where to send a response with confirmation of a successful transfer and the rest of the incoming message Toncoins.

`custom_payload` - optional custom data (which is used by either sender or receiver SFT wallet for inner logic).

`forward_ton_amount` - the amount of nanotons to be sent to the destination address.

`forward_payload` - optional custom data that should be sent to the destination address.

**Should be rejected if:**

 1. message is not from the owner.
 2. there is no enough SFTs on the sender wallet
 3. there is no enough TON (with respect to SFT own storage fee guidelines and operation costs) to process operation, deploy receiver's sft-wallet and send `forward_ton_amount`.
 4. After processing the request, the receiver's sft-wallet **must** send at least `in_msg_value - forward_amount - 2 * max_tx_gas_price` to the `response_destination` address.

If the sender sft-wallet cannot guarantee this, it must immediately stop executing the request and throw error.

`max_tx_gas_price` is the price in Toncoins of maximum transaction `gas limit` of SFT habitat workchain. For the basechain it can be obtained from [`ConfigParam 21`](https://github.com/ton-blockchain/ton/blob/78e72d3ef8f31706f30debaf97b0d9a2dfa35475/crypto/block/block.tlb#L660) from `gas_limit` field.

**Otherwise should do:**

 1. Decrease SFTs amount on sender wallet by `amount` and send message which increase SFTs amount on receiver wallet (and optionally deploy it).
 2. if `forward_amount > 0` ensure that receiver's sft-wallet send message to `destination` address with `forward_amount` nanotons attached and with the following layout:

TL-B schema:

```
transfer_notification#7362d09c query_id:uint64 amount:(VarUInteger 16) 
                              sender:MsgAddress forward_payload:(Either Cell ^Cell)
                              = InternalMsgBody;
```

```
`query_id` should be equal with request's `query_id`.

`amount` amount of transferred SFTs.

`sender` is address of the previous owner of transferred SFTs.

`forward_payload` should be equal with request's `forward_payload`.

If `forward_amount` is equal to zero, notification message should not be sent.
```

 3. Receiver's wallet should send all excesses of incoming message coins to `response_destination` with the following layout:

TL-B schema: `excesses#d53276db query_id:uint64 = InternalMsgBody;`

`query_id` should be equal with request's `query_id`.

2. **`burn`**

**Request**

TL-B schema of inbound message:

```
burn#595f07bc query_id:uint64 amount:(VarUInteger 16) 
              response_destination:MsgAddress custom_payload:(Maybe ^Cell)
              = InternalMsgBody;
```

`query_id` - arbitrary request number.

`amount` - amount of burned SFTs.

`response_destination` - address where to send a response with confirmation of a successful burn and the rest of the incoming message coins.

`custom_payload` - optional custom data.

**Should be rejected if:**

 1. Message is not from the owner.

 2. There is no enough SFTs on the sender wallet.

 3. There is no enough TONs to send after processing the request at least `in_msg_value - max_tx_gas_price` to the `response_destination` address.

 If the sender sft-wallet cannot guarantee this, it **must** immediately stop executing the request and throw error.

**Otherwise should do:**

 1. decrease SFTs amount on burner wallet by `amount` and send notification to **SFT minter** with information about burn.

 2. **SFT minter** should send all excesses of incoming message coins to `response_destination` with the following layout:

TL-B schema: `excesses#d53276db query_id:uint64 = InternalMsgBody;`

`query_id` should be equal with request's `query_id`.

### Get-methods
`get_sft_wallet_data()` returns `(int balance, slice owner, slice sft_minter, cell sft_wallet_code)`

`balance` - (uint256) amount of SFTs on wallet.  
`owner` - (MsgAddress) address of wallet owner.  
`sft_minter` - (MsgAddress) address of SFT minter.  
`sft_wallet_code` - (cell) with code of this wallet.  

## SFT minter

Must implement:

**Internal message handlers**

#### 1. `get_static_data`

**Request**

TL-B schema of inbound message:

`get_static_data#2fcb26a2 query_id:uint64 = InternalMsgBody;`

`query_id` - arbitrary request number.

**should do:**

Send back message with the following layout and send-mode `64` (return msg amount except gas fees):

TL-B schema: `report_static_data#8b771735 query_id:uint64 index:uint256 collection:MsgAddress = InternalMsgBody;`

`query_id` should be equal with request's `query_id.`

`index` - numerical index of this SFT minter in the collection, usually serial number of deployment.

`collection` - address of the smart contract of the collection to which this SFT minter belongs.

 1. `get_sft_data()` returns `(int init?, int index, slice collection_address, int total_supply, slice admin_address, cell individual_sft_content, cell sft_wallet_code)`

 `init?` - if not zero, then this SFT minter is fully initialized and ready for interaction

 `index` - (integer) - index in SFT collection
 
 `collection_address` - (MsgAddress) - address of the smart contract of the collection to which this SFT minter belongs. For collection-less SFT minter this parameter should be addr_none;
 
 `total_supply` - (integer) - the total number of issues SFTs  

 `admin_address` - (MsgAddress) - address of smart-contrac which control SFT minter

`individual_sft_content` - (cell) - if SFT minter has collection - individual SFT content in any format;
if SFT minter has no collection - SFT minter content in format that complies with [Token Data Standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md) #64

`sft_wallet_code` - (cell) - code of wallet for that SFTs

 2. `get_wallet_address(slice owner_address)` return `slice sft_wallet_address`

 Returns SFT wallet address (MsgAddress) for this owner address (MsgAddress).
 
 # SFT Collection smart contract
 
It is assumed that the smart contract of the collection deploys smart contracts of SFT minters of this collection.

Must implement:

### Get-methods
1. `get_sft_collection_data()` returns `(int next_sft_minter_index, cell collection_content, slice owner_address)`

`next_sft_minter_index`- (int) the count of currently deployed SFT minter items in collection. Generally, collection should issue NFT with sequential indexes (see [Rationale(2)](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md#rationale-and-alternatives) in NFT ). -1 value of next_item_index is used to indicate non-sequential collections, such collections should provide their own way for index generation / item enumeration.

`collection_content` - (cell) - collection content in a format that complies with NFT standard [TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md).

`owner_address` - (MsgAddress) - collection owner address, zero address if no owner.

2. `get_sft_minter_address_by_index(int index)` returns `slice address`

Gets the serial number of the SFT minter of this collection and returns the address (MsgAddress) of this SFT minter smart contract.

3. `get_sft_minter_content(int index, cell individual_content)` returns `cell full_content`
Gets the serial number of the SFT minter item of this collection and the individual content of this SFT minter item and returns the full content of the SFT minter item in format that complies with standard [TEP-64](https://github.com/ton-blockchain/TEPs/blob/master/text/0064-token-data-standard.md).

As an example, if an SFT minter item stores a metadata URI in its content, then a collection smart contract can store a domain (e.g. "[https://site.org/](https://site.org/)"), and an SFT minter item smart contract in its content will store only the individual part of the link (e.g "kind-cobra").

In this example the `get_nft_content` method concatenates them and return "[https://site.org/kind-cobra](https://site.org/kind-cobra)".

# Drawbacks
There is no way to get current owner of SFT collection and SFT minter onchain because TON is an asynchronous blockchain. When the message with info about SFT owner will be delivered, this info may become irrelevant, so we can't guarantee that current owner hasn't changed.

There is no way to get actual wallet balance onchain, because when the message with balance will arrive, wallet balance may be not actual.

# Rationale 
Look in [NFT Rationale and alternatives](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md#rationale-and-alternatives)

# Prior art
1. [Ethereum NFT Standard (EIP-1155)](https://eips.ethereum.org/EIPS/eip-1155)

# Unresolved questions

# Future possibilities
none

# TL-B schema
```
nothing$0 {X:Type} = Maybe X;
just$1 {X:Type} value:X = Maybe X;
left$0 {X:Type} {Y:Type} value:X = Either X Y;
right$1 {X:Type} {Y:Type} value:Y = Either X Y;
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8))
         = VarUInteger n;

addr_none$00 = MsgAddressExt;
addr_extern$01 len:(## 9) external_address:(bits len) 
             = MsgAddressExt;
anycast_info$_ depth:(#<= 30) { depth >= 1 }
   rewrite_pfx:(bits depth) = Anycast;
addr_std$10 anycast:(Maybe Anycast) 
   workchain_id:int8 address:bits256  = MsgAddressInt;
addr_var$11 anycast:(Maybe Anycast) addr_len:(## 9) 
   workchain_id:int32 address:(bits addr_len) = MsgAddressInt;
_ _:MsgAddressInt = MsgAddress;
_ _:MsgAddressExt = MsgAddress;

transfer query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress
           response_destination:MsgAddress custom_payload:(Maybe ^Cell)
           forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell)
           = InternalMsgBody;
           
transfer_notification query_id:uint64 amount:(VarUInteger 16)
           sender:MsgAddress forward_payload:(Either Cell ^Cell)
           = InternalMsgBody;

excesses query_id:uint64 = InternalMsgBody;

burn query_id:uint64 amount:(VarUInteger 16) 
       response_destination:MsgAddress custom_payload:(Maybe ^Cell)
       = InternalMsgBody;
       
get_static_data query_id:uint64 = InternalMsgBody;

report_static_data query_id:uint64 index:uint256 collection:MsgAddress = InternalMsgBody;
       
// ----- Unspecified by standard, but suggested format of internal message

internal_transfer  query_id:uint64 amount:(VarUInteger 16) from:MsgAddress
                     response_address:MsgAddress
                     forward_ton_amount:(VarUInteger 16)
                     forward_payload:(Either Cell ^Cell) 
                     = InternalMsgBody;
                     
burn_notification query_id:uint64 amount:(VarUInteger 16) 
       sender:MsgAddress response_destination:MsgAddress
       = InternalMsgBody;  
```

`crc32('transfer query_id:uint64 amount:VarUInteger 16 destination:MsgAddress response_destination:MsgAddress custom_payload:Maybe ^Cell forward_ton_amount:VarUInteger 16 forward_payload:Either Cell ^Cell = InternalMsgBody') = 0x8f8a7ea5 & 0x7fffffff = 0xf8a7ea5`

`crc32('transfer_notification query_id:uint64 amount:VarUInteger 16 sender:MsgAddress forward_payload:Either Cell ^Cell = InternalMsgBody') = 0xf362d09c & 0x7fffffff = 0x7362d09c`

`crc32('excesses query_id:uint64 = InternalMsgBody') = 0x553276db | 0x80000000 = 0xd53276db`

`crc32('burn query_id:uint64 amount:VarUInteger 16 response_destination:MsgAddress custom_payload:Maybe ^Cell = InternalMsgBody') = 0x595f07bc & 0x7fffffff = 0x595f07bc`

`crc32('internal_transfer query_id:uint64 amount:VarUInteger 16 from:MsgAddress response_address:MsgAddress forward_ton_amount:VarUInteger 16 forward_payload:Either Cell ^Cell = InternalMsgBody') = 0x978d4519 & 0x7fffffff = 0x178d4519`

`crc32('burn_notification query_id:uint64 amount:VarUInteger 16 sender:MsgAddress response_destination:MsgAddress = InternalMsgBody') = 0x7bdd97de & 0x7fffffff = 0x7bdd97de`

`crc32('get_static_data query_id:uint64 = InternalMsgBody') = 0x2fcb26a2 & 0x7fffffff = 0x2fcb26a2`

`crc32('report_static_data query_id:uint64 index:uint256 collection:MsgAddress = InternalMsgBody') = 0xb771735 | 0x80000000 = 0x8b771735`
