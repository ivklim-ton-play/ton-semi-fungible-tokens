# SFT: semi-fungible token
A standard interface for semi-fungible tokens(SFT).

# Motivation
A standard interface will greatly simplify interaction and display of different collections tokenized assets.

- The way of ownership changing.
- The way of association of items into collections.
- The way of tonkens transfers.
- The way of retrieving common information (name, circulating supply, etc) about given semi-fungible tokens asset.

# Specification

Here and following we use:
 - "SFT" - semi-fungible token. Almost the same as jetton from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74) standart. However, the decimal number is always 0. It leads to the logic that each token is undivided but fungible.
 - "SFT-wallet" - wallet for semi-fungible tokens. Stores information about amount of SFTs owned by each user. Almost the same as jetton=wallets from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). Smart-contracts called "sft-wallet".
 - "SFT-collection" - contains a collection of sfts groups. Each group has its own unique id. Based on the idea of [nft-collection](https://github.com/ton-blockchain/TIPs/issues/62) to store the same sfts under the same id. Smart-contracts called "sft-collection".
 - "SFT-group" - contains sfts under the same id from SFT-collection. Ideologically very similar to the [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). But we do not create a separate smart contract, since the "sft-collection" smart-contract will take over all the necessary functions.
