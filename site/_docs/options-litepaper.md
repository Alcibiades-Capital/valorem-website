---
date: 2022-04-13 00:00:00 +01
last_modified_at: 2022-12-22 00:00:00 +01
title: Valorem Options Litepaper
usemathjax: true
description: This paper outlines Valorem Options, an oracle-free, permissionless, clearing and settling system for options on ERC20 tokens.
---

## Introduction

In this paper, we present [Valorem](https://valorem.xyz/) Options, an option contract clearing and settling system implemented for the 
[Ethereum Virtual Machine](https://ethereum.github.io/yellowpaper/paper.pdf).
The design of the Valorem Options protocol aims to provide superior 
flexibility and execution costs when compared with existing options protocols, by removing price 
oracles, reliance on existing DeFi primitives, and premium value assumptions. 
Valorem Options are settled physically using a novel fair settlement 
algorithm. The protocol can interact directly with any pair of non-rebasing, 
non-fee-on-transfer, 
[ERC-20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) 
tokens to enable the transfer, and settlement of long and short put and call 
option positions in a permissionless and non-custodial manner.

## Motivation

Options are an essential component in high-functioning financial systems. In
traditional finance, options volume exceeds spot volume, but in blockchain
finance, spot volumes still exceed options trading volumes. Although options
trading volume on assets like BTC and ETH has grown significantly in the
past year — both centralized exchanges, such as Deribit, and on-chain protocols
—  it is clear that significant untapped market opportunities remain.

There are already a number of on-chain options market making protocols. While
most of these trade products which emulate traditional options structures, the
reliance on price oracles and assumptions around option premiums via models
such as Black-Scholes make them inflexible and subject to toxic orderflow.
Recently, protocols synthesizing options via single tick Uniswap V3 LPs have
emerged, but they are restricted by the lack of Uniswap V3 deployment across
EVM chains, the gas inefficiency of Uniswap V3 LP NFTs, and the pricing
limitations of perpetual options (as opposed to ones with fixed exercise and
expiry [timestamps](https://en.wikipedia.org/wiki/Unix_time)).

A flexible base layer, which is EVM-portable, enables the proliferation of
derivatives products and facilitates the maturity of decentralized finance
by allowing new use cases to evolve.

## Guiding principles

### Permissionless

The Valorem protocol is permissionless. It is open to public use with no
ability to restrict who can or cannot use it. Any potential user can perform
any operation that the protocol supports.

### Physically settled

XYZ

### Fully collateralized

Options written via the protocol are fully collateralized, reducing
counterparty risk and ensuring settlement. This leaves the opportunity for
higher level margining systems, with risk, to be implemented atop the protocol, 
whilst remaining un-opinionated at the base layer.

### Minimal

XYZ, with 5–10x cheaper to buy an option, 20–50x cheaper to write an option, transaction costs compared with similar on-chain protocols.

### Composable

The protocol is composable; the design is centered on generality, such
that it can easily be integrated into other smart contract systems as a
"money lego" to constitute more complex derivatives.

## Mechanism

The Valorem Options protocol, at it's core, is a non-custodial engine for the
clearing and physical settlement of options, consisting of a set of
smart contracts. The engine utilizes the
[ERC1155 multi-token standard](https://ethereum.org/en/developers/docs/standards/tokens/erc-1155/)
to gas-efficiently tokenize long and short positions, or **options** and 
**claims**. On-chain actors — either individuals using wallets, or protocols 
using smart contracts — can create a new option type, defined as the
unique tuple with regard to the option contract's properties. They can then write
options of that type. Writing issues one or more fungible option tokens, and 
a non-fungible claim token, representing a claim to the collateral used for 
writing, or the exercise asset if that claim is assigned exercise via a 
fair assignment algorithm. The tokens can be transferred freely from actors 
to other actors. The option tokens can be exercised during a specified time 
window, after which the claim tokens can be redeemed.

### Creating a new option type

Actors can permissionlessly [create a new option type](/docs/options-smart-contracts#write-options) by specifying:

- The **underlying asset** for the option; this is the token the option 
  holder receives if the option is exercised.
- The **underlying amount** for the option; the amount of the underlying asset
  to be received upon exercise of one option.
- The **exercise asset** for the option; this is the token the option holder 
  pays with to exercise.
- The **exercise amount** of the exercise asset required to exercise one 
  option.
- The earliest **exercise timestamp** of the option.
- The **expiry timestamp** of the option.

#### Option data model

A Valorem option is represented with the [following struct](/docs/options-smart-contracts#option), packed into
4 storage slots:

```solidity
struct Option {
    address underlyingAsset;
    uint96 underlyingAmount;
    address exerciseAsset;
    uint96 exerciseAmount;
    uint40 exerciseTimestamp;
    uint40 expiryTimestamp;
    uint160 settlementSeed;
    uint96 nextClaimKey;
}
```

The key of an option type is defined as the unique tuple with regard to the option 
contract's properties, which comprise a unique hash:

```solidity
uint160 optionKey = uint160(
    bytes20(
        keccak256(
            abi.encode(
                underlyingAsset,
                underlyingAmount,
                exerciseAsset,
                exerciseAmount,
                exerciseTimestamp,
                expiryTimestamp
            )
        )
    )
);
```

This `uint160 optionKey` is used to determine if that type of option already 
exists and, if it doesn't, create it.

#### Claim data model

something something

```solidity
    /**
     * @notice Data about a claim to a short position written on an option type.
     * When writing an amount of options of a particular type, the writer will be issued an ERC 1155 NFT
     * that represents a claim to the underlying and exercise assets, to be claimed after
     * expiry of the option. The amount of each (underlying asset and exercise asset) paid to the claimant upon
     * redeeming their claim NFT depends on the option type, the amount of options written, represented in this struct,
     * and what portion of this claim was assigned exercise, if any, before expiry.
     */
    struct Claim {
        /// @custom:member amountWritten The number of option contracts written against this claim expressed as a 1e18 scalar value.
        uint256 amountWritten;
        /// @custom:member amountExercised The amount of option contracts exercised against this claim expressed as a 1e18 scalar value.
        uint256 amountExercised;
        /// @custom:member optionId The option ID of the option type this claim is for.
        uint256 optionId;
    }
```

#### Token address space

The ERC-1155 standard has a 256-bit address space for TODO BROKEN LINK [sub-tokens](/docs/options-smart-contracts#tokentype). Valorem uses 
the upper 160 bits for fungible option token types, keyed on 
`uint160 optionKey`, and the lower 96 bits for non-fungible claim tokens 
within each option type, keyed on an auto-incrementing `uint96 claimKey` 
starting from one. This results in a 256-bit token address space laid out as 
follows:

```
MSb
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │ 160b option key
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 ┘
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │ 96b claim key
0000 0000   0000 0000   0000 0000   0000 0000 ┘
                                          LSb
```

A tokenId with a valid `optionKey` in the upper 160 bits will be a 
fungible `TokenType.Option` token if it's lower 96 bits is zero, whereas 
it will be a `TokenType.Claim` NFT if it's lower 96 bits contains a 
valid `claimKey`.

This token address space supports `type(uint160).max` option types 
and `type(uint96).max - 1` individual claims for each option type.

### Writing options

Once an option type has been created, any actor can [write options](/docs/options-smart-contracts#write) of that 
type. Upon writing, the requisite amount of the underlying token is transferred 
in to the engine and the option writer receives a non-fungible claim token 
representing the short position, which is a claim to the underlying asset and 
the liability to accept the exercise asset on full or partial exercise 
assignment. In addition, the option writer receives fungible option tokens 
equal to the number of options they wrote, conferring the ability to exercise 
the option pursuant to the terms set during option type creation. Both the 
option tokens and the claim NFT can be transferred to other actors on the chain.

Additional options can be written on existing claims — that is, an option 
writer can add additional underlying assets to a previously written claim by 
providing the claim NFT identifier when writing.

### Exercising options

Holders of an option token can [exercise the option](/docs/options-smart-contracts#exercise) pursuant to the following 
conditions:

- The current block timestamp is on or after the exercise timestamp of the 
  option type.
- The current block timestamp is before the expiry timestamp of the option 
  type.
- The option token owner has enough of the exercise token as required by the 
  terms of the option type.
- The option token owner has granted sufficient ERC-20 approval to the engine,
  so it can transfer in the requisite amount of the exercise token.

### Fair exercise assignment

The Valorem protocol uses a novel pseudo-random fair exercise assignment 
algorithm to determine which claims are assigned exercise. To this end, 
the protocol uses a claim bucketing mechanism to bound the runtime complexity 
of exercise assignment. In this bucketing mechanism, a new bucket is created 
on the first write of a new option type. The amounts written for a bucket are 
stored as $ B_w $, the amount of options written to a bucket, and $ B_e $, the 
amount of options exercised from a bucket. Subsequent writes of the same 
option type will be added to the same bucket until it is assigned an exercise 
and $ B_e > 0 $. At that point, that bucket becomes partially or fully 
exercised, and the next write creates a new bucket. This defines the bucket 
lifecycle algorithm.

TODO insert graphic

Exercise assignment is performed using a deterministic algorithm seeded by
the `uint160 optionKey`, with entropy from actors who write and exercise
options without either party being able to influence the outcome. The runtime 
complexity of this algorithm is $ \mathcal{O}(n) $ where $ n $ is the number 
of buckets consumed by the algorithm to fulfill the exercise.

TODO check if this is still accurate wording "with entropy from actors who write and exercise options without either party being able to influence the outcome"

However, because the probability of an exercise assignment to 
the most recently created bucket is $ 1 \over n $, where $ n $ is the number 
of buckets, and because $ \sum_{n \rightarrow \infty} {1 \over n} = { \infty } $, 
and since 
$ \sum_{n=1}^k {1 \over n} = H_k $, and $ H_k = \sum_{n=1}^k {1 \over n} \approx \ln n + \gamma $, and $ \gamma \approx 0.5772156649 $, 
the average case growth rate of the number of buckets for an option type 
is $ \mathcal{O}(\ln n) $. This makes it prohibitively expensive for a
malicious writer to perform a denial of service attack on options exercisers,
and generally keeps the runtime complexity to exercise an option type bounded
by $ \mathcal{O}(\ln n) $ where $ n $ is the number of writes followed by a 
partial exercise which occur.

#### What comprises a claim?

Because of the bucketing mechanism and ability to add additional options to
an existing claim, Valorem claims are comprised of claim index data structures
which are stored for each bucket $ i $ the claim is written into as $ I_{wi} $, 
the amount of options written by that claim to bucket $ i $. Thus, calculating
a claim position involves summations over the claim's member buckets. 

#### Calculating exercise state for a claim

Given $B_{ei}$, the amount of options exercised from bucket $ i $, and 
$ B_{wi} $, the amount of options written into bucket $ i $, and 
$ B_{ui} = B_{wi} - B_{ei} $, the amount of options unexercised in
bucket $ i $, we can calculate $ C_e $, the amount of options exercised for a claim, 
and $ C_u $ the amount of options unexercised for a claim using the following 
summations:

$$ C_e = \sum_{i=1}^n C_{ei} = {B_{ei} I_{wi} \over B_{wi}} + \ldots + {B_{en} I_{wn} \over B_{wn}} $$

and,

$$ C_u = \sum_{i=1}^n C_{ui} = {B_{ui} I_{wi} \over B_{wi}} + \ldots + {B_{un} I_{wn} \over B_{wn}} $$

and we can verify that $ C_e + C_u = C_w $, options written for a given claim 
$ C $,

$$ C_w = \sum_{i=1}^n C_{wi} = ({B_{ei} I_{wi} \over B_{wi}} + {B_{ui} I_{wi} \over B_{wi}}) + \ldots + ({B_{en} I_{wn} \over B_{wn}} + {B_{un} I_{wn} \over B_{wn}}) $$

which simplifies to,

$$ C_w = \sum_{i=1}^n C_{wi} = I_{wi} + \ldots + I_{wn} $$

which is the sum of options written by a claim to each of it's member buckets. 

#### Calculating the position for a claim

To preserve as much precision as possible, we calculate the amounts of the 
exercise token position, $ P_e $, and underlying token position, $ P_u $, 
tokens collateralizing a claim by multiplying the amount of the exercise 
asset, $ O_e $, and the underlying asset, $ O_u $, before performing any 
division. Resulting in the following summations: 

$$ P_e = \sum_{i=1}^n P_{ei} = {B_{ei} O_e I_{wi} \over B_{wi}} + \ldots + {B_{en} O_e I_{wn} \over B_{wn}} $$

and 

$$ P_u = \sum_{i=1}^n P_{ui} = {B_{ui} O_u I_{wi} \over B_{wi}} + \ldots + {B_{un} O_u I_{wn} \over B_{wn}} $$

### Redeeming claims

Holders of a claim NFT can [redeem their claim](/docs/options-smart-contracts#redeem) from the engine when current 
block timestamp is on or after the expiry timestamp of the option type. If their 
claim was assigned full or partial exercise during the lifetime of the option 
type, the claim holder receives the correct ratio of the underlying and 
exercise tokens. If the claim NFT was not assigned exercise, the claim 
holder will receive the underlying tokens deposited upon writing back in full.

## Use cases

### Simple options

Where:

$ S_T $ is the price of the underlying asset at expiration; and $ X $ is the 
exercise asset price; and $ c_0 $ is the call option premium paid; and 
$ p_0 $ is the put option premium paid.

#### Call options

The Valorem protocol can be used to create covered call options with the 
payoff $ max(0,S_T-X) $ for the holder, and the payoff  $ -max(0,S_T-X) $ for 
the writer.

#### Put options

The Valorem protocol can be used to create covered put options with the 
payoff $ max(0,X-S_T) $ for the holder, and the payoff  $ -max(0,X-S_T) $ 
for the writer. This can be accomplished by writing a call option with the 
exercise and underlying asset order swapped. An ETH/DAI call is a DAI/ETH put.

#### European and American options

European options can be created by setting the exercise timestamp to the 
expiry timestamp minus one day. American options can be created by setting 
an exercise timestamp to the current block timestamp upon creation. Bermudan
options can be created by setting an exercise timestamp to a future timestamp.

### Trading and market making

The Valorem protocol provides web3 developers with an options base layer that 
can be seamlessly integrated into existing and future AMMs and CLOBs. By acting 
as the clearinghouse and settlement service needed for options execution, 
Valorem allows market makers to trade options without the need to implement 
their own options-specific smart contract adjustments, risk management system, 
or collateral assignment process. This frees up MMs to focus on raising 
capital/liquidity, improving their pricing algorithm, decreasing their exposure 
to toxic order flow, and a variety of other tasks that are critical for 
continued success. Users of the Valorem-integrated markets will be able to 
buy, sell, and provide liquidity pursuant to their individual needs, which 
might include goals such as hedging, speculation, income generation, and 
diversification.

### Structured products

Structured products, one of the fastest growing categories of on-chain 
derivatives, are financial products created by combining two or more financial 
instruments into a single, tradeable item that is typically secured by the 
underlying instruments held as collateral by the underwriter.  Many structured 
product underwriters do not focus on trading the underlying products; instead, 
the underwriters’ goal is to handle the execution, assignment, and transfer of 
the underlying in an efficient manner to minimize the structured product’s 
price volatility from settlement operations.

By acting as an efficient clearinghouse, the Valorem protocol allows structured 
product protocols to bypass open market actions such as the sale of underlying 
options.  If an appropriate counter-party has been identified, and the 
counterparty’s protocol has also integrated the Valorem protocol into its 
smart contracts, then the entire structured product creation process could be 
fully automated upon receipt of the purchaser’s intent to purchase. The Valorem 
protocol’s unique vault mechanism decreases the credit risk of the product by 
guaranteeing the full availability of the collateral backing the underlying
options.

#### Principle protected note

A structured product's protocol could accept DAI deposits from a user. The 
protocol then places the DAI into a future yield tokenization protocol like 
Alchemix. The principle investment is retained. The future income earning 
potential of this principle is leveraged to buy covered call options on an 
ERC20 which the strategy is bullish on. These call options would then be 
earmarked as underlying assets for the user and a new structured product would 
be minted. The protocol would never have to waste funds on open market actions 
and the users would have substantially decreased concern of credit risk of the 
issuer. All assets would be secured. Should the ERC20 not reach the strike 
price, the structured products protocol would retain the principal; otherwise, 
the option would be exercised or sold in the money upon the user’s redemption 
of the structured product, and the user would capture the upside without risk 
of initial principle.

### Vesting options

Although price-sensitive options writers may focus on the contract’s expiration 
date, the Valorem protocol also provides writers with the ability to set the 
earliest exercise date. Users holding these options are blocked from exercising 
them until the earliest exercise date has passed. While there are many possible 
use cases for being able to effectively "post date a check" in DeFi, one that
may be of particular interest to protocol developers is the ability to create
the DeFi version of an Employee Stock Option (ESO) with a cliff vesting 
schedule set via the option’s earliest exercise date. Protocols could implement
Valorem into their vaults and issue these DeFi equivalents of ESOs called 
Contributor Token Options (CTO) to contributors in key roles. This would align 
the selected contributors’ financial compensation with the long term success of 
the protocol thereby decreasing the likelihood of protocol abandonment, 
increasing users’ faith in the team, and encouraging continued innovation.

### Mitigation of cross-protocol financial contagion

Although AMMs are often the first type of DeFi entity to be associated with 
options, the Valorem protocol was designed as a base layer that can be 
integrated by virtually any user type. One niche use case for Valorem is to 
enable lending protocols to manage the risk of accepting deposits of untested 
collateralized stable tokens. 

On deposit of the collateralized stable token into the lending protocol, the 
lending protocol would require the collateralized stable token protocol to
write put options against assets (ideally other, safer stable tokens) in the
collateralized stable token protocol’s vault and transfer those put options 
to the lending protocol’s vault. Should the collateralized stable token’s 
vault not have enough free collateral to write a put capable of covering the 
potential deposit, the deposit could be rejected. If the collateralized stable 
token loses peg, rather than liquidating users who have borrowed against the 
stable token the lending protocol could instead exercise the put and be 
assigned the collateralized stable token’s collateral. This would be extremely 
beneficial for the collateralized stable token protocol because it would 
prevent the cascading market sell-off triggered by automated liquidation 
trades.

The lending protocol would have even greater benefits: first, it would retain 
all TVL that would have otherwise been liquidated and captured by liquidation 
bots; second, its risk of financial contagion from the collateralized stable 
coin protocol will have been mitigated due to the availability of alternative 
collateral, safe in Valorem’s vault; and third, the lending protocol would 
increase its TVL due to being able to safely accept deposits and facilitate 
borrowing with new tokens that would have been excluded for risk prior to 
integrating the Valorem protocol. Please note that in this scenario, both 
the lending protocol’s vault smart contract and the collateralized token’s 
vault smart contract would need be designed with callbacks which may create 
an attack vector if not done correctly.
