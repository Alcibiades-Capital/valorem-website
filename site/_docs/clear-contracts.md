---
date: 2022-11-30 00:00:00
title: Clear Smart Contract ABI
description: Developer documentation for the Valorem Clear smart contracts.
author:
  twitter: neodaoist
---

[Valorem Clear](/docs/clear-overview) is a persistent, non-upgradable smart contract written in Solidity,
responsible for the core clearing and settling of Valorem Options. It is designed to prioritize censorship resistance,
security, self-custody, and to function without any intermediaries who may restrict access. The source code can be found
on [GitHub](https://github.com/valorem-labs-inc/valorem-core).

The `ValoremOptionsClearinghouse` implements the `IValoremOptionsClearinghouse`
and [`ERC1155Metadata_URI`](https://eips.ethereum.org/EIPS/eip-1155) interfaces.

## Structs

### Option

Data comprising the unique tuple of an option type associated with an ERC-1155 option token.

- `underlyingAsset`: The underlying ERC20 asset which the option is collateralized with.
- `underlyingAmount`: The amount of the underlying asset contained within an option contract of this type.
- `exerciseAsset`: The ERC20 asset which the option can be exercised using.
- `exerciseAmount`: The amount of the exercise asset required to exercise each option contract of this type.
- `exerciseTimestamp`: The timestamp after which this option can be exercised.
- `expiryTimestamp`: The timestamp before which this option can be exercised.
- `settlementSeed`: Deterministic seed used for option fair exercise assignment.
- `nextClaimKey`: The next claim key available for this option type.

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

### Claim

Data about a claim to a short position written on an option type.
When writing an amount of options of a particular type, the writer will be issued an ERC 1155 NFT
that represents a claim to the underlying and exercise assets, to be claimed after
expiry of the option. The amount of each (underlying asset and exercise asset) paid to the claimant upon
redeeming their claim NFT depends on the option type, the amount of options written, represented in this struct,
and what portion of this claim was assigned exercise, if any, before expiry.

- `amountWritten`: The number of option contracts written against this claim expressed as a 1e18 scalar value.
- `amountExercised`: The amount of option contracts exercised against this claim expressed as a 1e18 scalar value.
- `optionId`: The option ID of the option type this claim is for.

```solidity
struct Claim {
    uint256 amountWritten;
    uint256 amountExercised;
    uint256 optionId;
}
```

### Position

Data about the ERC20 assets and liabilities for a given option (long) or claim (short) token,
in terms of the underlying and exercise ERC20 tokens.

- `underlyingAsset`: The address of the ERC20 underlying asset.
- `underlyingAmount`: The amount, in wei, of the underlying asset represented by this position.
- `exerciseAsset`: The address of the ERC20 exercise asset.
- `exerciseAmount`: The amount, in wei, of the exercise asset represented by this position.

```solidity
struct Position {
    address underlyingAsset;
    int256 underlyingAmount;
    address exerciseAsset;
    int256 exerciseAmount;
}
```

## Enums

### TokenType

The type of ERC1155 subtoken in Valorem Clear.

```solidity
enum TokenType {
    None,
    Option,
    Claim
}
```

## Events

### Write events

#### newOptionType

Emitted when a new option type is created.

- `optionId`: The token id of the new option type created.
- `exerciseAsset`: The ERC20 contract address of the exercise asset.
- `underlyingAsset`: The ERC20 contract address of the underlying asset.
- `exerciseAmount`: The amount, in wei, of the exercise asset required to exercise each contract.
- `underlyingAmount`: The amount, in wei of the underlying asset in each contract.
- `exerciseTimestamp`: The timestamp after which this option type can be exercised.
- `expiryTimestamp`: The timestamp before which this option type can be exercised.

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

Emitted when new options contracts are written.

- `optionId`: The token id of the option type written.
- `writer`: The address of the writer.
- `claimId`: The claim token id of the new or existing short position written against.
- `amount`: The amount of options contracts written.

```solidity
event OptionsWritten(uint256 indexed optionId, address indexed writer, uint256 indexed claimId, uint112 amount);
```

#### BucketWrittenInto

Emitted when options contracts are written into a bucket.

- `optionId`: The token id of the option type written.
- `claimId`: The claim token id of the new or existing short position written against.
- `bucketIndex`: The index of the bucket to which the options were written.
- `amount`: The amount of options contracts written.

```solidity
event BucketWrittenInto(
uint256 indexed optionId, uint256 indexed claimId, uint96 indexed bucketIndex, uint112 amount
);
```

### Redeem events

#### ClaimRedeemed

Emitted when a claim is redeemed.

- `optionId`: The token id of the option type of the claim being redeemed.
- `claimId`: The token id of the claim being redeemed.
- `redeemer`: The address redeeming the claim.
- `exerciseAmountRedeemed`: The amount of the option.exerciseAsset redeemed.
- `underlyingAmountRedeemed`: The amount of option.underlyingAsset redeemed.

```solidity
event ClaimRedeemed(
uint256 indexed claimId,
uint256 indexed optionId,
address indexed redeemer,
uint256 exerciseAmountRedeemed,
uint256 underlyingAmountRedeemed
);
```

### Exercise events

#### OptionsExercised

Emitted when option contract(s) is(are) exercised.

- `optionId`: The token id of the option type exercised.
- `exerciser`: The address that exercised the option contract(s).
- `amount`: The amount of option contracts exercised.

```solidity
event OptionsExercised(uint256 indexed optionId, address indexed exerciser, uint112 amount);
```

#### BucketAssignedExercise

Emitted when a bucket is assigned exercise.

- `optionId`: The token id of the option type exercised.
- `bucketIndex`: The index of the bucket which is being assigned exercise.
- `amountAssigned`: The amount of options contracts assigned exercise in the given bucket.

```solidity
event BucketAssignedExercise(uint256 indexed optionId, uint96 indexed bucketIndex, uint112 amountAssigned);
```

### Fee events

#### FeeAccrued

Emitted when protocol fees are accrued for a given asset.

- `optionId`: The token id of the option type being written or exercised.
- `asset`: The ERC20 asset in which fees were accrued.
- `payer`: The address paying the fee.
- `amount`: The amount, in wei, of fees accrued.

_Dev note — Emitted on write() when fees are accrued on the underlying asset, or exercise() when fees are accrued on the
exercise asset. Will not be emitted when feesEnabled is false._

```solidity
event FeeAccrued(uint256 indexed optionId, address indexed asset, address indexed payer, uint256 amount);
```

#### FeeSwept

Emitted when accrued protocol fees for a given ERC20 asset are swept to the feeTo address.

- `asset`: The ERC20 asset of the protocol fees swept.
- `feeTo`: The account to which fees were swept.
- `amount`: The total amount swept.

```solidity
event FeeSwept(address indexed asset, address indexed feeTo, uint256 amount);
```

#### FeeSwitchUpdated

Emitted when protocol fees are enabled or disabled.

- `feeTo`: The address which enabled or disabled fees.
- `enabled`: Whether fees are enabled or disabled.

```solidity
event FeeSwitchUpdated(address feeTo, bool enabled);
```

### Access control events

#### FeeToUpdated

Emitted when feeTo address is updated.

- `newFeeTo`: The new feeTo address.

```solidity
event FeeToUpdated(address indexed newFeeTo);
```

#### TokenURIGeneratorUpdated

Emitted when TokenURIGenerator is updated.

- `newTokenURIGenerator`: The new TokenURIGenerator address.

```solidity
event TokenURIGeneratorUpdated(address indexed newTokenURIGenerator);
```

## Errors

### Access control errors

#### AccessControlViolation

The caller doesn't have permission to access that function.

- `accessor`: The requesting address.
- `permissioned`: The address which has the requisite permissions.

```solidity
error AccessControlViolation(address accessor, address permissioned);
```

### Input errors

#### AmountWrittenCannotBeZero

The amount of option contracts written must be greater than zero.

```solidity
error AmountWrittenCannotBeZero();
```

#### CallerDoesNotOwnClaimId

This claim is not owned by the caller.

- `claimId`: Supplied claim ID.

```solidity
error CallerDoesNotOwnClaimId(uint256 claimId);
```

#### CallerHoldsInsufficientOptions

The caller does not have enough option contracts to exercise the amount specified.

- `optionId`: The supplied option id.
- `amount`: The amount of option contracts which the caller attempted to exercise.

```solidity
error CallerHoldsInsufficientOptions(uint256 optionId, uint112 amount);
```

#### ClaimTooSoon

Claims cannot be redeemed before expiry.

- `claimId`: Supplied claim ID.
- `expiry`: The timestamp at which the option type expires.

```solidity
error ClaimTooSoon(uint256 claimId, uint40 expiry);
```

#### ExerciseTooEarly

This option cannot yet be exercised.

- `optionId`: Supplied option ID.
- `exercise`: The time after which the option can be exercised.

```solidity
error ExerciseTooEarly(uint256 optionId, uint40 exercise);
```

#### ExerciseWindowTooShort

The option exercise window is too short.

- `exercise`: The timestamp supplied for exercise.

```solidity
error ExerciseWindowTooShort(uint40 exercise);
```

#### ExpiredOption

The optionId specified has already expired.

- `optionId`: The id of the expired option.
- `expiry`: The expiry time for the supplied option id.

```solidity
error ExpiredOption(uint256 optionId, uint40 expiry);
```

#### ExpiryWindowTooShort

The expiry timestamp is too soon.

- `expiry`: The expiry timestamp.

```solidity
error ExpiryWindowTooShort(uint40 expiry);
```

#### InvalidAddress

Invalid (zero) address.

- `input`: The address input.

```solidity
error InvalidAddress(address input);
```

#### InvalidAssets

The assets specified are invalid or duplicate.

- `asset1`: Supplied ERC20 asset.
- `asset2`: Supplied ERC20 asset.

```solidity
error InvalidAssets(address asset1, address asset2);
```

#### InvalidClaim

The token specified is not a claim token.

- `token`: The supplied token id.

```solidity
error InvalidClaim(uint256 token);
```

#### InvalidOption

The token specified is not an option token.

- `token`: The supplied token id.

```solidity
error InvalidOption(uint256 token);
```

#### OptionsTypeExists

This option contract type already exists and thus cannot be created.

- `optionId`: The token id of the option type which already exists.

```solidity
error OptionsTypeExists(uint256 optionId);
```

#### TokenNotFound

The requested token is not found.

- `token`: The token requested.

```solidity
error TokenNotFound(uint256 token);
```

## Views

### Option information

#### option

Gets information about an option.

- `tokenId`: The tokenId of an option or claim.
- `optionInfo`: The Option for the given tokenId.

```solidity
function option(uint256 tokenId) external view returns (Option memory optionInfo);
```

#### claim

Gets information about a claim.

- `claimId`: The tokenId of the claim.

```solidity
function claim(uint256 claimId) external view returns (Claim memory claimInfo);
```

#### position

Gets information about the ERC20 token positions of an option or claim.

- `tokenId`: The tokenId of the option or claim.

```solidity
function position(uint256 tokenId) external view returns (Position memory positionInfo);
```

### Token information

#### tokenType

Gets the TokenType for a given tokenId.

- `tokenId`: The token id to get the TokenType of.

_Dev note — Option and claim token ids are encoded as follows:_

```
MSb
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │ 160b option key, created from Option struct hash.
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 │
0000 0000   0000 0000   0000 0000   0000 0000 ┘
0000 0000   0000 0000   0000 0000   0000 0000 ┐
0000 0000   0000 0000   0000 0000   0000 0000 │ 96b auto-incrementing claim key.
0000 0000   0000 0000   0000 0000   0000 0000 ┘
                                          LSb
```

```solidity
function tokenType(uint256 tokenId) external view returns (TokenType typeOfToken);
```

#### tokenURIGenerator

Gets the contract address for generating token URIs for tokens.

```solidity
function tokenURIGenerator() external view returns (ITokenURIGenerator uriGenerator);
```

### Fee information

#### feeBalance

Gets the balance of protocol fees for a given token which have not been swept yet.

- `token`: The token for the un-swept fee balance.

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

Returns the address to which protocol fees are swept.

```solidity
function feeTo() external view returns (address);
```

## Write options

#### newOptionType

Creates a new option contract type if it doesn't already exist.

- `underlyingAsset`: The contract address of the ERC20 underlying asset.
- `underlyingAmount`: The amount of underlyingAsset, in wei, collateralizing each option contract.
- `exerciseAsset`: The contract address of the ERC20 exercise asset.
- `exerciseAmount`: The amount of exerciseAsset, in wei, required to exercise each option contract.
- `exerciseTimestamp`: The timestamp after which this option can be exercised.
- `expiryTimestamp`: The timestamp before which this option can be exercised.

Returns `optionId`: The token id for the new option type created by this call.

_Dev note — optionId can be precomputed using:_

```
uint160 optionKey = uint160(
    bytes20(
        keccak256(
            abi.encode(
                 underlyingAsset,
                underlyingAmount,
                exerciseAsset,
                exerciseAmount,
                exerciseTimestamp,
                expiryTimestamp,
                uint160(0),
                uint96(0)
            )
        )
    )
);
optionId = uint256(optionKey) << OPTION_ID_PADDING;
```

_and then_ `tokenType(optionId) == TokenType.Option` _to check if the option already exists._

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

Writes a specified amount of the specified option.

- `tokenId`: The desired token id to write against, input an optionId to get a new claim, or a claimId to add to an
  existing claim.
- `amount`: The desired number of option contracts to write.

Returns `claimId`: The token id of the claim NFT which was input or created.

```solidity
function write(uint256 tokenId, uint112 amount) external returns (uint256 claimId);
```

## Redeem claims

#### redeem

Redeems a claim NFT, transfers the underlying/exercise tokens to the caller. Can be called after option expiry
timestamp (inclusive).

- `claimId`: The ID of the claim to redeem.

```solidity
function redeem(uint256 claimId) external;
```

## Exercise options

#### exercise

Exercises specified amount of optionId, transferring in the exercise asset, and transferring out the underlying asset if
requirements are met. Can be called from exercise timestamp (inclusive), until option expiry timestamp (exclusive).

- `optionId`: The option token id of the option type to exercise.
- `amount`: The amount of option contracts to exercise.

```solidity
function exercise(uint256 optionId, uint112 amount) external;
```

## Protocol administration

#### setFeesEnabled

Enables or disables protocol fees.

- `enabled`: Whether protocol fees should be enabled.

```solidity
function setFeesEnabled(bool enabled) external;
```

#### setFeeTo

Nominates a new address to which fees should be swept, requiring the new feeTo address to accept before the update is
complete. See also acceptFeeTo().

- `newFeeTo`: The new address to which fees should be swept.

```solidity
function setFeeTo(address newFeeTo) external;
```

#### acceptFeeTo

Accepts the new feeTo address and completes the update. See also setFeeTo(address newFeeTo).

_See also setFeeTo(address newFeeTo)._

```solidity
function acceptFeeTo() external;
```

#### setTokenURIGenerator

Updates the contract address for generating token URIs for tokens.

- `newTokenURIGenerator`: The address of the new ITokenURIGenerator contract.

```solidity
function setTokenURIGenerator(address newTokenURIGenerator) external;
```

#### sweepFees

Sweeps fees to the feeTo address if there is more than 1 wei for feeBalance for a given token.

- `tokens`: An array of tokens to sweep fees for.

```solidity
function sweepFees(address[] memory tokens) external;
```
