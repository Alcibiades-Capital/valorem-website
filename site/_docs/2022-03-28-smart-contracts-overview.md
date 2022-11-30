---
date: 2022-11-30 00:00:00 +02
title: Smart Contracts Overview
description: An overview of the Valorem protocol smart contracts.
---

The Valorem Options V1 Core consists of a settlement engine which allows users to write options, exercise options, and redeem claims on assets, while handling fair assignment of exercises to claims written. It is designed to be gas efficient, minimal, and provide a secure settlement layer upon which more complex systems can be built.

The Settlement Engine follows the [ERC-1155 multi-token](https://eips.ethereum.org/EIPS/eip-1155) standard. Options can be written for any pair of valid ERC-20 assets (excluding rebasing, fee-on-transfer, and ERC-777 tokens). When written, an options contract is represented by semi-fungible Option tokens, which can be bought/sold/transferred between addresses like any ERC-1155 token.

An option writer's claim to the underlying asset(s) (if not exercised) and exercise asset(s) (if exercised) is represented by a non-fungible option lot Claim token. This Claim NFT can be redeemed for their share of the underlying plus exercise assets, based on currently exercised.

The structure of an option is as follows:

- **Underlying asset:** the ERC-20 address of the asset to be received upon exercise.
- **Underlying amount:** the amount of the underlying asset contained within an option of this type.
- **Exercise asset:** the ERC-20 address of the asset needed for exercise.
- **Exercise amount:** the amount of the exercise asset required to exercise this option.
- **Exercise timestamp:** the timestamp after which this option can be exercised and physically settled.
- **Expiry timestamp:** the timestamp before which this option can be exercised.

The Core is unopinionated on the type of option (call vs. put), where, when, or for how much an option is bought/sold, and whether or not the option is profitable when exercised. Because all options written with Valorem are fully collateralized, physical settlement at exercise or redeem time is instant and gas-efficient.

[Core Contracts Source Code](https://github.com/valorem-labs-inc/valorem-core)

## Core Interface

The core exposes an interface for users of the protocol, which is documented here and in the codebase with NatSpec.

### IOptionSettlementEngine

#### Structs
##### Option
This struct contains the data about an options type associated with an ERC-1155 token.

```solidity
struct Option {
    /// @param underlyingAsset The underlying asset to be received
    address underlyingAsset;
    /// @param underlyingAmount The amount of the underlying asset contained within an option contract of this type
    uint96 underlyingAmount;
    /// @param exerciseAsset The address of the asset needed for exercise
    address exerciseAsset;
    /// @param exerciseAmount The amount of the exercise asset required to exercise this option
    uint96 exerciseAmount;
    /// @param exerciseTimestamp The timestamp after which this option can be exercised
    uint40 exerciseTimestamp;
    /// @param expiryTimestamp The timestamp before which this option can be exercised
    uint40 expiryTimestamp;
    /// @param settlementSeed Random seed created at the time of option type creation
    uint160 settlementSeed;
    /// @param nextClaimNum Which option was written
    uint96 nextClaimNum;
}
```

##### OptionLotClaim
This struct contains the data about a lot of options written for a particular option type. When writing an amount of options of a particular type, the writer will be issued an ERC 1155 NFT that represents a claim to the underlying and exercise assets of the options lot, to be claimed after expiry of the option. The amount of each (underlying asset and exercise asset) paid to the claimant upon redeeming their claim NFT depends on the option type, the amount of options written in their options lot (represented in this struct) and what portion of their lot was exercised before expiry.

```solidity
struct OptionLotClaim {
    /// @param amountWritten The number of options written in this option lot claim
    uint112 amountWritten;
    /// @param claimed Whether or not this option lot has been claimed by the writer
    bool claimed;
}
```

##### Underlying
Struct used in returning data regarding positions underlying a claim or option.

```solidity
struct Underlying {
    /// @param underlyingAsset address of the underlying asset erc20
    address underlyingAsset;
    /// @param underlyingPosition position on the underlying asset
    int256 underlyingPosition;
    /// @param exerciseAsset address of the exercise asset erc20
    address exerciseAsset;
    /// @param exercisePosition position on the exercise asset
    int256 exercisePosition;
}
```

#### Enums
##### Type
This enumeration is used to determine the type of an ERC1155 subtoken in the engine.

```solidity
enum Type {
    None,
    Option,
    OptionLotClaim
}
```

#### Events
##### FeeSwept
Emitted when accrued protocol fees for a given token are swept to the `feeTo` address.

```solidity
event FeeSwept(address indexed token, address indexed feeTo, uint256 amount);
```

##### NewOptionType
Emitted when a new unique options type is created.

```solidity
event NewOptionType(
    uint256 indexed optionId,
    address indexed exerciseAsset,
    address indexed underlyingAsset,
    uint96 exerciseAmount,
    uint96 underlyingAmount,
    uint40 exerciseTimestamp,
    uint40 expiryTimestamp,
    uint96 nextClaimNum
);
```

##### OptionsExercised
Emitted when an option is exercised.

```solidity
event OptionsExercised(uint256 indexed optionId, address indexed exercisee, uint112 amount);
```

##### OptionsWritten
Emitted when a new option is written.

```solidity
event OptionsWritten(uint256 indexed optionId, address indexed writer, uint256 indexed claimId, uint112 amount);
```

##### FeeAccrued
Emitted when protocol fees are accrued for a given asset.

```solidity
event FeeAccrued(address indexed asset, address indexed payor, uint256 amount);
```

##### ClaimRedeemed
Emitted when a claim is redeemed.

```solidity
event ClaimRedeemed(
    uint256 indexed claimId,
    uint256 indexed optionId,
    address indexed redeemer,
    address exerciseAsset,
    address underlyingAsset,
    uint96 exerciseAmount,
    uint96 underlyingAmount
);
```

##### ExerciseAssigned
Emitted when an option id is exercised and assigned to a particular claim NFT.

```solidity
event ExerciseAssigned(uint256 indexed claimId, uint256 indexed optionId, uint112 amountAssigned);
```

#### Errors
##### TokenNotFound
The requested token is not found.

```solidity
error TokenNotFound(uint256 token);
```

##### AccessControlViolation
The caller doesn't have permission to access that function.

```solidity
error AccessControlViolation(address accessor, address permissioned);
```

##### InvalidFeeToAddress
Invalid fee to address.

```solidity
error InvalidFeeToAddress(address feeTo);
```

##### InvalidTokenURIGeneratorAddress
Invalid TokenURIGenerator address.

```solidity
error InvalidTokenURIGeneratorAddress(address tokenURIGenerator);
```

##### OptionsTypeExists
This options chain already exists and thus cannot be created.

```solidity
error OptionsTypeExists(uint256 optionId);
```

##### ExpiryWindowTooShort
The expiry timestamp is less than 24 hours from now.

```solidity
error ExpiryWindowTooShort(uint40 expiry);
```

##### ExerciseWindowTooShort
The option exercise window is less than 24 hours long.

```solidity
error ExerciseWindowTooShort(uint40 exercise);
```

##### InvalidAssets
The assets specified are invalid or duplicate.

```solidity
error InvalidAssets(address asset1, address asset2);
```

##### InvalidOption
The token specified is not an option.

```solidity
error InvalidOption(uint256 token);
```

##### InvalidClaim
The token specified is not a claim.

```solidity
error InvalidClaim(uint256 token);
```

##### EncodedOptionIdInClaimIdDoesNotMatchProvidedOptionId
Provided claimId does not match provided option id in the upper 160b encoding the corresponding option ID for which the claim was written.

```solidity
error EncodedOptionIdInClaimIdDoesNotMatchProvidedOptionId(uint256 claimId, uint256 optionId);
```

##### ExpiredOption
The optionId specified expired at expiry.

```solidity
error ExpiredOption(uint256 optionId, uint40 expiry);
```

##### ExerciseTooEarly
This option cannot yet be exercised.

```solidity
error ExerciseTooEarly(uint256 optionId, uint40 exercise);
```

##### CallerDoesNotOwnClaimId
This claim is not owned by the caller.

```solidity
error CallerDoesNotOwnClaimId(uint256 claimId);
```

##### CallerHoldsInsufficientOptions
The caller does not have enough of the option to exercise the amount specified.

```solidity
error CallerHoldsInsufficientOptions(uint256 optionId, uint112 amount);
```

##### ClaimTooSoon
You can't claim before expiry.

```solidity
error ClaimTooSoon(uint256 claimId, uint40 expiry);
```

##### AmountWrittenCannotBeZero
The amount provided to write() must be > 0.

```solidity
error AmountWrittenCannotBeZero();
```

#### Functions - Accessors
##### option
Returns Option struct details about a given tokenID if that token is an option.

```solidity
function option(uint256 tokenId) external view returns (Option memory optionInfo);
```

##### claim
Returns OptionLotClaim struct details about a given tokenId if that token is a claim NFT.

```solidity
function claim(uint256 tokenId) external view returns (OptionLotClaim memory claimInfo);
```

##### underlying
Information about the position underlying a token, useful for determining value. When supplied an Option Lot Claim id, this function returns the total amounts of underlying and exercise assets currently associated with a given options lot.

```solidity
function underlying(uint256 tokenId) external view returns (Underlying memory underlyingPositions);
```

##### tokenType
Returns the token type (e.g. Option/OptionLotClaim) for a given token id.

```solidity
function tokenType(uint256 tokenId) external view returns (Type);
```

##### isOptionInitialized
Check to see if an option is already initialized.

```solidity
function isOptionInitialized(uint160 optionKey) external view returns (bool);
```


#### Functions - Token ID Encoding
##### encodeTokenId
Encode the supplied option id and claim id. Option and claim token ids are encoded as follows:
    MSb
    0000 0000   0000 0000   0000 0000   0000 0000 ┐
    0000 0000   0000 0000   0000 0000   0000 0000 │
    0000 0000   0000 0000   0000 0000   0000 0000 │ 160b option key, created from hash of Option struct
    0000 0000   0000 0000   0000 0000   0000 0000 │
    0000 0000   0000 0000   0000 0000   0000 0000 │
    0000 0000   0000 0000   0000 0000   0000 0000 ┘
    0000 0000   0000 0000   0000 0000   0000 0000 ┐
    0000 0000   0000 0000   0000 0000   0000 0000 │ 96b auto-incrementing option lot claim number
    0000 0000   0000 0000   0000 0000   0000 0000 ┘
                                              LSb

```solidity
function encodeTokenId(uint160 optionKey, uint96 claimNum) external pure returns (uint256 tokenId);
```

##### decodeTokenId
Decode the supplied token id.

```solidity
function decodeTokenId(uint256 tokenId) external pure returns (uint160 optionKey, uint96 claimNum);
```

#### Functions - Write Options
##### newOptionType
Create a new option type if it doesn't already exist.

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

##### write new
Writes a specified amount of the specified option, returning claim NFT id.

```solidity
function write(uint256 optionId, uint112 amount) external returns (uint256 claimId);
```

##### write existing
This override allows additional options to be written against a particular claim id.

```solidity
function write(uint256 optionId, uint112 amount, uint256 claimId) external returns (uint256);
```

#### Functions - Exercise Options
##### exercise
Exercises specified amount of optionId, transferring in the exercise asset, and transferring out the underlying asset if requirements are met. Will revert with an underflow/overflow if the user does not have the required assets.

```solidity
function exercise(uint256 optionId, uint112 amount) external;
```

#### Functions - Redeem Claims
##### redeem
Redeem a claim NFT, transfers the underlying tokens.

```solidity
function redeem(uint256 claimId) external;
```

#### Functions - Protocol Admin
##### feeBps
The protocol fee, expressed in basis points.

```solidity
function feeBps() external view returns (uint8);
```

##### feeBalance
The balance of protocol fees for a given token which have not yet been swept.

```solidity
function feeBalance(address token) external view returns (uint256);
```

##### feeTo
The address to which protocol fees are swept.

```solidity
function feeTo() external view returns (address);
```

##### setFeeTo
Updates the address fees can be swept to.

```solidity
function setFeeTo(address newFeeTo) external;
```

##### sweepFees
Sweeps fees to the feeTo address if there are more than 0 wei for each address in tokens.

```solidity
function sweepFees(address[] memory tokens) external;
```

##### setTokenURIGenerator
Updates the contract address for generating the token URI for claim NFTs.

```solidity
function setTokenURIGenerator(address newTokenURIGenerator) external;
```
