---
date: 2023-06-03 00:00:00 +01
title: Building a dynamic PMF random variate generator for on-chain games
description: Dive into our ETH Denver 2023 Hackathon project where we built a Dynamic PMF Random Variate Generator using Solidity for blockchain-based games. Discover how this innovation enhances fairness for on-chain games.
usemathjax: true
image: '/assets/images/posts/blackjack.jpg'
---

In late February, Valorem was deep in an
[audit of Valorem Clear with Zellic](https://github.com/valorem-labs-inc/valorem-core/blob/master/audits/Valorem_April_2023_-_Zellic_Audit_Report.pdf),
while simultaneously preparing for the
[ETH Denver 2023 Hackathon](https://www.ethdenver.com/). During our
audit, one issue regarding the fairness of our option exercise assignment
algorithm emerged as a challenging problem. We managed to get launch-ready by
making the algorithm deterministic, ensuring fair play for both players, writer
and exerciser, involved in the option. Yet, we were convinced we could
enhance it further.

Ahead of the hackathon, we delved into a more comprehensive solution. It became
clear that we needed a mechanism for generating random variates fitting a
dynamically weighted discrete probability mass function (PMF). We aimed to
leverage this unique type of random number to equalize the probability of
[option exercise assignment across various buckets]((https://valorem.xyz/docs/clear-litepaper/#fair-exercise-assignment)).

## A random forest in Solidity

A simple implementation of dynamic PMF random variate generation could be
easily implemented but, unfortunately, runs in linear time, making it
unsuitable for the Ethereum Virtual Machine (EVM). Our research led us to a
1993 research paper by Matias, Vitter, and Ni, referenced in Knuth’s The Art
of Computer Programming.
[This paper](https://kuscholarworks.ku.edu/bitstream/handle/1808/7224/MVN03.dynamic_rv_gen.pdf)
proposed an innovative approach that operates in log n time, using a forest of
trees data structure, and takes a uniform random variate (URV) as input. This
URV can be sourced from a reliable oracle like Chainlink VRF, a randomness
beacon like RANDAO, or a pseudorandom source like a collaborative entropy queue.

![Working on LibDDRV in the metaverse](/assets/images/posts/metaverse-coworking.jpg)

This research was the basis for our new randomness library for the Solidity
ecosystem, [LibDDRV](https://github.com/valorem-labs-inc/LibDDRV).
The library enables the generation of random variates conforming to a
dynamically weighted discrete PMF. It required considerable effort in the
Metaverse, as well as with traditional pen and paper, to devise pseudocode
that would later be transformed into Solidity. Our generator thrives
in the dynamic case, facilitating weight updates and generating random variates
efficiently.

![Working on LibDDRV with pencil and paper](/assets/images/posts/pencil-and-paper.jpg)

With the library still under development going into the hackathon, we sought
on-chain games that could benefit from it. Our decision was to continue with
the library implementation while competing in the hackathon gaming track.
After some consideration, we realized we could utilize the library to
construct a European no-hole-card blackjack game without zero-knowledge proofs.

## A blackjack shoe in Solidity

Historically, static methods such as Walker's Alias Method have been
employed to generate random variates with a constant set of weights. However,
these methods are inadequate in a dynamic context like a Blackjack game,
where the odds of drawing a specific card change with every dealt card.
Blackjack is a game of chance, but it's also a game of skill. The skill comes in
understanding the probabilities of the game, and how they dynamically change
based on the cards which have already been drawn. This translates almost
perfectly into a dynamically weighted discrete probability mass function (PMF).

Thus, we integrated LibDDRV into a Solidity smart contract for a game of
[EVM Blackjack](https://github.com/valorem-labs-inc/evm-blackjack),
which implements game rules and provides players with the ability to place
bets, receive cards, hit, or stand. Along with a
[user interface](https://github.com/valorem-labs-inc/evm-blackjack-ui) so that
users could play the game. All of this was a bit ambitious for a hackathon, but
we did get pretty far along, particularly with the library.

## Doubling-down: A dynamic future

The hackathon was a whirlwind of coding and innovation, and we have much more
to explore. The ability to generate dynamic random variates could greatly
improve blockchain-based prediction markets or gaming scenarios where each event
influences the odds of subsequent events. Potential applications range from
NFTs, GameFi, DeFi, and DAOs to more technical uses like fuzz testing inputs
for real-world scenario modeling and consensus algorithms.

## Wrapping up

ETH Denver 2023 served as a fantastic platform to connect theoretical
probability concepts with their practical on-chain implementations. Our journey
continues as we strive to refine our implementation, contributing to the
evolving landscape of randomness in the EVM ecosystem, and beyond.

What other on-chain problems could LibDDRV solve? 🤔

**Credit to [Neodaoist](https://twitter.com/neodaoist),
[Nick Adamson](https://twitter.com/nickadams0n),
[Flip](https://twitter.com/flip_liquide), and
[0xAlcibiades](https://twitter.com/0xAlcibiades) for their contributions during
the hackathon.** 

