---
date: 2023-04-12 00:00:00
title: Querying the Subgraph
description: Developer documentation for the Valorem Options Clearinghouse subgraph.
---

The Clearinghouse Subgraph indexes data on the Valorem Options Clearinghouse smart contract using a GraphQL interface. It updates data in response to contract ABI function calls and events. The Subgraph can be used to power frontend apps and integration use cases.

The GraphQL schema for the subgraph is defined at [/schema.graphql](https://github.com/valorem-labs-inc/valorem-subgraph/blob/master/schema.graphql)

#### Subgraph URLs

| Network            | Subgraph URL |
| ------------------ | ------------ |
| Arbitrum           | [/valorem-options-clearinghouse-arbitrum](https://thegraph.com/legacy-explorer/subgraph/valorem-labs/valorem-options-clearinghouse-arbitrum)        |
| Arbitrum Goerli    | [/valorem-options-clearinghouse-arbitrum-goerli](https://thegraph.com/hosted-service/subgraph/nickadamson/ch-arb-goerli) |

## Example Queries

The following example queries can be used to retrieve useful data from the subgraph.

### Option tokens for an address
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

### Claim NFTs for an address
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

### Addresses writing options

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

### Popular underlying assets

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

### Popular exercise assets

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
