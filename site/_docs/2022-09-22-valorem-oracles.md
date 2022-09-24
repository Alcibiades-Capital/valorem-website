---
date: 2022-09-22 00:00:00 +01
title: Oracles 
description: The Valorem oracle contracts provide on chain models and views for Valorem options.  
---

The Valorem oracle smart contracts provide token price, volatility, yield and 
higher level models useful for modeling and valuing Valorem options.

[Oracle Contracts Source Code](https://github.com/valorem-labs-inc/valorem-oracles)

## Oracle Interfaces

The oracle smart contracts are designed to support multiple data sources over 
time, with a common interface to interact with higher level models.

### Price Oracle Interface

`IPriceOracle` is an interface for contracts providing the price of a token, in 
USD. An external price oracle can be used seamlessly by being wrapped in a 
contract implementing this interface.

#### Functions

##### getPriceUSD

Returns the price of `IERC20 token`, an ERC20 token contract, in USD, or 
reverts `PriceNotAvailable()`.

```solidity
function getPriceUSD(IERC20 token) external view returns (uint256 price);
```

##### scale

Returns the scaling factor for `getPriceUSD` as a `uint8` representing the
power of 10 by which the `uint256 price` is scaled.

```solidity
function scale() external view returns (uint8 scale);
```

#### Errors

##### PriceNotAvailable

`IPriceOracle` implementations revert with this error when they cannont derive
the USD price for a given `IERC20 token`.

```solidity
error PriceNotAvailable();
```