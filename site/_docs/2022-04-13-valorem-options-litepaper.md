---
date: 2022-04-13 00:00:00 +01
title: Valorem Options Litepaper
usemathjax: true
description: This litepaper introduces Valorem Options V1, an oracle-free, permissionless, physically settled options protocol for ERC20 tokens. 
---

## Introduction

Options are a component, essential to the functioning of any financial system. 
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
opposed to ones with fixed exercise and expiry timestamps) present on chain 
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
additionally issued an ERC-1155 NFT representing group contracts written
for claiming collateral and exercise assignment. These tokens allow for novel 
structured products to be created atop the protocol.

#### Oracle-lite

Part of what enables the flexibility for the protocol, is the absence of a need
for price oracles. Option exercise happens at the discretion of the option
holder. The only oracles in the protocol design are VRF for provably random and
fair exercise assignment.

## Protocol Description

The Valorem protocol acts as an underwriting aide and clearinghouse for options 
which are tokenized for owership on EVM blockchains.  The protocol allows the 
following actions:

### Writing Options

Actors on chain, either individuals using their wallets or protocols using smart
contracts, can write options by depositing the necessary collateral into the 
Valorem vault. They can specify the following:

- The underlying collateral of the option, this is what the option owner receives 
  if the option is exercised.
- The exercise token of the option,  this is what the option owner pays, and 
  what the option writer receives, if the option is exercised.
- The expiration date of the option.
- The earliest exercise date of the option.
- The strike price of the option.

This information comprises a unique hash `keccak256(abi.encode(Option memory))`, 
which is then used to determine if that type of option already exists, and if 
not, create it. Once the option type is created, upon writing, the option 
writer will receive a `claim` token indicating that they have written the option.
They will also receive an `option` token, indicating that they have the ability
to exercise the option pursuant to the terms set during writing.  Both the 
`claim` token and the `option` token can be transferred or traded to other 
actors on the chain.

### Exercising Options
Owners of an `option` token can exercise the option pursuant to the following
conditions:

- The earliest exercise date has passed.
- The expiration date has not passed
- The owner of the `option` token has enough of the exercise token required
  to purchase the amount of the underlying token at the strike price.

### Redeeming Options
Owners of a `claim` token can exercise their claim to the collateral in the 
Valorem protocol vault once the expiration date of the option has passed. 
If the collateral associated with their claims token was assigned during 
exercise of the option, the appropriate amount of the underlying token will
be sent to the `claim` token holder and the `claim` token will be burned. 
There also exists the possibility of partial assignment, in which the
`claim` holder will receive some ration of the underlying and exercise 
token. If the collateral associated with the `claim` token was not 
assigned, the `claim` token holder will receive their underlying token back
instead.


### Exercise Assignment

When an option is exercised, the Valorem protocol uses a provably fair pseudo 
random number from a VRF oracle to determine which `claim` token’s 
corresponding collateral will be assigned.  This process repeatedly cycles 
through randomly selected `claim` tokens’ collateral until sufficient 
collateral has been assigned to cover the full amount owed to the option 
holder.

## Use cases

### Simple Options

Where:

$ S_T $ is the price of the underlying asset at expiration; and
$ X $ is the exercise asset price; and
$ c_0 $ is the call option premium paid; and
$ p_0 $ is the put option premium paid.

#### Call options

The Valorem protocol can be used to create covered call options with the payoff 
$ max(0,S_T-X) $ for the holder, and the payoff  $ -max(0,S_T-X) $ for the 
writer.

#### Put options

The Valorem protocol can be used to create covered put options with the payoff
$ max(0,X-S_T) $ for the holder, and the payoff  $ -max(0,X-S_T) $ for the 
writer.

### Trading and Market Making

The Valorem protocol provides web3 developers with an options base layer 
that can be seamlessly integrated into existing and future AMMs and CLOBs. 
By acting as the clearinghouse and settlement service needed for options 
execution, Valorem allows market makers to list options on without the 
need to implement their own options-specific smart contract adjustments,
risk management system, or collateral assignment process. This frees up 
MMs to focus on raising capital/liquidity, improving their pricing 
algorithm, decreasing their exposure to toxic order flow, and a variety 
of other tasks that are critical for continued success. Users of the 
Valorem-integrated markets will be able to buy, sell, and provide 
liquidity pursuant to their individual needs, which might include 
goals such as hedging, speculation, income generation, and 
diversification.

### Structured Products

Structured products, one of the fastest growing categories of on-chain 
derivatives, are financial products created by combining two or more 
financial instruments into a single, tradeable item that is typically secured 
by the underlying instruments held as collateral by the underwriter.  Many 
structured product underwriters do not focus on trading the underlying 
products, instead, the underwriters’ goal is to handle the execution, 
assignment, and transfer of the underlying in an efficient manner to minimize the 
structured product’s price volatility due to the settlement operations of 
the underlying.  By acting as an efficient clearinghouse, the Valorem protocol
allows structured product protocols to bypass open market actions such as the
sale of underlying options.  If an appropriate counter-party has been 
identified, and the counter party’s protocol has also integrated the Valorem 
protocol into its smart contracts, then the entire structured note creation 
process could be fully automated upon receipt of the purchaser’s intent to 
purchase. The Valorem protocol’s unique vault mechanism decreases the credit 
risk of the product by guaranteeing the full availability of the collateral 
backing the underlying options.

#### Example: Principle Protected Note 

A structured products protocol could accept DAI deposits from a user. 
The protocol then places the DAI into a future yield tokenization protocol like
Alchemix. The principle investment is retained. The future income earning 
potential of this principle is leveraged to buy covered call options on an ERC20 
which the strategy is bullish on. These call options would then be earmarked as 
underlying assets for the user and a new structured product would be minted. 
The protocol would never have to waste funds on open market actions and the users 
would have substantially decreased concern of credit risk of the issuer.  
All assets would be secured.  Should the ERC20 not reach the strike price, 
the structured products protocol would retain the principal; otherwise, the 
option would be exercised or sold in the money upon the user’s redemption of
the structured product, and the user would capture the upside without risk of 
initial principle.

### Vesting Options

Although many options writers may focus on the contract’s expiration date, 
the Valorem protocol also provides them with the ability to set an earliest 
exercise date. Users holding these options are blocked from exercising them 
until the earliest exercise date has passed. While there are many possible 
use cases for being able to effectively "post date a check" in DeFi, one 
that may be of particular interest to protocol developers is the ability to 
create the DeFi version of an Employee Stock Option (ESO) with a cliff vesting
schedule set via the option’s earliest exercise date. Protocols could 
implement Valorem into their vaults and issue these DeFi equivalents of 
ESOs called Developer Token Options (DTO) to developers in key roles. 
This would align the selected developers’ financial compensation with 
the long term success of the protocol thereby decreasing the likelihood of 
protocol abandonment, increasing users’ faith in the developer team, and 
encouraging continued innovation.

### Mitigation of Cross-Protocol Financial Contagion 

Although AMMs are often the first type of DeFi entity to be associated with 
options, the Valorem protocol was designed as a base layer that can be 
integrated by virtually any user type. One extremely niche use case for 
Valorem is to enable lending protocols to manage the risk of accepting 
deposits of untested collateralized stable tokens*. On deposit of the 
collateralized stable token into the lending protocol, the lending protocol 
would require the collateralized stable token protocol to write put options 
against assets (ideally other, safer stable tokens) in the collateralized 
stable token protocol’s vault and transfer those put options to the lending
protocol’s vault. Should the collateralized stable token’s vault not have 
enough free collateral to write a put capable of covering the potential 
deposit, the deposit could be rejected. If the collateralized stable token
loses peg, rather than liquidating users who have borrowed against the stable
token the lending protocol could instead exercise the put and be assigned 
the collateralized stable token’s collateral. This would be extremely 
beneficial for the collateralized stable token protocol because it would 
prevent the cascading market sell-off triggered by automated liquidation 
trades. The lending protocol would have even greater benefits: first, it would 
retain all TVL that would have otherwise been liquidated and captured by 
liquidation bots; second, its risk of financial contagion from the 
collateralized stable coin protocol will have been mitigated due to the 
availability of alternative collateral, safe in Valorem’s vault; and third, 
the lending protocol would increase its TVL due to being able to safely 
accept deposits and facilitate borrowing with new tokens that would have 
been excluded for risk prior to integrating the Valorem protocol. Please 
note that in this scenario, both the lending protocol’s vault smart contract
and the collateralized token’s vault smart contract would need be designed 
with callbacks which may create an attack vector if not done correctly.
