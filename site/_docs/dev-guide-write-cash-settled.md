---
date: 2023-06-15 00:00:00
title: How to write cash settled options
description: Developer guide for writing cash settled options on digital assets with Valorem Clear.
---

Picking up from where we left off in the previous guide to writing [physically settled options](/docs/dev-guide-write-asset-settled), let's learn how to achieve cash settlement with Valorem Clear by integrating with Uniswap v3.

We'll write a call on LINK as the underlying asset. As of 15 June 2023, LINK is trading at $5.2279, so we'll write an [ITM strike](https://www.investopedia.com/terms/i/inthemoney.asp) of $5 to make the demonstration clear.

```solidity
// Create a new option type for a LINK-USDC call option.
vm.startPrank(ALICE);
uint256 optionId = clearinghouse.newOptionType({
    underlyingAsset: LINK,
    underlyingAmount: 1e18,
    exerciseAsset: USDC,
    exerciseAmount: 5e6,
    exerciseTimestamp: earliestExercise,
    expiryTimestamp: earliestExercise + 1 weeks
});

// Write 10 options on a new claim.
uint256 claimId = clearinghouse.write(optionId, 10);

// Transfer 10 options to Bob.
clearinghouse.safeTransferFrom(ALICE, BOB, optionId, 5, "");
vm.stopPrank();
```

## Exercising a long position with cash settlement

Directionally, here's how we'll use Uniswap v3 to achieve cash settlement of a Valorem Clear long position:
1. Initiate a LINK for WETH swap,
2. Swap all WETH for USDC,
3. Exercise our calls on the Valorem Clear clearinghouse,
4. Pay back the initial swap with the LINK received from exercise,
5. And finally pay out Bob with all the remaining USDC.

Because we don't know the premium which Bob paid for these 10 options, and because we need to support price impact protection / slippage tolerance, Bob will need to specify 2 parameters in addition to how many options to exercise. The first is the amountOut of WETH in order to cover the exercise cost, the clearinghouse fee, the 2 swap fees, with a buffer for slippage tolerance for the 2 pools. The second additional parameter is the minimum surplus of USDC he should receive — otherwise the transaction will revert, because Bob would't receive sufficient USDC to cover his premium paid and be truly in the money.

The reason we need to use 2 pools is that LINK is not LPed in the LINK-USDC pair with meaningful liquidity on Arbitrum. For WETH options, the process is simpler and we can just swap once through the WETH-USDC pool.

Also, because we need to initiate swap 1 as a pseudo flash swap and execute more transactions in a callback, we can't use ISwapRouter and instead have to call swap directly on the pool. Note we  don't use Uniswap v3 flash method, because we can't repay the borrowed asset with the same asset. Otherwise we would be attempting to swap through the same pool twice and hit a reentrancy lock.

## Calculating the amount of WETH to request

TODO

```solidity
// TODO
```

## Initiating the first swap

TODO

```solidity
// TODO
```

## Initiating the second swap

TODO

```solidity
// TODO
```

## Exercising the calls on the clearinghouse

TODO

```solidity
// TODO
```

## Paying back the first swap and paying out Bob

TODO

```solidity
// TODO
```

## Putting it all together

Alright, let's put it all together to see how the process works e2e.

```solidity
// TODO

// Warp forward to the exercise timestamp.
vm.warp(earliestExercise);

// Bob exercises his options.
vm.prank(BOB_ADDRESS);
clearinghouse.exercise(optionId, 10);

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

## Conclusion

There we have it, a cash settled American option using the Valorem Clear clearinghouse. We picked up from the [physical settlement developer guide](/docs/dev-guide-write-cash-settled/) and learned how to exercise a long position using Uniswap v3 swaps.

A full working code example is coming soon, with a working e2e demonstrations for exercising both calls and puts.

Please get in touch on our [Discord server](https://discord.gg/5jZdPuY9kR) if you have any questions or feedback. We are always looking for ways to improve our documentation and tutorials. Good luck building!
