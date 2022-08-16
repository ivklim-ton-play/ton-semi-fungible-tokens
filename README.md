# SFT: semi-fungible token
A standard interface for semi-fungible tokens(SFT). 

# Motivation
The idea is simple and seeks to create standard that can represent and control any number of fungible and non-fungible token types.

- The way of ownership changing.
- The way of association of similar items into collections.
- The way of tonkens transfers.
- The way of retrieving common information (name, circulating supply, etc) about given semi-fungible tokens asset.

# Specification

Here and following we use:
 - "SFT" - semi-fungible token. Almost the same as jetton from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74) standart. However, the decimal number is always 0. It leads to the logic that each token is undivided but fungible.
 - "SFT wallet" - wallet for semi-fungible tokens. Stores information about amount of SFTs owned by each user. Almost the same as jetton-wallets from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). Smart-contracts called **"[sft-wallet](#sft-wallet)"**.
 - "SFT minter" - minter of semi-fungible tokens. It stores one [token data](https://github.com/ton-blockchain/TIPs/issues/64) for all SFTs that it minted. Ideologically very similar to Jetton master from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). Smart-contracts called **"sft-minter"**.
 - "SFT collection" - collection for SFT minters. Each SFT minter has its own unique id. Based on the idea of [nft-collection](https://github.com/ton-blockchain/TIPs/issues/62). Smart-contracts called **"sft-collection"**.

### Example: 
You release a SFT-collection with circulating supply of 200 SFTs for id = 0, and circulating supply of 100 SFTs for id = 1.
- We have 2 owners.
  - Owner_1 has 100 SFTs by id = 0 and 100 by id = 1.
  - Owner_2 has 100 SFTs by id = 0.

We need deploy 6 contracts: 
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

`max_tx_gas_price` is the price in Toncoins of maximum transaction `gas limit` of NFT habitat workchain. For the basechain it can be obtained from [`ConfigParam 21`](https://github.com/ton-blockchain/ton/blob/78e72d3ef8f31706f30debaf97b0d9a2dfa35475/crypto/block/block.tlb#L660) from `gas_limit` field.

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
