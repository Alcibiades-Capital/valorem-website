---
date: 2022-11-30 00:00:00
title: Valorem Clear
description: Overview of Valorem Clear, a clearing and settlement primitive for DeFi options.
---

[Valorem](https://valorem.xyz/) Clear is a DeFi money lego, enabling writing covered calls and cash-secured puts, physically
settled or cash settled, American, European, and Exotic options.

## Protocol introduction

Valorem Clear V1 consists of a clearing and settling engine which allows users to write options,
exercise options, and redeem claims on assets, while handling fair assignment of exercises to claims written. It is
designed to be gas efficient, minimal, and provide a secure settlement layer upon which more complex systems can be
built.

Clear follows the [ERC-1155 multi-token](https://eips.ethereum.org/EIPS/eip-1155) standard. Options can be
written for any pair of valid ERC-20 assets (excluding rebasing, fee-on-transfer, and ERC-777 tokens). When written, an
option contract is represented by semi-fungible Option tokens, which can be bought/sold/transferred between addresses
like any ERC-1155 token.

An option writer's claim to the underlying asset(s) (if not exercised) and exercise asset(s) (if exercised) is
represented by a non-fungible Claim token. This Claim NFT can be redeemed for their share of the underlying plus
exercise assets, based on currently exercised.

The structure of an option is as follows:

- **Underlying asset:** the ERC-20 address of the asset to be received upon exercise.
- **Underlying amount:** the amount of the underlying asset contained within an option of this type.
- **Exercise asset:** the ERC-20 address of the asset needed for exercise.
- **Exercise amount:** the amount of the exercise asset required to exercise this option.
- **Exercise timestamp:** the timestamp after which this option can be exercised and physically settled.
- **Expiry timestamp:** the timestamp before which this option can be exercised.

Clear is unopinionated on the type of option (call vs. put), where, when, or for how much an option is
bought/sold, and whether or not the option is profitable when exercised. Because all options written with Valorem are
fully collateralized, physical settlement at exercise or redeem time is instant and gas-efficient.

## Trust model

### Actors

There are 3 main actors in the core protocol:

- Protocol Admin
- Option Writer
- Option Holder

The **Protocol Admin** is the address to which protocol fees are swept. This address can update the Protocol Admin
address, update the contract address of the TokenURIGenerator, enable/disable protocol fees, and sweep accrued fees. No
other permissioned actions are possible.

**Option Writers** can create new option types and write options for any valid ERC-20 asset pair (excluding rebasing,
fee-on-transfer, and ERC-777 tokens). Sufficient approval must be granted for Clear to transfer in the
requisite amount of the underlying asset. Once an option expires, the writer can redeem their Claim NFT for their share
of the underlying and/or exercise assets.

**Option Holders** can transfer and exercise options that have been written. This is accomplished via a standard
ERC-1155 transfer of the desired amount of option contracts from writer to holder. When exercising an option, they must
have enough of the exercise asset, and similarly to when writing an option, they must have granted sufficient approval
for Clear on the ERC20 exercise asset.

### Assets

When an option is written, Clear transfers in the requisite amount of the underlying asset. When an option
is exercised, Clear transfers in the requisite amount of the exerise asset and transfers out the underlying
asset. When a claim is redeemed, Clear transfers out that claimant's share of the exercise asset and any
remaining, unexercised underlying asset.

### Actions

What can each actor do, when and with how much, of each asset?

- `ERC1155`
    - `anyone` can
        - check token balances of any address
        - check ownership of any token
        - check transfer approvals
        - check supported interfaces (ERC165 and ERC1155)
        - render the URI of any token
- `ValoremOptionsClearinghouse`
    - `anyone` can
        - get info for any Option fungible token ID
        - get info for any Claim non-fungible token ID
        - check position of the underlying and exercise assets of any token
        - check whether any token ID is a fungible Option or a non-fungible Claim
        - view the contract address of the TokenURIGenerator
        - check fee balance of any ERC20 asset
        - view protocol fee (expressed in basis points)
        - check if protocol fees are enabled
        - view Protocol Admin ("feeTo") address
        - create an Option Type which does not exist yet
        - write a new Claim for a given Option Type
    - `ERC1155 Claim NFT holders` (i.e., Option Writers) can
        - write new options to a given Claim which they hold, before the expiry timestamp
        - redeem a Claim NFT which they hold for their share of the underlying and/or exercise assets, on or after the
          expiry timestamp
    - `ERC1155 Option fungible token holders` (i.e., Option Holders) can
        - transfer (up to their amount held) Option fungible tokens to another address
        - exercise (up to their amount held) Option fungible tokens, on or after the earliest exercise timestamp, and
          before the expiry timestamp
    - `Protocol Admin` can
        - enable/disable protocol fees
        - nominate a new Protocol Admin ("feeTo") address
        - accept a new Protocol Admin ("feeTo") address
        - update the contract address of the TokenURIGenerator
        - sweep accrued fees for any ERC20 asset to the Protocol Admin ("feeTo") address

## Security info

Valorem Clear v1.0.0 contains one unresolved medium severity audit finding.

The probability of an options bucket being chosen for exercise is not correlated
with the number of options contained in that bucket. Thus, individual option
contracts in smaller buckets have a higher probability of being selected for
assignment.

This probabilistic imperfection will be remediated in a future version. Discrete
probability mass functions with dynamic weighting are an unsolved issue in
solidity. Work on the remediation is underway with [LibDDRV](https://github.com/valorem-labs-inc/LibDDRV).

- Audits: [PDF reports](https://github.com/valorem-labs-inc/valorem-core/tree/master/audits)
- Security contact: info(at)valorem(dot)xyz
