---
date: 2022-04-13 00:00:00 +01
last_modified_at: 2022-12-16 00:00:00 +01
title: Valorem Options Litepaper
usemathjax: true
description: This paper outlines the Valorem Options protocol, an oracle-free, permissionless, underwriter and clearinghouse for ERC20 token options. 
---

## Introduction

This litepaper introduces the Valorem Options protocol, a DeFi native options 
protocol which aims to provide superior flexibility in comparison with existing 
options protocols by removing price oracles, reliance on existing defi 
primitives, and premium value assumptions. Valorem achieves this by 
implementing an option contract underwriting system and clearinghouse driven 
by market forces and settled physically using a novel fair settlement 
algorithm. The Valorem Options protocol consists of a set of smart contracts 
targeting the 
[Ethereum Virtual Machine](https://ethereum.github.io/yellowpaper/paper.pdf), 
or EVM, which can interact directly with any pair of non-rebasing, 
non-fee-on-transfer,
[ERC-20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/)
tokens to enable the transfer, and settlement of long and short put 
and call option positions in a permissionless manner.

## Motivation

Options are an essential component in high-functioning financial systems.
In traditional finance, options volume exceeds spot volume, but in blockchain 
finance, spot volumes still exceed options trading volumes. Although options 
trading volume on assets like BTC and ETH has grown significantly in the past 
year &#151; both on centralized exchanges such as Deribit, and on-chain 
protocols &#151; it is clear that significant untapped market opportunities 
remain.

There are already a number of on-chain options market making protocols. While 
most of these trade products that emulate traditional options structures, 
the reliance on price oracles and assumptions around option premiums via 
models such as black-scholes make them inflexible and subject to toxic 
orderflow. Recently, protocols synthesizing options via single tick Uniswap 
V3 LPs have emerged, but they are restricted by the lack of Uniswap V3 
deployment across evm chains, gas inefficiency of Uniswap V3 LP NFTs, 
and the pricing limitations of perpetual options (as opposed to ones with 
fixed exercise and expiry timestamps).

A flexible base layer for that is portable to EVMs enables the 
proliferation of derivatives products and facilitate the maturity of 
decentralized finance by allowing new use cases to be developed.

## Solution

### Permissionless

The Valorem protocol is permissionless; the protocol is open to public use 
with no ability to restrict who can or cannot use it. Any potential user 
can perform any operation that the protocol supports.

### Fully collateralized

Options written via the protocol are fully collateralized, eliminating 
counterparty risk and ensuring settlement.

### Composable

The protocol is composable; generality is centered in the design such
that it can easily be integrated into other smart contract systems as a 
"money lego" to comprise more complex derivatives.

## Mechanism

The Valorem protocol, at its core, is a non-custodial engine for the
underwriting and physical settlement of options. The engine utilizes the 
[ERC1155 multi-token standard](https://ethereum.org/en/developers/docs/standards/tokens/erc-1155/)
to gas-efficiently tokenize long and short positions, or options and claims. 
Actors on chain &#151; either individuals using their wallets or protocols 
using smart contracts &#151; can create a new option type, defined as the 
unique tuple with regard to the option contract's properties. They can then write
options of that type. Writing issues a fungible option token, and a non-fungible 
claim token, representing a claim to the collateral used for writing, or the 
exercise asset if assigned exercise via a fair assignment algorithm. The tokens 
can be transferred freely from actors to other actors. The option tokens can 
be exercised during a specified time window, after which the claim tokens can 
be redeemed.

### Timekeeping

All timekeeping in the Valorem options protocol uses `uint40` timestamps in
seconds since the unix epoch.

### Creating an option type

Actors can permissionlessly create new option types. They can specify the 
following:

- The underlying collateral token of the option; this is what the option 
  owner receives if the option is exercised.
- The amount of the underlying token collateralizing the option.
- The exercise token of the option; this is what the option owner pays to exercise the option.
- The amount of the exercise token required to exercise the option.
- The earliest exercise timestamp of the option.
- The expiration timestamp of the option.

This information comprises a unique hash:

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
       )
```

which is then used to determine if that type of option already exists and, if
it doesn't, create it.

### Token address space

The ERC1155 standard has a 256-bit address space for sub-tokens.
Valorem uses 160 bits for fungible option tokens keyed on `uint160 optionKey`, 
and 96 bits for claim NFTs per option type, keyed on an auto-incrementing 
`uint96 claimKey`, starting from 1, resulting in a 256-bit token address 
space laid out as follows:

```
MSb
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │ 160b option key.
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 ┘
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │ 96b claim key.
0000 0000   0000 0000   0000 0000   0000 0000 ┘
                                            LSb
```

Supporting `uint160` option types with `uint96 - 1` claim NFTs each.

### Writing options

Once an option type has been created, an actor can write options of that type.
Upon writing, the option writer will receive a non-fungible claim token representing the 
short position, which is a claim to the underlying asset and the responsibility 
to accept the exercise asset on full or partial exercise assignment. They will 
also receive option tokens equal to the amount of the option type they have
written, conferring the ability to exercise the option pursuant to the terms 
set during type creation. Both the claim token and the option token can be 
transferred to other actors on the chain.

Claims are also able to have additional options written on them. That is, an
options writer can add additional underlying assets to a previously written
claim by providing the claim ID to the `write` function.

### Exercising options

Holders of an option token can exercise the option pursuant to the following
conditions:

- The current timestamp is at least the exercise timestamp of the option type. 
- The current timestamp is before the expiry timestamp of the option type. 
- The owner of the option token has enough of the exercise token as required
  by the terms of the option type.

### Redeeming options

Holders of a claim token can redeem their claim from the Valorem protocol 
vault on or after the expiration timestamp of the option type for the claim. 
If their claim was assigned full or partial exercise during the lifetime of 
the option type, the claim holder will receive some ratio of the underlying 
and exercise tokens. If the claim token was not assigned exercise, the claim 
token holder will receive the underlying tokens deposited upon writing 
back in full.

### Buckets

The Valorem protocol uses a claim bucketing mechanism to bound the runtime 
complexity of exercise assignment. This also to ensures that malicious option 
writers cannot perform a denial of service attack on the protocol by writing 
large numbers of claims on an option type, which, without the bucketing 
mechanism, could result in exercise assignment becoming prohibitively expensive.

In this mechanism, a new bucket is created on the first write of a new option 
type. Subsequent writes of the same option type will be added to the same 
bucket until it is assigned an exercise. At that point, that bucket becomes 
partially or fully exercised, and the next write creates a new bucket. Thus, 
a bucket has an enumeration of states:

```solidity
enum BucketExerciseState {
        Exercised,
        PartiallyExercised,
        Unexercised
    }
```

The amounts written for a bucket are stored in a `Bucket` struct:

```solidity
struct Bucket {
    /// amountWritten The number of option contracts written into this bucket.
    uint112 amountWritten;
    /// amountExercised The number of option contracts exercised from this bucket.
    uint112 amountExercised;
}
```

and the bucket lifecycle algorithm is:

```solidity
function _addOrUpdateClaimBucket(
    OptionTypeState storage optionTypeState,
    uint112 amount
  ) private returns (uint96 bucketIndex) {
  BucketInfo storage bucketInfo = optionTypeState.bucketInfo;
  Bucket[] storage claimBuckets = bucketInfo.buckets;
  uint96 writtenBucketIndex = uint96(claimBuckets.length);

  if (claimBuckets.length == 0) {
    // Then add a new bucket to this option type, because none exist.
    claimBuckets.push(Bucket(amount, 0));
    _updateUnexercisedBucketIndices(bucketInfo, writtenBucketIndex);

    return writtenBucketIndex;
  }

  // Else, get the currentBucket.
  uint96 currentBucketIndex = writtenBucketIndex - 1;
  Bucket storage currentBucket = claimBuckets[currentBucketIndex];

  if (
    bucketInfo.bucketExerciseStates[currentBucketIndex]
    != BucketExerciseState.Unexercised
  ) {
    // Add a new bucket to this option type, because the last was exercised.
    claimBuckets.push(Bucket(amount, 0));
    _updateUnexercisedBucketIndices(bucketInfo, writtenBucketIndex);
  } else {
    // Write to the existing unexercised bucket
    currentBucket.amountWritten += amount;
    writtenBucketIndex = currentBucketIndex;
  }

  return writtenBucketIndex;
}

```

The probability of an exercise assignment to the most recently created bucket 
is $ 1 \over n $, where $ n $ is the number of buckets. Because 
$ \sum_{n \rightarrow \infty} {1 \over n} = { \infty } $, and since
$ \sum_{n=1}^k {1 \over n} = H_k $, and 
$ H_k = \sum_{n=1}^k {1 \over n} \approx \ln n + \gamma $, and 
$ \gamma \approx 0.5772156649 $, the average case growth rate of the number 
of buckets for an option type is $ \mathcal{O}(\ln n) $. This makes it 
prohibitively expensive for a malicious writer to perform a denial of service 
attack on options exercisers, and generally keeps the runtime complexity to 
exercise an option type bounded by $ \mathcal{O}(\ln n) $.

### What comprises a claim?

Because of the bucketing mechanism and ability to add additional options to an
existing claim, Valorem claims are comprised of claim index data structures:

```solidity
struct ClaimIndex {
        /// amountWritten The amount of option contracts written into claim for given bucket.
        uint112 amountWritten;
        /// bucketIndex The index of the Bucket into which the options collateral was deposited.
        uint96 bucketIndex;
    }
```

which are stored for each bucket the claim is written into. A claim might be 
composed of options written into multiple buckets, and the `ClaimIndex` stores 
how many options are written into which bucket. Calculating a claim position
involves summations over the claim's bucket indices.

#### Calculating exercise state for a claim

Given $ I_wi $, the amount of options written into bucket i for a claim, $ C $,
we can calculate $ C_e $, the amount of options exercised for a claim, and 
$ C_u $ the amount of options unexercised for a claim using $ B_wi $, the amount of
options written into bucket i, and $ B_ei $ the amount of options exercised from 
bucket i.

We can calculate the remaining amount of options unexercised for bucket i as

$$ B_ui = B_wi - B_ei $$

therefore, with 

$ C_ei $ = the amount of options exercised in bucket i for a particular claim, 

and

$ C_wi $ = the amount of options written in bucket i for a particular claim:

$$ C_e = \sum_{i=1}^n C_ei = {B_ei I_wi \over B_wi} + \ldots + n $$

and

$$ C_u = \sum_{i=1}^n C_wi = {B_ui I_wi \over B_wi} + \ldots + n $$

and

$$ C_w = \sum_{i=1}^n i = ({B_ei I_wi \over B_wi} + {B_ui I_wi \over B_wi}) + \ldots + n $$

which simplifies to

$$ C_w = \sum_{i=1}^n i = I_wi + \ldots + n $$

#### Calculating underlying assets for a claim

To preserve as much precision as possible, we calculate the amounts of the 
exercise, $ U_e $, and underlying, $ U_u$, tokens collateralizing a claim by 
multiplying the amount of the exercise asset, $ O_e $, and the underlying 
asset, $ O_u $, before performing any division. Thus:

$$ U_e = \sum_{i=1}^n exercised in ith bucket = {B_ei O_e I_wi \over B_wi} + \ldots + n $$

and 

$$ U_u = \sum_{i=1}^n unexercised in ith bucket = {B_ui O_u I_wi \over B_wi} + \ldots + n $$

### Option exercise assignment

Exercise assignment is performed using a deterministic algorithm seeded by
the `uint160 optionKey`, with entropy from actors who write and exercise 
options without either party being able to influence the outcome.

The assignment algorithm is as follows:

```solidity
function _assignExercise(
        OptionTypeState storage optionTypeState,
        Option storage optionRecord,
        uint112 amount
    ) private {
        // Setup pointers to buckets and buckets with collateral available for exercise.
        Bucket[] storage buckets = optionTypeState.bucketInfo.buckets;
        uint96[] storage unexercisedBucketIndices =
            optionTypeState.bucketInfo.unexercisedBucketIndices;
        uint96 numUnexercisedBuckets = uint96(unexercisedBucketIndices.length);
        uint96 exerciseIndex =
            uint96(optionRecord.settlementSeed % numUnexercisedBuckets);

        while (amount > 0) {
            // Get the claim bucket to assign exercise to.
            uint96 bucketIndex = unexercisedBucketIndices[exerciseIndex];
            Bucket storage bucketInfo = buckets[bucketIndex];

            uint112 amountAvailable =
                bucketInfo.amountWritten - bucketInfo.amountExercised;
            uint112 amountPresentlyExercised = 0;
            if (amountAvailable <= amount) {
                amount -= amountAvailable;
                amountPresentlyExercised = amountAvailable;
                // Perform "swap and pop" index management.
                numUnexercisedBuckets--;
                uint96 overwrite =
                    unexercisedBucketIndices[numUnexercisedBuckets];
                unexercisedBucketIndices[exerciseIndex] = overwrite;
                unexercisedBucketIndices.pop();

                optionTypeState.bucketInfo.bucketExerciseStates[bucketIndex] =
                    BucketExerciseState.Exercised;
            } else {
                amountPresentlyExercised = amount;
                amount = 0;
                optionTypeState.bucketInfo.bucketExerciseStates[bucketIndex] =
                    BucketExerciseState.PartiallyExercised;
            }
            bucketInfo.amountExercised += amountPresentlyExercised;

            if (amount != 0) {
                exerciseIndex = (exerciseIndex + 1) % numUnexercisedBuckets;
            }
        }
  // Update the seed for the next exercise.
  optionRecord.settlementSeed = uint160(
    uint256(
      keccak256(
        abi.encode(optionRecord.settlementSeed, exerciseIndex)
      )
    )
  );
}
```

The runtime complexity of this algorithm is $ \mathcal{O}(n) $ where $ n $ is 
the number of buckets consumed by the algorithm to fulfill the exercise. 
However, the average case runtime complexity is better than
$ \mathcal{O}(\ln n) $, since we know the growth rate of the number of 
buckets, and that we are operating on a subset of buckets.

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
execution, Valorem allows market makers to trade options without the
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
products; instead, the underwriters’ goal is to handle the execution,
assignment, and transfer of the underlying in an efficient manner to minimize the
structured product’s price volatility from settlement operations.  
By acting as an efficient clearinghouse, the Valorem protocol
allows structured product protocols to bypass open market actions such as the
sale of underlying options.  If an appropriate counter-party has been
identified, and the counter party’s protocol has also integrated the Valorem
protocol into its smart contracts, then the entire structured product creation
process could be fully automated upon receipt of the purchaser’s intent to
purchase. The Valorem protocol’s unique vault mechanism decreases the credit
risk of the product by guaranteeing the full availability of the collateral
backing the underlying options.

#### Example: Principle Protected Note

A structured product's protocol could accept DAI deposits from a user.
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

### Example: Vesting Options

Although price-sensitive options writers may focus on the contract’s expiration date,
the Valorem protocol also provides writers with the ability to set an earliest
exercise date. Users holding these options are blocked from exercising them
until the earliest exercise date has passed. While there are many possible
use cases for being able to effectively "post date a check" in DeFi, one
that may be of particular interest to protocol developers is the ability to
create the DeFi version of an Employee Stock Option (ESO) with a cliff vesting
schedule set via the option’s earliest exercise date. Protocols could
implement Valorem into their vaults and issue these DeFi equivalents of
ESOs called Contributor Token Options (CTO) to contributors in key roles.
This would align the selected contributors’ financial compensation with
the long term success of the protocol thereby decreasing the likelihood of
protocol abandonment, increasing users’ faith in the team, and encouraging 
continued innovation.

### Example: Mitigation of Cross-Protocol Financial Contagion

Although AMMs are often the first type of DeFi entity to be associated with
options, the Valorem protocol was designed as a base layer that can be
integrated by virtually any user type. One niche use case for
Valorem is to enable lending protocols to manage the risk of accepting
deposits of untested collateralized stable tokens. 

On deposit of the collateralized stable token into the lending protocol, the  
lending protocol would require the collateralized stable token protocol to write 
put options against assets (ideally other, safer stable tokens) in the collateralized
stable token protocol’s vault and transfer those put options to the lending
protocol’s vault. Should the collateralized stable token’s vault not have
enough free collateral to write a put capable of covering the potential
deposit, the deposit could be rejected. If the collateralized stable token
loses peg, rather than liquidating users who have borrowed against the stable
token the lending protocol could instead exercise the put and be assigned
the collateralized stable token’s collateral. This would be extremely
beneficial for the collateralized stable token protocol because it would
prevent the cascading market sell-off triggered by automated liquidation
trades.

The lending protocol would have even greater benefits: first, it would
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
