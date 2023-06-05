---
date: 2023-05-29 00:00:00
title: Trade API Reference
description: Developer documentation for the Valorem Exchange API.
---

## Introduction

Valorem Trade has an expanding ecosystem of high-performance options trading infrastructure. The API currently supports RFQ (Request for Quote) services and returns signed offers for execution on [Seaport](https://docs.opensea.io/reference/seaport-overview). We utilise the [gRPC](https://grpc.io/docs/what-is-grpc/introduction/) framework and use protocol buffers for data serialization for fast and efficient communication.

### API Endpoint

`https://exchange.valorem.xyz/`

### Proto Files

[rfq.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/rfq.proto)

[seaport.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/seaport.proto)

[session.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/session.proto)

[types.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/types.proto)

 

## Example Usage

We will use [Connect-Node](https://connect.build/docs/node/getting-started) to make requests to the gRPC API in the following examples.

### Starting Out

Install the neccessary dependencies outlined in this package.json.
```bash
yarn install
```
Then generate the code from the protobuf service definitions. We will use [Buf](https://www.npmjs.com/package/@bufbuild/buf), a modern replacement for Google's protobuf compiler, using the following config file.

`buf.gen.yaml`
```yaml
version: v1
plugins:
  - plugin: es
    opt: target=ts,import_extension=none
    out: gen
  - plugin: connect-es
    opt: target=ts,import_extension=none
    out: gen
```
Then generate the code from the proto definitions stored in the `proto` directory.
```
npx buf generate proto
```

### RFQ Taker

#### 1. Connect and authenticate with Quay

```typescript

```

#### 2. Request a buy quote from the Maker


#### 3. If the Maker offers a quote:


#### 4. Accept and fulfill the Order from the Maker


### 5. Request a sell quote from the Maker


### 6. Accept and fulfill the Order from the Maker



## RFQ Maker
