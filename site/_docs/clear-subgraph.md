---
date: 2023-04-12 00:00:00
title: Clear Subgraph
description: Get insights into the Valorem Clear Subgraph, a tool for indexing and querying data from Valorem Clear smart contracts using GraphQL. Discover example queries, supported networks, and how it powers frontend apps.
---

### Understanding the Valorem Clear Subgraph

Welcome to the guide on the Valorem Clear Subgraph. The Clear Subgraph is designed to index data directly from the 
Valorem Clear smart contracts, queryable through a user-friendly GraphQL interface. It dynamically updates data as 
ABI function calls and events unfold. Whether you're looking to power frontend applications or integrate specific 
use cases, our subgraph offers a seamless solution. Below, we've provided essential URLs for different networks and 
sample GraphQL queries to help you extract the most out of this feature-rich tool. Dive in and explore how the 
Valorem Clear Subgraph can transform your Valorem integration.

The GraphQL schema for the subgraph is defined [here](https://github.com/valorem-labs-inc/valorem-subgraph/blob/master/schema.graphql).

The subgraph is designed to give developers a clear view of the dynamic actions happening within Valorem Clear, from 
option type creation to exercise and everything in between. So, saddle up! Let's embark on this journey together.

#### Subgraph URLs

| Network         | Subgraph URL                                                                                                                 |
|-----------------|------------------------------------------------------------------------------------------------------------------------------|
| Arbitrum        | [/valorem-clear-arbitrum](https://thegraph.com/legacy-explorer/subgraph/valorem-labs/valorem-options-clearinghouse-arbitrum) |
| Arbitrum Goerli | [/valorem-clear-arbitrum-goerli](https://thegraph.com/hosted-service/subgraph/nickadamson/ch-arb-goerli)                     |


### Entities
1. `OptionType`
   - This represents the very heart of an option. It contains detailed information about the underlying asset, exercise asset, option's time window, and the creator's information.

2. `Claim`
   - A claim outlines the collateralized options. Every claim relates to an `OptionType` and tracks the short side of the option's lifecycle.

3. `OptionTypeBucket` & `ClaimBucket`
   - These denote the various buckets into which options and claims are categorized.

4. `DayData` & `TokenDayData`
   - For those who love crunching daily data, these entities contain metrics of notional volumes, total values locked, 
     and more, available both for the whole clearing system and specific tokens.

5. `Token`

   - Essential details of a token, including the total amount locked in the clearing system and fees related data.

6. `ERC1155` Extensions

   The subgraph is designed in harmony with ERC-1155, a multi-token standard. Here you'll find how each token and 
   balance is mapped in the Valorem ecosystem.

### Example queries

The following example queries can be used to retrieve useful data from the subgraph.

#### Option tokens for an address

```graphql
query OptionTokensForAddress($address: ID!) {
    account(id: $address) {
        ERC1155balances(where: { valueExact_gt: "0" }) {
            token {
                id
                type
                optionType {
                    id
                    underlyingAsset {
                        id
                        symbol
                        name
                        decimals
                    }
                    underlyingAmount
                    exerciseAsset {
                        id
                        symbol
                        name
                        decimals
                    }
                    exerciseAmount
                    exerciseTimestamp
                    expiryTimestamp
                    amountWritten
                    amountExercised
                }
            }
            value
        }
    }
}
```

#### Claim NFTs for an address

```graphql

query ClaimNFTsForAddress($address: ID!) {
    account(id: $address) {
        ERC1155balances(where: { valueExact_gt: "0" }) {
            token {
                id
                type
                claim {
                    id
                    writer {
                        id
                    }
                    writeTx {
                        id
                    }
                    redeemed
                    redeemer {
                        id
                    }
                    redeemTx {
                        id
                    }
                    optionType {
                        id
                        underlyingAsset {
                            id
                            symbol
                        }
                        underlyingAmount
                        exerciseAsset {
                            id
                            symbol
                        }
                        exerciseAmount
                        exerciseTimestamp
                        expiryTimestamp
                        amountWritten
                        amountExercised
                    }
                }
            }
            value
        }
    }
}
```

#### Addresses writing options

```graphql
{
    claims(first: 5, orderBy: amountWritten, orderDirection: desc) {
        id
        writer {
            id
        }
    }
}
```

#### Popular underlying assets

```graphql
query PopularUnderlyingAssets {
    optionTypes(orderBy: amountWritten, orderDirection: desc) {
        id
        amountWritten
        underlyingAsset {
            id
            symbol
            name
        }
    }
}
```

#### Popular exercise assets

```graphql
query PopularExerciseAssets {
    optionTypes(orderBy: amountExercised, orderDirection: desc) {
        exerciseAsset {
            id
            symbol
            name
        }
    }
}
```
