---
date: 2022-11-30 00:00:00 +02
title: Options Smart Contracts 
description: Developer documentation for the Valorem Options smart contracts and related interfaces.
---

[Valorem Options](/docs/options-litepaper/#mechanism) is implemented as a set 
of persistent, non-upgradable smart contracts. The contracts are designed to 
prioritize censorship resistance, security, self-custody, and to function 
without any intermediaries who may restrict access. The source code may be found 
[here](https://github.com/valorem-labs-inc/valorem-core).

## OptionSettlementEngine

The `OptionSettlementEngine` implements the core underwriting and settlement 
of Valorem Options. It implements  the `IOptionSettlementEngine` and 
`IERC1155MetadataURI` interfaces.

## IOptionSettlementEngine

### Structs
#### Option
Data comprising the unique tuple of an option type associated with an ERC-1155 option token.

```solidity
struct Option {
    address underlyingAsset;
    uint96 underlyingAmount;
    address exerciseAsset;
    uint96 exerciseAmount;
    uint40 exerciseTimestamp;
    uint40 expiryTimestamp;
    uint160 settlementSeed;
    uint96 nextClaimKey;
}
```

#### Claim
Data about a claim to a short position written on an option type.
When writing an amount of options of a particular type, the writer will be issued an ERC 1155 NFT
that represents a claim to the underlying and exercise assets, to be claimed after
expiry of the option. The amount of each (underlying asset and exercise asset) paid to the claimant upon
redeeming their claim NFT depends on the option type, the amount of options written, represented in this struct,
and what portion of this claim was assigned exercise, if any, before expiry.

```solidity
struct OptionLotClaim {
    uint256 amountWritten;
    uint256 optionId;
}
```

#### Position
Data about the ERC20 assets and liabilities for a given option (long) or claim (short) token,
in terms of the underlying and exercise ERC20 tokens.

```solidity
struct Underlying {
    address underlyingAsset;
    int256 underlyingAmount;
    address exerciseAsset;
    int256 exerciseAmount;
}
```

### Enums
#### TokenType
The type of an ERC1155 subtoken in the engine.

```solidity
enum Type {
    None,
    Option,
    Claim
}
```

### Events
#### ClaimRedeemed
Emitted when a claim is redeemed.

```solidity
event ClaimRedeemed(
    uint256 indexed claimId,
    uint256 indexed optionId,
    address indexed redeemer,
    uint256 exerciseAmountRedeemed,
    uint256 underlyingAmountRedeemed
);
```

#### NewOptionType
Emitted when a new option type is created.

```solidity
event NewOptionType(
    uint256 optionId,
    address indexed exerciseAsset,
    address indexed underlyingAsset,
    uint96 exerciseAmount,
    uint96 underlyingAmount,
    uint40 exerciseTimestamp,
    uint40 indexed expiryTimestamp
);
```

#### OptionsWritten
Emitted when new options contract(s) are written.

```solidity
event OptionsWritten(uint256 indexed optionId, address indexed writer, uint256 indexed claimId, uint112 amount);
```

#### OptionsExercised
Emitted when option contract(s) are exercised.

```solidity
event OptionsExercised(uint256 indexed optionId, address indexed exerciser, uint112 amount);
```

#### FeeAccrued
Emitted when protocol fees are accrued for a given asset.
Dev note -- Emitted on write() when fees are accrued on the underlying asset,
or exercise() when fees are accrued on the exercise asset.
Will not be emitted when feesEnabled is false.

```solidity
event FeeAccrued(uint256 indexed optionId, address indexed asset, address indexed payer, uint256 amount);
```

#### FeeSwept
Emitted when accrued protocol fees for a given ERC20 asset are swept to the feeTo address.

```solidity
event FeeSwept(address indexed asset, address indexed feeTo, uint256 amount);
```

#### FeeSwitchUpdated
Emitted when protocol fees are enabled or disabled.

```solidity
event FeeSwitchUpdated(address feeTo, bool enabled);
```

#### FeeToUpdated
Emitted when feeTo address is updated.

```solidity
event FeeToUpdated(address indexed newFeeTo);
```

#### TokenURIGeneratorUpdated
Emitted when TokenURIGenerator is updated.

```solidity
event TokenURIGeneratorUpdated(address indexed newTokenURIGenerator);
```

### Errors
#### AccessControlViolation
The caller doesn't have permission to access that function.

```solidity
error AccessControlViolation(address accessor, address permissioned);
```

#### AmountWrittenCannotBeZero
The amount of options contracts written must be greater than zero.

```solidity
error AmountWrittenCannotBeZero();
```

#### CallerDoesNotOwnClaimId
This claim is not owned by the caller.

```solidity
error CallerDoesNotOwnClaimId(uint256 claimId);
```

#### CallerHoldsInsufficientOptions
The caller does not have enough options contracts to exercise the amount specified.

```solidity
error CallerHoldsInsufficientOptions(uint256 optionId, uint112 amount);
```

#### ClaimTooSoon
Claims cannot be redeemed before expiry.

```solidity
error ClaimTooSoon(uint256 claimId, uint40 expiry);
```

#### ExerciseTooEarly
This option cannot yet be exercised.

```solidity
error ExerciseTooEarly(uint256 optionId, uint40 exercise);
```

#### ExerciseWindowTooShort
The option exercise window is too short.

```solidity
error ExerciseWindowTooShort(uint40 exercise);
```

#### ExpiredOption
The optionId specified expired has already expired.

```solidity
error ExpiredOption(uint256 optionId, uint40 expiry);
```

#### ExpiryWindowTooShort
The expiry timestamp is too soon.

```solidity
error ExpiryWindowTooShort(uint40 expiry);
```

#### InvalidAddress
Invalid (zero) address.

```solidity
error InvalidAddress(address input);
```

#### InvalidAssets
The assets specified are invalid or duplicate.

```solidity
error InvalidAssets(address asset1, address asset2);
```

#### InvalidClaim
The token specified is not a claim token.

```solidity
error InvalidClaim(uint256 token);
```

#### InvalidOption
The token specified is not an option token.

```solidity
error InvalidOption(uint256 token);
```

#### OptionsTypeExists
This option contract type already exists and thus cannot be created.

```solidity
error OptionsTypeExists(uint256 optionId);
```

#### TokenNotFound
The token specified is not found.

```solidity
error TokenNotFound(uint256 token);
```

### Views
#### option
Gets information about an option.

```solidity
function option(uint256 tokenId) external view returns (Option memory optionInfo);
```

#### claim
Gets information about a claim.

```solidity
function claim(uint256 claimId) external view returns (Claim memory claimInfo);
```

#### position
Gets information about the ERC20 token positions of an option or claim.

```solidity
function position(uint256 tokenId) external view returns (Position memory positionInfo);
```

#### tokenType
Gets the TokenType for a given tokenId.

```solidity
function tokenType(uint256 tokenId) external view returns (TokenType typeOfToken);
```

#### tokenURIGenerator
Gets the address of the URI generator contract.

```solidity
function tokenURIGenerator() external view returns (ITokenURIGenerator uriGenerator);
```

#### feeBalance
Gets the balance of protocol fees for a given token which have not been swept yet.

```solidity
function feeBalance(address token) external view returns (uint256);
```

#### feeBps
Gets the protocol fee, expressed in basis points.

```solidity
function feeBps() external view returns (uint8 fee);
```

#### feesEnabled
Checks if protocol fees are enabled.

```solidity
function feesEnabled() external view returns (bool enabled);
```

#### feeTo
Gets the address to which protocol fees are swept.

```solidity
function feeTo() external view returns (address);
```

### Write Options
#### newOptionType
Creates a new option contract type if it doesn't already exist.

```solidity
function newOptionType(
    address underlyingAsset,
    uint96 underlyingAmount,
    address exerciseAsset,
    uint96 exerciseAmount,
    uint40 exerciseTimestamp,
    uint40 expiryTimestamp
) external returns (uint256 optionId);
```

#### write
Writes a specified amount of the specified option, returning claim NFT id.

```solidity
function write(uint256 tokenId, uint112 amount) external returns (uint256 claimId);
```

### Redeem Claims
#### redeem
Redeems a claim NFT, transfers the underlying/exercise tokens to the caller.
Can be called after option expiry timestamp (inclusive).

```solidity
function redeem(uint256 claimId) external;
```

### Exercise Options
#### exercise
Exercises specified amount of optionId, transferring in the exercise asset,
and transferring out the underlying asset if requirements are met. Can be called
from exercise timestamp (inclusive), until option expiry timestamp (exclusive).

```solidity
function exercise(uint256 optionId, uint112 amount) external;
```

### Protocol Admin
#### setFeesEnabled
Enables or disables protocol fees.

```solidity
function setFeesEnabled(bool enabled) external;
```

#### setFeeTo
Nominates a new address to which fees should be swept, requiring
the new feeTo address to accept before the update is complete. See also
acceptFeeTo().

```solidity
function setFeeTo(address newFeeTo) external;
```

#### acceptFeeTo
Accepts the new feeTo address and completes the update.
See also setFeeTo(address newFeeTo).

```solidity
function acceptFeeTo() external;
```

#### setTokenURIGenerator
Updates the contract address for generating token URIs for tokens.

```solidity
function setTokenURIGenerator(address newTokenURIGenerator) external;
```

#### sweepFees
Sweeps fees to the feeTo address if there is more than 1 wei for feeBalance for a given token.

```solidity
function sweepFees(address[] memory tokens) external;
```
