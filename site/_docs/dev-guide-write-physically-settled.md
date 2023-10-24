---
date: 2023-06-06 00:00:00
title: Writing Physically Settled ERC20 Options with Valorem Clear - A Developer Guide
description: Master the process of writing physically settled options on digital assets using Valorem Clear. Dive into step-by-step Solidity and CLI code examples covering option creation, transfer, exercise, and claim redemption.
---

Welcome to the developer guide on constructing physically settled options with Valorem Clear. Whether you're interested 
in call or put options, this comprehensive tutorial provides hands-on Solidity and CLI examples to get you started. 
From option creation, transferring positions, exercising options, to the final steps of claim redemption, we cover it 
all. Ready to leverage the power of Valorem Clear for optimized option writing? Let's dive in!

All code examples are provided in Solidity // `forge` and CLI // `cast` (more languages to come).

## Creating an option type

First we need to create a new option type in the clearinghouse for our call option. This is done by invoking the [`newOptionType`](/docs/clear-contracts/#newOptionType) function on the ABI.

Here are the details of the call option we want to write:

- Underlying asset: WETH
- Underlying amount: 1 (ie, 1e18)
- Exercise asset: USDC
- Exercise amount: 2100 (ie, 2100e6)
- Exercise timestamp: Now
- Expiry timestamp: 1 week from now

Note that this call option grants the holder the right, but not the obligation, to buy the volatile asset, in this example Wrapped Ether. The underlying asset is volatile, the exercise asset is stable, and the strike price is implicitly the exercise amount divided by the underlying amount (2100 USDC / 1 WETH = $2100).

For put options, it's the inverse — the holder has the option to sell the exercise asset (WETH) at the strike price. The underlying asset is stable, the exercise asset is volatile, and the strike is the underlying amount divided by the exercise amount (which is still 2100 USD / 1 WETH, because for puts the underlying and exercise assets are inverted in comparision with calls).

In both cases, upon exercise, the exercise asset is transferred in and the underlying asset is transferred out. The simplicity of this *Asset A In, Asset B Out* mechanism is how Valorem Clear is able to achieve gas-efficient, oracle-free clearing and settlement of option contracts.

{% tabs log %}

{% tab log solidity %}
```solidity
// Alice creates a new option type.
vm.startPrank(ALICE);

uint96 underlyingAmount = 1e18;
uint96 exerciseAmount = 2100e6;
uint40 earliestExercise = uint40(block.timestamp);
uint40 expiry = uint40(earliestExercise + 1 weeks);

uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: address(WETH),
    underlyingAmount: underlyingAmount,
    exerciseAsset: address(USDC),
    exerciseAmount: exerciseAmount,
    exerciseTimestamp: earliestExercise,
    expiryTimestamp: expiry
});
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "newOptionType(address,uint96,address,uint96,uint40,uint40) (uint256)" "$WETH" 1000000000000000000 "$USDC" 2100000000 1686115811 1686720611
```
{% endtab %}

{% endtabs %}

## Writing an option

Next we will write 10 options for this option type. This is done by calling the [`write`](/docs/clear-contracts/#write) function. The Clearinghouse will transfer in the required amount of the underlying asset (10 * 1e18 WETH) and in return, we receive 10 fungible Option tokens, representing our long position, and 1 Claim NFT, representing our short position. The function returns the token ID for this Claim NFT.

Note that sufficient ERC20 approval for the underlying asset, WETH, must be granted to the Clearinghouse contract before calling this function.

{% tabs log %}

{% tab log solidity %}
```solidity
// Alice writes 10 options.
WETH.approve(address(clearinghouse), type(uint256).max);
uint256 claimId = clearinghouse.write(optionId, 10);
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "write(uint256,uint112)" $OPTION_ID 10
```
{% endtab %}

{% endtabs %}

## Transferring a long position

Now that we have written some options, we can transfer our long (or short!) position to another address using the tried and true [`safeTransferFrom`](https://eips.ethereum.org/EIPS/eip-1155#specification) function, native to the `IERC1155` interface.

In this example, we transfer 4 options to Bob. So we retain 6 long Option tokens and our 1 short Claim NFT, which represents our redemption rights over the correct proportion of underlying and exercise asset remaining at expiry.

{% tabs log %}

{% tab log solidity %}
```solidity
// Alice transfers 4 options to Bob.
clearinghouse.safeTransferFrom(ALICE, BOB, optionId, 4, "");
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "safeTransferFrom(address,address,uint256,uint256,bytes)" "$ALICE" "$BOB" $OPTION_ID 4 ""
```
{% endtab %}

{% endtabs %}

## Exercising an option

We have transferred some of our Option tokens to Bob, and as long as it is within the exercise window defined for this option type (on or after `exerciseTimestamp`, before `expiryTimestamp`), he can exercise his some or all of his position.

Let's imagine for this example that the position is in-the-money, and Bob wants to exercise his options. This is done by calling the [`exercise`](/docs/clear-contracts/#exercise) function. The result is that Bob's 4 Option tokens are burned, the required amount of the exercise asset (4 * 2100e6 USDC) is transferred in to the Clearinghouse from Bob, and the correct amount of the underlying asset (4 * 1e18 WETH) is transferred out from the Clearinghouse to Bob.

Note again that sufficient ERC20 approval must be granted by Bob to the Clearinghouse contract before calling this function (although in this case for USDC, the exercise asset).

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp to the exercise timestamp.
vm.warp(earliestExercise);

// Bob exercises his options.
vm.startPrank(BOB);
USDC.approve(address(clearinghouse), type(uint256).max);
clearinghouse.exercise(optionId, 4);
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$BOB_PRIVATE_KEY "exercise(uint256,uint112)" $OPTION_ID 4
```
{% endtab %}

{% endtabs %}

## Redeeming a claim

Finally, we will redeem our Claim NFT (using the Claim token ID returned from [`write`](/docs/clear-contracts/#write)), which is achieved by calling the [`redeem`](/docs/clear-contracts/#redeem) function. This function transfers out either one or both of the underlying and exercise assets, depending on the exercise assignment status of the Claim:
- For unexercised positions, solely the `underlyingAsset`
- For fully exercised positions, solely the `exerciseAsset`
- For partially exercised positions, a mix of both assets in the correct proportions

In our example, we will redeem the claim for the WETH and USDC remaining in the position. And we can always use the [`claim`](/docs/clear-contracts/#claim) function to check the exercise status of our Claim NFT before redeeming.

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp to the expiry timestamp.
vm.warp(expiry);

// Check out claim position
(, int256 underlyingAmount,, int256 exerciseAmount) = clearinghouse.position(claimId);

// Alice redeems her claim.
clearinghouse.redeem(claimId);
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "redeem(uint256)" $CLAIM_ID
```
{% endtab %}

{% endtabs %}

## Conclusion

There we have it, a physically settled American option using the Valorem Clear clearinghouse. We learned how to create a new option type and write options, transfer a long position to another address, exercise this long position, and finally redeem our claim over the collateral backing the short position.

A full working code example is available [here](https://github.com/valorem-labs-inc/valorem-dev-guides/blob/main/test/ClearPhysicallySettled.t.sol), complete with balance assertions that demonstrate the movements of the underlying and option tokens.

Please get in touch on our [Discord server](https://discord.gg/valorem) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
