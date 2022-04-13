---
date: 2022-04-13 00:00:00 +01
title: Valorem Options Litepaper
usemathjax: true
description: This litepaper introduces Valorem Options V1, an oracle-free, permissionless, physically settled options protocol for ERC20 tokens. 
---

## Introduction

Options are a primitive essential to the functioning of any financial system. 
In traditional finance, options volume exceeds spot volume. In the world of 
digital assets, spot volumes still exceed options trading volumes. Options 
trading volumes on assets like BTC and ETH have grown significantly in the 
past year, both on centralized exchanges such as Deribit, and on-chain 
protocols.

There are already a number of on-chain options protocols. While most of these 
emulate traditional options structures, the reliance on price oracles and 
assumptions around options premiums via models like black-scholes make them 
inflexible and subject to adverse-selection. There are some recent protocols 
which emulate options via single tick Uniswap V3 LPs. These are restricted by 
the lack of Uniswap V3 deployment across evm chains, gas inefficiency 
of Uniswap V3 LP NFTs, and the nature of perpetual options (as 
opposed to ones with fixed exercise and expiry timestamps). Present on chain 
protocols are fundamentally limited by these design choices.

This litepaper introduces the Valorem Options V1 protocol. Valorem 
is a DeFi native options protocol which aims to provide superior flexibility 
over existing options protocols by removing price oracles, reliance on existing 
defi primitives, and premium value assumptions. Valorem achieves this by 
implementing an options settlement protocol driven by market forces and settled 
in a provably fair and random ordering using VRF. The Valorem Options V1 
protocol consists of a set of smart contracts which can interact directly with 
any non-rebasing, non-fee on transfer, ERC20 token pair to handle the minting 
and exercise settlement of any put or call options in a permissionless and 
trustless manner.

### Key innovations

#### Permissionless

The Valorem protocol is designed to be permissionless. This permissionless
design means that the protocol is open to public use, with no ability to restrict
who can or cannot use it. Anyone can create new options chains, write options,
exercise the options they hold, transfer those options, and transfer claims
generated during writing.

#### Fully collateralized

The protocol being fully collateralized means that options can be exercised at
any time prior to expiry and after the exercise timestamp, without the risk of
default. This design choice was made due to the nature of decentralization and
the permissionless nature of the protocol.

#### Composibility

The Valorem protocol allows writers to write options on any ERC-20 pair, with
any strike price, any expiry timestamp at least 24 hours in the future, and
any exercise timestamp at least 24 hours before expiry. This flexibility enables
the settlement layer to be used in a large range of applications. Options 
contracts are issued as fungible ERC-1155 tokens, herein referred to as
vTokens, with each token representing a contract. Option writers are
additionally issued an ERC-1155 NFT representing a lot of contracts written
for claiming collateral and exercise assignment. These tokens allow for novel 
structured products to be created atop the protocol.

#### Oracle-lite

Part of what enables the flexibility for the protocol, is the absence of a need
for price oracles. Option exercise happens at the discretion of the option
holder. The only oracles in the protocol design are VRF for provably random and
fair exercise assignment.

## Use cases

### Call options

The Valorem protocol can be used to write covered call options with the payoff 
$$\[\max(0,-S_T)\]$$ for the buyer.
