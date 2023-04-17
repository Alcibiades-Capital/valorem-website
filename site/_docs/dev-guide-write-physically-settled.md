---
date: 2023-04-12 00:00:00
title: How to write physically settled options
description: Developer guide for writing physically settled options on digital assets with the Valorem Options Clearinghouse.
---

In this developer guide, we will walk through how to write physically settled options on digital assets using the Valorem Options Clearinghouse.

We are Alice and we will use Bob as a second address. We'll write call options on the WETH/LUSD pair, transfer some to Bob, Bob will exercise their options (if it's in-the-money), and finally we'll redeem our claim over the underlying/exercise assets from the Clearinghouse.

All code examples are provided in Solidity // `forge` and CLI // `cast`.

## Creating an option type

First we need to create a new option type in the Clearinghouse for our call option. This is done by invoking the [`newOptionType`](/docs/clearinghouse-contracts/#newOptionType) function on the ABI.

Here are the details of the call option we want to write:

- Underlying asset: WETH
- Underlying amount: 1
- Exercise asset: LUSD
- Exercise amount: 2100
- Exercise timestamp: Now
- Expiry timestamp: 1 week from now

Note that this call option grants the holder the right (but not the obligation) to 'buy' the volatile asset. The underlying asset is volatile, the exercise asset is stable, and the strike is implicitly the exercise amount divided by the underlying amount (2100 LUSD / 1 WETH).

For put options, it's the inverse — the underlying asset is stable, the exercise asset is volatile, and the strike is the underlying amount divided by the exercise amount (still 2100 USD / 1 WETH, but for puts the underlying and exercise assets are inverted). The holder has the option to 'sell' the volatile asset at the strike, which upon exercise is transferred in  and the underlying asset is transferred out.

The simplicity of this *Asset A In, Asset B Out* mechanism is how Valorem is able to achieve gas-efficient, oracle-free clearing and settling of options on digital assets.

{% tabs log %}

{% tab log solidity %}
```solidity
// Instantiate the IValoremOptionsClearinghouse interface.
IValoremOptionsClearinghouse clearinghouse = new IValoremOptionsClearinghouse(CLEARINGHOUSE_ADDRESS);

// Setup option parameters.
address WETH_ADDRESS = 0xTODO;
uint96 underlyingAmount = 1 ether;
address LUSD_ADDRESS = 0xTODO;
uint96 exercisePrice = 2100e18;
uint40 earliestExercise = block.timestamp;
uint40 expiry = earliestExercise + 1 weeks;

// Create a new option type and get the optionId.
uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: WETH_ADDRESS,
    underlyingAmount: underlyingAmount,
    exerciseAsset: LUSD_ADDRESS,
    exerciseAmount: exercisePrice,
    exerciseTimestamp: earliestExercise,
    expiryTimestamp: expiry
});
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "newOptionType(address,uint96,address,uint96,uint40,uin40) (uint256)" "0xTODO" 1e18 "0xTODO" 2100e18 TODONOW TODONOWPLUSONEWEEK
```
{% endtab %}

{% endtabs %}

## Writing an option

Next we will write 10 options for this option type. This is done by calling the [`write`](/docs/clearinghouse-contracts/#write) function. The Clearinghouse will transfer in the required amount of the underlying asset (10 * 1 ether WETH) and in return, we receive 10 fungible Option tokens, representing our long position, and 1 Claim NFT, representing our short position. The function returns the token ID for this Claim NFT.

Note that sufficient ERC20 approval for the underlying asset, WETH, must be granted to the Clearinghouse contract before calling this function.

{% tabs log %}

{% tab log solidity %}
```solidity
// Write the option with a specified optionId and amount.
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

Now that we have written some options, we can transfer some or all of our long position to another address using the tried and true [`safeTransferFrom`](https://eips.ethereum.org/EIPS/eip-1155#specification) function, native to the IERC1155 interface.

In this example, we transfer 4 options to Bob. So we retain 6 long Option tokens and our 1 short Claim NFT, which represents our redemption rights over the correct proportion of underlying and exercise asset remaining at expiry.

{% tabs log %}

{% tab log solidity %}
```solidity
// Define the address of the recipient, Bob.
address BOB = address(0xB0B);

// Transfer 4 options to Bob.
clearinghouse.safeTransferFrom(address(this), BOB, optionId, 4, "");
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "safeTransferFrom(address,address,uint256,bytes32TODO)" "$ALICE_ADDRESS" "$BOB_ADDRESS" $OPTION_ID 4 ""
```
{% endtab %}

{% endtabs %}

## Exercising an option

We have transferred some of our Option tokens to Bob, and as long as it is within the exercise window defined for the option type (on or after `exerciseTimestamp`, before `expiryTimestamp`), he can exercise his position.

Let's imagine for this example that the position is finishing in-the-money, and Bob wants to exercise his options. This is done by calling the [`exercise`](/docs/clearinghouse-contracts/#exercise) function. The result is that Bob's 4 Option tokens are burned, the required amount of the exercise asset (4 * 2100e18 LUSD) is transferred to the Clearinghouse from Bob, and the correct amount of the underlying asset (4 * 1 ether WETH) is transferred from the Clearinghouse to Bob.

Note again that sufficient ERC20 approval must be granted to the Clearinghouse contract before calling this function (although in this case, for the exercise asset, LUSD).

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp forward to the exercise timestamp.
vm.warp(earliestExercise);

// Bob exercises his options.
vm.prank(BOB);
clearinghouse.exercise(optionId, 4);
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$BOB_PRIVATE_KEY "exercise(uint256,uint112)" $OPTION_ID 4
```
{% endtab %}

{% endtabs %}

## Redeeming a Claim

Finally, we will redeem our Claim NFT (using the Claim token ID returned from [`write`](/docs/clearinghouse-contracts/#write)), which is done by calling the [`redeem`](/docs/clearinghouse-contracts/#redeem) function. This function transfers out either one or both of the underlying and exercise assets, depending on the exercise assignment status of the Claim:
- For unexercised positions, solely the `underlyingAsset`
- For fully exercised positions, solely the `exerciseAsset`
- For partially exercised positions, a mix of both assets in the correct proportions

In our example, TODO. And we can always use the [`claim`](/docs/clearinghouse-contracts/#claim) function to check the exercise status of our Claim NFT before redeeming.

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp to the expiry timestamp.
vm.warp(expiry);

// Redeem our claim.
clearinghouse.redeem(claimId);

// Log the amount of WETH and LUSD we received upon redemption.
TODO
uint256 lusdBalance = LUSD.balanceOf(address(this));
emit LogUint("LUSD received from swap", lusdBalance);
emit LogUint("LUSD received from claim", uint256(exerciseAmount));
emit LogUint("Total LUSD received", lusdBalance + uint256(exerciseAmount));
```
{% endtab %}

{% tab log bash %}
```bash
$ cast send $CH_ADDRESS --rpc-url=$RPC_URL --private-key=$PRIVATE_KEY "redeem(uint256)" $CLAIM_ID
```
{% endtab %}

{% endtabs %}

## Conclusion

There we have it, a physically settled American option using the Valorem Options Clearinghouse. We learned how to create a new option type and write options, transfer a long position to another address, exercise this long position, and finally redeem our claim over the collateral of the short position.

Please get in touch on our [Discord server](https://discord.gg/5jZdPuY9kR) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
