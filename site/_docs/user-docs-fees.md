---
date: 2023-09-06 00:00:00
title: Fees
description: User guide describing the Valorem Clear and Trade fee model.
---

Valorem has a maker-taker fee model for relaying trades, meaning actions which provide liquidity may have different fees assessed than actions that take liquidity. Likewise for clearing and settlement — different fees may be assessed for option writers and option holders.

### Trade Relay Fees

Taker fees: 1.25% of option premium

`option_premium * quantity * 0.0125`

Maker fees: None currently

### Clearing and Settlement Fees

Write fee: 0.15% of optional notional (assessed in the underlying asset)

`option_notional * quantity * 0.0015`

Exercise fee: 0.15% of option notional (assessed in the exercise asset)

`option_notional * quantity * 0.0015`

### Example Scenario

- Symbol: ETH-8SEP23-1650-C
- Type: Call
- Underlying Asset: WETH
- Exercise Asset: USDC
- Expiry: 2023-09-08T08:00:00+0000
- Strike: $1,650
- Exercise Style: American
- Quantity: 2
- Premium: $5

Alice (taker) requests a quote for 2 ETH-8SEP23-1650-C options. Bob (maker) responds with a quote of 10 USDC, and Alice executes the order. The system transfers 2 Valorem options from Bob to Alice, 9.875 USDC from Alice to Bob, and 0.125 USDC from Alice to Valorem 'feeTo' address.

`5 USDC * 2 options * 1.0125 (trade relay fee)`

When Bob initially wrote the 2 options, 2.003 WETH were transferred from Bob to Valorem Clear (a write fee assessed of 0.003 WETH).

`2 WETH * 2 options * 1.0015 (write fee)`

If Alice were to exercise the 2 options, 3304.95 USDC would be transferred from Alice to Valorem Clear (an exercise fee assessed of 4.95 USDC), and 2 WETH would be transferred from Valorem Clear to Alice.

`1,650 USDC * 2 options * 1.0015 (exercise fee)`
