---
date: 2022-11-30 00:00:00
title: Unraveling Valorem Clear - A Revolution in DeFi Options Clearing
description: Learn about the transformative capabilities of Valorem Clear, a cutting-edge DeFi clearing and settlement system for options. Understand how it streamlines covered calls, cash-secured puts, and more.
redirect_from:
  - /docs/smart-contracts-overview/
  - /docs/clearinghouse-overview/
---

Welcome to [Valorem's](https://valorem.xyz/) next-gen innovation – Valorem Clear. This powerful clearing system 
empowers enthusiasts to navigate the world of covered calls, cash-secured puts, and various option types, be it 
American, European, or Exotic.

### Dive Into the Blueprint

Version 1 of Valorem Clear is much more than just an engine — it's a symphony of clearing and settling. Users can 
effortlessly pen options, bring them to fruition, and assert claims on their assets. The elegance lies in the 
engine's capability to fairly match exercises with written claims. Designed for optimal gas efficiency, this 
minimalistic powerhouse offers a rock-solid foundation for building advanced derivatives structures.

Here's the gist: Valorem Clear is in harmony with the [ERC-1155 multi-token](https://eips.ethereum.org/EIPS/eip-1155) ethos. So, you can script options for any compliant ERC-20 duo, with a few exceptions. Once penned, these options manifest as semi-fungible Option tokens—tradeable, transferrable, and tangible.

For those who pen options, their stake in the underlying assets is depicted through a unique Claim token. This token can be redeemed, reflecting the asset's current status, be it exercised or not.

To provide a snapshot:
- **Underlying Asset & Amount:** The ERC-20 asset you'll receive when you exercise.
- **Exercise Asset & Amount:** The requisite ERC-20 asset and amount for option exercise.
- **Exercise & Expiry Timestamp:** The windows defining when the option can be exercised.

The elegance of Valorem Clear? It's neutral. Whether it's a call or a put, where you purchase it, its profitability upon exercise—Valorem doesn't judge. With every penned option backed to the hilt, the settlement is swift and gas-friendly.

## Trust Landscape

### The Protagonists:

1. **Protocol Admin:** This entity oversees the operational side of the protocol. The Protocol Admin is the beneficiary of protocol fees. Their powers include:
    - Modifying the Protocol Admin address.
    - Tweaking the contract address of the TokenURIGenerator.
    - Switching the protocol fees on or off.
    - Collecting the accrued fees.

2. **Option Writers:** The creators who give birth to options. Their capabilities are:
    - Crafting new option types for valid ERC-20 asset pairs, albeit with certain exceptions.
    - Writing new options and staking the required amount of the underlying asset.
    - Redeeming their Claim NFT after the option's expiration to retrieve their proportionate share of the underlying and exercise assets.

3. **Option Holders:** The stewards of the options once they are written. Their roles include:
    - Transferring the Option tokens to other addresses based on ERC-1155 norms.
    - Exercising the options they hold, ensuring they possess sufficient exercise assets and have provided the necessary approvals.

### Asset Dynamics:

- **Option Creation:** As an option springs into existence, Valorem Clear pulls in the designated amount of the underlying asset.

- **Option Exercise:** When an option is exercised, Clear ushers in the necessary exercise asset and sends out the underlying one.

- **Claim Redemption:** Post the expiration of an option, Clear disburses the claimant's proportionate share of both the exercise asset and any remaining unexercised underlying asset.

### Actions:

- **General (`ERC1155` Interactions):**
    - Anyone can peek into token balances, ownership details, transfer approvals, supported interfaces (like ERC165 and ERC1155), and render the URI for any token.

- **Valorem’s Options Clearinghouse Dynamics:**
    - Any participant can:
        - Dig deep into details of any Option fungible token ID or Claim non-fungible token ID.
        - Check the positions of underlying and exercise assets of any token.
        - Identify if a token ID represents an Option or a Claim.
        - Explore contract details of the TokenURIGenerator, verify fee balances of ERC20 assets, and glean other protocol-specific details.
        - Craft a new Option Type or script a new Claim for an existing Option Type.
    - Claim NFT holders (Option Writers) can:
        - Write new options aligned to their Claim NFT before its expiry.
        - Post the expiry, redeem their Claim NFT to fetch their share of assets.
    - Option token holders (Option Holders) can:
        - Transfer Option tokens and exercise them within the stipulated window.
    - The Protocol Admin can:
        - Toggle protocol fees, appoint a new Protocol Admin, modify the contract address of the TokenURIGenerator, and gather the accrued fees.

## Safety First

While Valorem Clear v1.0.0 has been diligently audited, it's essential to note a medium-risk observation. The odds of an option bucket being chosen for an exercise isn't strictly proportional to its size. This means smaller buckets might have a marginally higher chance of selection. Our tech wizards are hard at work, devising a remedy. Until then, for the curious minds, the nitty-gritty can be found at [LibDDRV](https://github.com/valorem-labs-inc/LibDDRV).

- Deep Dives: [Audit Chronicles](https://github.com/valorem-labs-inc/valorem-core/tree/master/audits)
- Connect with our Security Guardians: info(at)valorem(dot)xyz
