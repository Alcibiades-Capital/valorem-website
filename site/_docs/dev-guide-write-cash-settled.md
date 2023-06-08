---
date: 2023-04-12 00:00:00
title: How to write cash settled options
description: Developer guide for writing cash settled options on digital assets with Valorem Clear.
---

Picking up from where we left off in the previous guide to writing [physically settled options](/docs/dev-guide-write-asset-settled), let's now learn how to achieve cash settlement of your Valorem Clear positions.

## Writing an option 

{% tabs log %}

{% tab log solidity %}
```solidity
// Create a new option type for a MAGIC-USDC call option.
uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: MAGIC,
    underlyingAmount: 1e18,
    exerciseAsset: USDC,
    exerciseAmount: 1.20e6,
    exerciseTimestamp: exerciseTimestamp,
    expiryTimestamp: exerciseTimestamp + 1 weeks
});

// Write 10 options on a new claim.
uint256 claimId = clearinghouse.write(optionId, 10);

// Transfer 5 options to Bob.
address BOB = address(0xB0B);
clearinghouse.safeTransferFrom(address(this), BOB, optionId, 5, "");
```
{% endtab %}

{% tab log bash %}
```bash
$ cast TODO
```
{% endtab %}

{% endtabs %}

## Exercising a long position with cash settlement

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp forward to the exercise timestamp.
vm.warp(earliestExercise);

// Bob exercises his options.
vm.prank(BOB_ADDRESS);
clearinghouse.exercise(optionId, 5);

// Bob swaps his MAGIC for USDC via Uniswap v3.
uint256 bobMagicBalance = MAGIC.balanceOf(address(this));
ISwapRouter.ExactInputParams memory params =
ISwapRouter.ExactInputParams({
    path: abi.encodePacked(MAGIC, 300, WETH, 50, USDC),
    recipient: msg.sender,
    deadline: block.timestamp,
    amountIn: bobMagicBalance,
    amountOutMinimum: 0 // of course you'll want to set price impact / slippage tolerance
});
amountOut = swapRouter.exactInput(params);

// Log the amount of USDC we received for cash settled exercise.
uint256 bobUSDCBalance = USDC.balanceOf(address(this));
```
{% endtab %}

{% tab log bash %}
```bash
$ cast TODO
```
{% endtab %}

{% endtabs %}

## Redeeming a short position with cash settlement

{% tabs log %}

{% tab log solidity %}
```solidity
// Warp to the expiry timestamp.
vm.warp(expiry);

// Check the position of our claim -- 5 options should be exercised, totaling
// 5 * 1e18 MAGIC and 1.5e18 USDC should be available to redeem from the claim.
(, int256 underlyingAmount,, int256 exerciseAmount) = clearinghouse.position(claimId);

// Redeem our claim.
vm.prank(ALICE_ADDRESS);
clearinghouse.redeem(claimId);

// Swap any remaining MAGIC for USDC via Uniswap.
uint256 magicBalance = MAGIC.balanceOf(address(this));
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
    tokenIn: address(MAGIC),
    tokenOut: address(USDC),
    fee: 3000,
    recipient: address(this),
    deadline: block.timestamp + 1,
    amountIn: magicBalance,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0
});

// Log the amount of USDC we received for cash settled writing.
uint256 USDCBalance = USDC.balanceOf(address(this));
emit log_named_uint("USDC received from swap", USDCBalance);
emit log_named_uint("USDC received from claim", uint256(exerciseAmount));
emit log_named_uint("Total USDC received", USDCBalance + uint256(exerciseAmount));
```
{% endtab %}

{% tab log bash %}
```bash
$ cast TODO
```
{% endtab %}

{% endtabs %}

## Conclusion

There we have it, a cash settled American option using the Valorem Clear clearinghouse. We picked up from the [physical settlement developer guide](/docs/dev-guide-write-cash-settled/) and learned how to exercise a long position using a flash swap.

Please get in touch on our [Discord server](https://discord.gg/5jZdPuY9kR) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
