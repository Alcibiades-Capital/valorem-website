---
date: 2022-09-04 00:00:00 +01
title: Contributor token options with Valorem 
description: In our litepaper we highlight contributor tokens as a use case for Valorem. Here's how it works.
image: '/assets/images/posts/abstract-2.jpg'
---

In the web2 world, equity is a big part of compensation for employees. Web3 is a bit 
different. Equity doesn't matter so much, there aren't employees per se. Instead, we 
have tokens, and contributors. Presently, many contributors are paid in stablecoins, 
and also in token equity. The issue with compensating contributors in tokens, is that 
they have a zero cost basis and no lock-up. This can have negative effects on the 
protocol tokenomics.

![Writing Contributor Token Options with Valorem](/assets/images/posts/contributor-token-options.png)

In our [litepaper](https://valorem.xyz/docs/clear-litepaper#vesting-options),
we outlined an alternative; Valorem contributor token optionsg. Although our 
options primitive is not on mainnet yet, our UI is at a point where it's easy to 
show how simple it is to create token options for contributors with Valorem. Let's 
say that MakerDAO wanted to compensate contributors with MKR from the treasury, 
they could easily write a call option for 1 MKR in exchange for 700 DAI vesting 
3 months from now, and expiring a year after that, as pictured above. MakerDAO 
would then get the options tokens, and a claim NFT. The options tokens could be 
distributed to contributors, and after expiry, MakerDAO could redeem the mix of 
MKR and DAI using the claim NFT.