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
 - "SFT-wallet" - wallet for semi-fungible tokens. Stores information about amount of SFTs owned by each user. Almost the same as jetton-wallets from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). Smart-contracts called **"sft-wallet"**.
 - "SFT-minter" - minter of semi-fungible tokens. It stores one [token data](https://github.com/ton-blockchain/TIPs/issues/64) for all SFTs that it minted. Ideologically very similar to Jetton master from [Jetton](https://github.com/ton-blockchain/TIPs/issues/74). Smart-contracts called **"sft-minter"**.
 - "SFT-collection" - collection for SFT-minters. Each SFT-minter has its own unique id. Based on the idea of [nft-collection](https://github.com/ton-blockchain/TIPs/issues/62). Smart-contracts called **"sft-collection"**.

## Example: 
You release a SFT-collection with circulating supply of 200 SFTs for id = 0, and circulating supply of 100 SFTs for id = 1.
- We have 2 owners.
  - Owner_1 has 100 SFTs by id = 0 and 100 by id = 1.
  - Owner_2 has 100 SFTs by id = 0.

We need deploy 6 contracts: 
- **1** sft-collection smart-contract.
- **1** sft-minter smart-contract by id = 0 and **1** sft-minter smart-contract by id = 1.
- For owner_1 we need **1** sft-wallet smart-contract from sft-minter by id = 0 and **1** sft-wallet smart-contract from sft-minter by id = 1.
- For owner_2 we need **1** sft-wallet smart-contract from sft-minter by id = 0.


