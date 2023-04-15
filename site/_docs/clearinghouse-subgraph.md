---
date: 2023-04-12 00:00:00
title: Querying the Subgraph
description: Developer documentation for the Valorem Options Clearinghouse subgraph.
---

- TODO review and revise examples (feel free to add additional or remove extraneous)

The Clearinghouse Subgraph indexes data on the Valorem Options Clearinghouse smart contract using a GraphQL interface. It updates data in response to contract ABI function calls and events. The Subgraph can be used to power frontend apps and integration use cases.

The GraphQL schema for the subgraph is defined at [/schema.graphql](https://github.com/valorem-labs-inc/valorem-subgraph/blob/master/schema.graphql)

#### Subgraph URLs

| Network            | Subgraph URL |
| ------------------ | ------------ |
| Arbitrum           | [/valorem-options-clearinghouse-arbitrum](https://thegraph.com/legacy-explorer/subgraph/valorem-labs/valorem-options-clearinghouse-arbitrum)        |
| Arbitrum Goerli    | [/valorem-options-clearinghouse-arbitrum-goerli](https://thegraph.com/legacy-explorer/subgraph/balancer-labs/valorem-options-clearinghouse-arbitrum-goerli) |

## Example Queries

The following example queries can be used to retrieve useful data from the subgraph.

### Option tokens for an address
```graphql

```

### Claim NFTs for an address
```graphql

```

### Addresses writing options

```graphql
{
  optionTypes(first: 5, orderBy: totalUnderlyingAmount, orderDirection: desc) {
    id
    totalUnderlyingAmount
    totalExerciseAmount
    totalOptions
  }
}
```

### Addresses exercising options

```graphql
{
  optionExercises(first: 5, orderBy: amountExercised, orderDirection: desc) {
    id
    amountExercised
    amountExercisedUnderlying
    amountExercisedExercise
  }
}
```

### Popular underlying assets

```graphql
{
  optionTypes(first: 5, orderBy: totalUnderlyingAmount, orderDirection: desc) {
    id
    underlyingAsset
    totalUnderlyingAmount
    totalExerciseAmount
    totalOptions
  }
}
```

### Popular exercise assets

```graphql
{
  optionTypes(first: 5, orderBy: totalExerciseAmount, orderDirection: desc) {
    id
    exerciseAsset
    totalUnderlyingAmount
    totalExerciseAmount
    totalOptions
  }
}
```

### Recently created option types

```graphql
{
  optionTypes(first: 5, orderBy: createdAt, orderDirection: desc) {
    id
    createdAt
    updatedAt
    creator
    underlyingAsset
    underlyingAmount
    exerciseAsset
    exerciseAmount
    exerciseTimestamp
    expiryTimestamp
    settlementSeed
    nextClaimKey
  }
}
```

### Recently exercised options

```graphql
{
  optionExercises(first: 5, orderBy: timestamp, orderDirection: desc) {
    id
    timestamp
    optionId
    amount
    amountExercised
    amountExercisedUnderlying
    amountExercisedExercise
    amountRemaining
    amountRemainingUnderlying
    amountRemainingExercise
  }
}
```

### Recently redeemed claims

```graphql
{
  claims(first: 5, orderBy: lastRedeem, orderDirection: desc) {
    id
    lastRedeem
    lastRedeemAmount
    lastRedeemPrice
  }
}
```
