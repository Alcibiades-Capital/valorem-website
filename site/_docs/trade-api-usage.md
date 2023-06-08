---
date: 2023-06-07 00:00:00
title: Trade API Usage
description: Developer documentation for the Valorem Exchange API.
---

## Environment setup for Typescript

We will use [Connect-Node](https://connect.build/docs/node/getting-started) to connect with the gRPC API in the following examples.

It is most efficient to clone [this repository](https://github.com/valorem-labs-inc/exchange-proto/tree/main) containing all the protobuf definitions and examples to get started.

Install the neccessary dependencies outlined in the [package.json](https://github.com/valorem-labs-inc/exchange-proto/blob/rfq-api-usage-examples/package.json).

Then generate the code from the protobuf service definitions. We will use [Buf](https://www.npmjs.com/package/@bufbuild/buf), a modern replacement for Google's protobuf compiler, using the following config file.

`buf.gen.yaml`
```yaml
version: v1
plugins:
  - plugin: es
    opt: target=ts,import_extension=none
    out: gen/quay
  - plugin: connect-es
    opt: target=ts,import_extension=none
    out: gen/quay
```
Use this command to generate the proto definitions from the proto files in `proto/quay`.
```
npx buf generate proto/quay
```


### Connecting and Authenticating with Valorem Trade

We use [sign-in with ethereum](https://docs.login.xyz/) for authenticating our users.

```typescript
import { createPromiseClient } from '@bufbuild/connect';
import { createGrpcTransport } from '@bufbuild/connect-node';
import { SiweMessage } from 'siwe';
import * as ethers from 'ethers';
import { Session } from '../gen/quay/session_connect';

// replace with account to use for signing
const PRIVATE_KEY = '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80';
const NODE_ENDPOINT = 'https://goerli-rollup.arbitrum.io/rpc';

const provider = new ethers.providers.JsonRpcProvider(NODE_ENDPOINT);
const signer = new ethers.Wallet(PRIVATE_KEY, provider);

const CHAIN_ID = 421613;  // Arbitrum Goerli
const gRPC_ENDPOINT = 'https://exchange.valorem.xyz';
const DOMAIN = 'exchange.valorem.xyz';

var cookie: string;  // to be used for all server interactions
// custom Connect interceptor for retrieving cookie
const trackCookie= (next: any) => async (req: any) => {
  const res = await next(req);
  cookie = res.header?.get('set-cookie')?.split(';')[0] ?? cookie;
  return res
};

// transport for connection to Valorem Trade gRPC server
const transport = createGrpcTransport({
  baseUrl: gRPC_ENDPOINT,
  httpVersion: '2',
  interceptors: [trackCookie]
});

async function authenticateWithTrade() {
  const sessionClient = createPromiseClient(Session, transport);
  const { nonce } = await sessionClient.nonce({});

  // create SIWE message
  const message = new SiweMessage({
    domain: DOMAIN,
    address: signer.address,
    uri: gRPC_ENDPOINT,
    version: '1',
    chainId: CHAIN_ID,
    nonce,
    issuedAt: new Date().toISOString(),
  }).toMessage();

  // sign SIWE message
  const signature = await signer.signMessage(message);

  // verify with Valorem Trade
  await sessionClient.verify(
    {
      body: JSON.stringify({
        message: message,
        signature: signature,
      })
    },
    {headers: [['cookie', cookie]]},
  );

  // authenticate with Valorem Trade
  await sessionClient.authenticate({}, {headers: [['cookie', cookie]]});

  console.log('Client has authenticated with Valorem Trade!');
}
```

### RFQ Maker

After connecting and authenticating with Valorem Trade, we can now make requests to the RFQ service.
Here we will connect as an RFQ client, listen for incoming quote requests from takers, and respond with quotes. The full working example can be found [here](https://github.com/valorem-labs-inc/exchange-proto/blob/main/examples/RFQ_maker.ts).

Note: the below code is for demonstration purposes only so you should create your own robust steam handling for production.

```typescript
import { RFQ } from '../gen/quay/rfq_connect';
import { QuoteResponse } from '../gen/quay/rfq_pb';
import { toH160} from '../lib/fromBNToH';

async function createResponse(optionId: ethers.BigNumber) {
  const rfqClient = createPromiseClient(RFQ, transport);

  const response = new QuoteResponse({ 
    ulid: undefined,
    makerAddress: toH160(signer.address),
    order: undefined,
  });

  // create your own quote request and response stream handling logic here
  const responseStream = async function* () {
    yield response;
  };

  const requestStream = rfqClient.maker(
    responseStream(),
    {headers: [['cookie', cookie]]}
  );

  for await (const request of requestStream) {
    console.log(request);
  }
};

```

### RFQ Taker

After connecting and authenticating with Valorem Trade, we can now make requests to the RFQ service.
Here we will request a quote to buy an option, and listen for responses. The full working example can be found [here](https://github.com/valorem-labs-inc/exchange-proto/blob/main/examples/RFQ_taker.ts).

Note: the below code is for demonstration purposes only so you should create your own robust stream handling for production.

```typescript
import { RFQ } from '../gen/quay/rfq_connect';
import { Action, QuoteRequest } from '../gen/quay/rfq_pb';
import { ItemType } from '../gen/quay/seaport_pb';
import { toH160, toH256 } from '../lib/fromBNToH';

async function createRequest(optionId: ethers.BigNumber) {
  const rfqClient = createPromiseClient(RFQ, transport);

  // create an option buy quote request 
  const request = new QuoteRequest({
    ulid: undefined,
    takerAddress: toH160(signer.address),
    itemType: ItemType.NATIVE,
    tokenAddress: toH160(VALOREM_CLEAR_ADDRESS),
    identifierOrCriteria: toH256(optionId),
    amount: toH256(BigInt(5)),
    action: Action.BUY
  });

  // create your own quote request and response stream handling logic here
  const requestStream = async function* () {
    yield request;
  };

  const responseStream = rfqClient.taker(
    requestStream(), 
    {headers: [['cookie', cookie]]}
  );

  for await (const response of responseStream) {
    console.log(response);
  }
};
```
