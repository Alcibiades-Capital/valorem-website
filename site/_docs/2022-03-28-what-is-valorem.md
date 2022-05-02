---
date: 2022-03-28 00:00:00 +01
title: What is Valorem? 
description: Introduction to the Valorem protocol, a peer-to-peer system for option settlement.
---

## Introduction

The Valorem protocol is a peer-to-peer system designed for settling options 
on ERC-20 tokens. The protocol is implemented as a set of persistent, 
non-upgradable smart contracts; designed to prioritize censorship resistance, 
security, self-custody, and to function without any intermediaries who may 
restrict access.

The Valorem protocol enables writing covered call and 
covered put, physically settled, options. All written options are fully 
collateralized against an ERC-20 underlying asset and exercised with an ERC-20 
exercise asset using a deterministic algorithm based on the option hash for fair 
settlement.

Options contracts are issued as fungible ERC-1155 tokens, herein referred to as 
vTokens, with each token representing a contract. Option writers are 
additionally issued an ERC-1155 NFT representing a lot of contracts written 
for claiming collateral and exercise assignment. This design eliminates the 
need for market price oracles, and allows for permission-less writing, and 
gas efficient transfer, of a broad swath of traditional and exotic options.

## Design Choices

### Permissionless

The Valorem protocol is designed to be permissionless. This permissionless 
design means that the protocol is open to public use, with no ability to restrict 
who can or cannot use it. Anyone can create new options chains, write options, 
exercise the options they hold, transfer those options, and transfer claims 
generated during writing.

### Fully Collateralized

The protocol being fully collateralized means that options can be exercised at 
any time prior to expiry and after the exercise timestamp, without the risk of 
default. This design choice was made due to the nature of decentralization and 
the permissionless nature of the protocol.

### Flexible

The Valorem protocol allows writers to write options on any ERC-20 pair, with 
any strike price, any expiry timestamp at least 24 hours in the future, and 
any exercise timestamp at least 24 hours before expiry. This flexibility enables 
the settlement layer to be used in a large range of applications.

#### Oracle-lite

Part of what enables the flexibility for the protocol, is the absence of a need 
for price oracles. Option exercise happens at the discretion of the option 
holder. The only oracles in the protocol design are VRF for provably random and 
fair exercise assignment.
