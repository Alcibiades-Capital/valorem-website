---
date: 2023-06-07 00:00:00
title: Trade API Usage
description: Developer documentation for the Valorem Exchange API.
---

## Environment setup for Typescript

We will use [Connect-Node](https://connect.build/docs/node/getting-started) to connect with the gRPC API in the following examples.

It is most efficient to clone [this repository](https://github.com/valorem-labs-inc/trade-interfaces/tree/main) containing all the protobuf definitions and examples to get started.

Install the neccessary dependencies outlined in the [package.json](https://github.com/valorem-labs-inc/trade-interfaces/tree/main/docs/examples/typescript/package.json).

Then generate the code from the protobuf service definitions. We will use [Buf](https://www.npmjs.com/package/@bufbuild/buf), a modern replacement for Google's protobuf compiler, using the following config file.

`buf.gen.yaml`
```yaml
version: v1
plugins:
  - plugin: es
    opt: target=ts,import_extension=none
    out: gen/valorem/trade/v1
  - plugin: connect-es
    opt: target=ts,import_extension=none
    out: gen/valorem/trade/v1
```
Use this command to generate from the proto files in `proto/quay`.
```bash
npx buf generate proto/quay
```

## Connecting and Authenticating with Valorem Trade

We use [sign-in with ethereum](https://docs.login.xyz/) for authenticating our users.

```typescript
import { createPromiseClient } from '@bufbuild/connect';
import { createGrpcTransport } from '@bufbuild/connect-node';
import { SiweMessage } from 'siwe';
import { Wallet, providers } from 'ethers';  // v5.5.0
const { JsonRpcProvider } = providers;
import { Auth } from '../gen/valorem/trade/v1/auth_connect';  // generated from auth.proto

// replace with account to use for signing
const PRIVATE_KEY = '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80';
const NODE_ENDPOINT = 'https://goerli-rollup.arbitrum.io/rpc';

const provider = new JsonRpcProvider(NODE_ENDPOINT);
const signer = new Wallet(PRIVATE_KEY, provider);

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
  const authClient = createPromiseClient(Auth, transport);
  const { nonce } = await authClient.nonce({});
  const { chainId } = await provider.getNetwork();
  // create SIWE message
  const message = new SiweMessage({
    domain: DOMAIN,
    address: signer.address,
    uri: gRPC_ENDPOINT,
    version: '1',
    chainId: chainId,
    nonce,
    issuedAt: new Date().toISOString(),
  }).toMessage();

  // sign SIWE message
  const signature = await signer.signMessage(message);

  // verify with Valorem Trade
  await authClient.verify(
    {
      body: JSON.stringify({
        message: message,
        signature: signature,
      })
    },
    {headers: [['cookie', cookie]]},
  );

  // authenticate with Valorem Trade
  await authClient.authenticate({}, {headers: [['cookie', cookie]]});

  console.log('Client has authenticated with Valorem Trade!');
}
```

## RFQ Taker
After connecting and authenticating with Valorem Trade, we can now make requests to the RFQ service.

Here we will connect as an RFQ client to the stream, request a quote to buy an option, listen for responses, and execute the signed orders on Seaport.

To send a request you will need an option ID which can be [calculated from the option parameters](https://github.com/valorem-labs-inc/tree/main/docs/examples/typescript/src/lib/getOptionId.ts#L34), or it is returned when you initialse a new option type with the clearing house.

The full working example can be found [here](https://github.com/valorem-labs-inc/trade-interfaces/tree/main/docs/examples/typescript/src/RFQ_taker.ts).

Note: the below code is for demonstration purposes only so you should create your own robust stream handling for production.
```typescript
import { BigNumber, constants } from 'ethers';  // v5.5.0
const { formatUnits } = utils;
import { RFQ } from '../gen/valorem/trade/v1/rfq_connect';  // generated from rfq.proto
import { Action, QuoteRequest } from '../gen/valorem/trade/v1/rfq_pb';  // generated from rfq.proto
import { ItemType } from '../gen/valorem/trade/v1/seaport_pb';  // generated from seaport.proto
import { toH160, toH256 } from './lib/fromBNToH';  // library script for H number conversions
import ISeaport from '../../abi/ISeaport.json';
import IERC20 from '../../abi/IERC20.json';

const SEAPORT_ADDRESS = '0x00000000006c3852cbEf3e08E8dF289169EdE581';  // seaport 1.1
const seaportContract = new Contract(SEAPORT_ADDRESS, ISeaport, provider);
const usdcContract = new Contract(USDC_ADDRESS, IERC20, provider);

async function sendRfqRequests(optionId: BigNumber) {
  /* Send RFQ requests, then and execute the returned signed offers on Seaport */
  const rfqClient = createPromiseClient(RFQ, transport);

  // Create your own quote request and response stream handling logic here!

  // create a quote request to buy 5 options
  const quoteRequest = new QuoteRequest({
    takerAddress: toH160(signer.address),
    itemType: ItemType.ERC1155,  // see Seaport ItemType enum
    tokenAddress: toH160(VALOREM_CLEAR_ADDRESS),  // clearing house is the options token contract
    identifierOrCriteria: toH256(optionId),  // the erc1155 token id = optionId
    amount: toH256(5),  // 5 options
    action: Action.BUY  
  });

  // continuously send requests and handle responses
  console.log('Sending RFQs for option ID', optionId.toString());

  // create your own quote request and response stream handling logic here
  const quoteRequestStream = async function* () {
    yield quoteRequest;
  };

  while (true) {
    for await (const quoteResponse of rfqClient.taker(quoteRequestStream(), {headers: [['cookie', cookie]]})) {
      if (Object.keys(quoteResponse).length === 0) { continue };  // empty response
      console.log('Received a quote response...');

      // format the response into an order to be executed on seaport
      const signedOrder = await formatQuoteResponse(quoteResponse);

      console.log('Accepting quote to buy', signedOrder.parameters.offer[0].startAmount, 'options for', formatUnits(signedOrder.parameters.consideration[0].startAmount, 6) , 'USDC.'); 
      
      console.log('Executing order on Seaport...');

      // first approve Seaport spend of usdc price
      let txReceipt = await (await usdcContract.connect(signer).approve(SEAPORT_ADDRESS, signedOrder.parameters.consideration[0].startAmount)).wait();  // assumes start and end are the same
      if (txReceipt.status == 0) {
        console.log('Skipping executing order; USDC approval failed.');
        return;
      };
      
      txReceipt = await (await seaportContract.connect(signer).fulfillOrder(signedOrder, constants.HashZero)).wait();
      if (txReceipt.status == 0) {
        console.log('Skipping executing order; order fulfillment failed.');
        return;
      };

      console.log('Success!');
      console.log('txn hash:', txReceipt.transactionHash);
    };
  };
};
```

Upon receiving quotes, we format the responses into the signed orders for seaport.
```typescript
const { hexValue, joinSignature, hexlify } = utils;
import { QuoteResponse } from '../gen/valorem/trade/v1/rfq_pb';  // generated from rfq.proto
import { fromH160, fromH256 } from './lib/fromHToBN';  // library script for H number conversions

async function formatQuoteResponse(quoteResponse: QuoteResponse) {
  /* Format quote responses from makers into the signed order for Seaport */
  // convert order fields from H types back to BigNumbers
  const signedOrder_H = quoteResponse.order;
  const order_H = signedOrder_H.parameters;
  const [ offerItem_H ] = order_H.offers;
  const [ considerationItem_H ] = order_H.considerations;

  const offerItem = {
    itemType: offerItem_H.itemType,
    token: hexValue(fromH160(offerItem_H.token)),
    identifierOrCriteria: fromH256(offerItem_H.identifierOrCriteria),
    startAmount: fromH256(offerItem_H.startAmount),
    endAmount: fromH256(offerItem_H.endAmount),
  };
  const considerationItem = {
    itemType: considerationItem_H.itemType,
    token: hexValue(fromH160(considerationItem_H.token)),
    identifierOrCriteria: fromH256(considerationItem_H.identifierOrCriteria),
    startAmount: fromH256(considerationItem_H.startAmount),
    endAmount: fromH256(considerationItem_H.endAmount),
    recipient: hexValue(fromH160(considerationItem_H.recipient)),
  };

  if (considerationItem.token !== USDC_ADDRESS) { 
    console.log('Skipping responding to RFQ; only accepting quotes in USDC.');
    return;
  };

  const OrderComponents = {
    offerer: hexValue(fromH160(order_H.offerer)),
    zone: hexValue(fromH160(order_H.zone)),
    offer: [ offerItem ],
    consideration: [ considerationItem ],
    orderType: order_H.orderType,
    startTime: fromH256(order_H.startTime),
    endTime: fromH256(order_H.endTime),
    zoneHash: fromH256(order_H.zoneHash),
    salt: fromH256(order_H.salt),
    conduitKey: fromH256(order_H.conduitKey),
  };

  const signature = joinSignature({
    r: hexlify(signedOrder_H.signature.r),
    s: hexlify(signedOrder_H.signature.s),
    v: BigNumber.from(hexlify(signedOrder_H.signature.v)).toNumber(),
  }) as `0x${string}`;

  const signedOrder = {
    parameters: OrderComponents,
    signature: signature,
  };

  if (considerationItem.startAmount.gt(parseUnits('200', 6))) {  // assumes start and end amount are equal
    console.log('Skipping responding to RFQ; only accepting quotes for 200 USDC or less.');
    return;
  };

  return signedOrder;
};
```


## RFQ Maker
After connecting and authenticating with Valorem Trade, we can now respond to requests from takers on the RFQ service.

Here we will connect as an RFQ client, listen for quote requests, and respond with signed offers for Seaport 

The full working example can be found [here](https://github.com/valorem-labs-inc/trade-interfaces/tree/main/docs/examples/typescript/src/RFQ_maker.ts).

Note: the below code is for demonstration purposes only so you should create your own robust steam handling for production.
```typescript
import { Contract } from 'ethers';  // v5.5.0
import { RFQ } from '../gen/valorem/trade/v1/rfq_connect';  // generated from rfq.proto
import { QuoteResponse } from '../gen/valorem/trade/v1/rfq_pb';  // generated from rfq.proto
import IERC1155abi from '../../abi/IERC1155.json';

const VALOREM_CLEAR_ADDRESS = '0x7513F78472606625A9B505912e3C80762f6C9Efb';  // Valorem Clearinghouse on Arb Goerli
const SEAPORT_ADDRESS = '0x00000000006c3852cbEf3e08E8dF289169EdE581';  // seaport 1.1

const optionTokenContract = new Contract(VALOREM_CLEAR_ADDRESS, IERC1155abi, provider);

async function respondToRfqs() {
  /* Listen for RFQs and respond with signed Seaport offers */
  const rfqClient = createPromiseClient(RFQ, transport);

  // Create your own quote request and response stream handling logic here!

  // empty response used to open the stream
  const emptyQuoteResponse = new QuoteResponse();
  var emptyQuoteResponseStream = async function* () {
    yield emptyQuoteResponse;
  };

  console.log('Listening for RFQs...');

  while (true) {
    for await (const quoteRequest of rfqClient.maker(emptyQuoteResponseStream(), {headers: [['cookie', cookie]]})) {
      console.log('Received a quote request...');

      // construct a quote response with a signed seaport offer
      const quoteResponse = await constructQuoteResponse(quoteRequest);

      console.log('Sending quote response...');

      // send response over RFQ service
      const quoteResponseStream = async function* () {
        yield quoteResponse;
      };
    
      // approve spend of option amount before sending offer
      let txReceipt = await (await optionTokenContract.connect(signer).setApprovalForAll(SEAPORT_ADDRESS, true)).wait();
      if (txReceipt.status == 0) { 
        console.log('Skipping responding to RFQ; Option ERC1155 token spend approval failed.');
        return;
      };
    
      rfqClient.maker(
        quoteResponseStream(), 
        {headers: [['cookie', cookie]]}
      );

    };
  };
};
```

Upon receiving requests, we will construct the signed offers and wrap in a quote response.
```typescript
import { BigNumber, utils, constants } from 'ethers';  // v5.5.0
const { parseUnits, randomBytes, splitSignature, arrayify } = utils;
import { Action, QuoteRequest } from '../gen/valorem/trade/v1/rfq_pb';  // generated from rfq.proto
import IValoremOptionsClearinghouse from '../../abi/IValoremOptionsClearinghouse.json';
import IERC20 from '../../abi/IERC20.json';
import { EthSignature } from '../gen/valorem/trade/v1/types_pb';  // generated from types.proto
import { 
  Order, 
  SignedOrder, 
  ConsiderationItem, 
  OfferItem, 
  OrderType, 
  ItemType 
} from '../gen/valorem/trade/v1/seaport_pb';  // generated from seaport.proto
import { fromH256 } from './lib/fromHToBN';  // library script for H number conversions
import { toH160, toH256 } from './lib/fromBNToH';  // library script for H number conversions

const USDC_ADDRESS = '0x8AE0EeedD35DbEFe460Df12A20823eFDe9e03458';  // our mock USDC on Arb Goerli
const WETH_ADDRESS = '0x618b9a2Db0CF23Bb20A849dAa2963c72770C1372';  // our mock Wrapped ETH on Arb Goerli

const clearinghouseContract = new Contract(VALOREM_CLEAR_ADDRESS, IValoremOptionsClearinghouse, provider);
const wethContract = new Contract(WETH_ADDRESS, IERC20, provider);

async function constructQuoteResponse(quoteRequest: QuoteRequest) {
  /* Construct the signed Seaport offer for a quote request and wrap in a quote response */
  if (!quoteRequest.identifierOrCriteria) {
    console.log('Skipping Quote Request because "identifierOrCriteria" is undefined.');
  };
  const optionId = fromH256(quoteRequest.identifierOrCriteria);

  if (quoteRequest.action !== Action.BUY) { 
    console.log('Skipping Quote Request because only responding to buy requests.');
    return;
  };

  if (!quoteRequest.amount) {
    console.log('Skipping Quote Request because "amount" is undefined.');
  };
  const optionAmount = fromH256(quoteRequest.amount);

  // get option info
  const optionInfo = await clearinghouseContract.option(optionId);
  if (optionInfo.underlyingAsset !== WETH_ADDRESS) {
    console.log('Skipping Quote Request because only responding to WETH options.');
    return;
  };
  console.log('Responding to quote request to buy', optionAmount.toString(), 'WETH options with ID', optionId.toString());
  console.log('Option info:');
  console.log(optionInfo);

  // approve clearing house transfer of underlying asset
  const totalUnderlyingAmount = optionInfo.underlyingAmount.mul(optionAmount).mul(10);    
  let txReceipt = await (await wethContract.connect(signer).approve(VALOREM_CLEAR_ADDRESS, totalUnderlyingAmount)).wait();
  if (txReceipt.status == 0) { 
    console.log('Skipping responding to RFQ; Underlying ERC20 approval failed.');
    return;
  };

  // write option with clearing house
  txReceipt = await (await clearinghouseContract.connect(signer).write(optionId, optionAmount)).wait();
  if (txReceipt.status == 0) { 
    console.log('Skipping responding to RFQ; Writing option with clearing house failed.');
    return;
  };

  // Construct Seaport offer:
  // Note we use Seaport v1.1; see https://github.com/ProjectOpenSea/seaport/blob/seaport-1.1/docs/SeaportDocumentation.md

  // option we are offering
  const offerItem = {
    itemType: ItemType.ERC1155,  // see Seaport ItemType enum
    token: VALOREM_CLEAR_ADDRESS,
    identifierOrCriteria: optionId,
    startAmount: fromH256(quoteRequest.amount),
    endAmount: fromH256(quoteRequest.amount),         
  };
  // price we want for the option
  const USDCprice = parseUnits('100', 6);  // 100 USDC
  const considerationItem = {
    itemType: ItemType.ERC20,
    token: USDC_ADDRESS,
    startAmount: USDCprice.toString(),
    endAmount: USDCprice.toString(),
    recipient: signer.address,
    identifierOrCriteria: BigNumber.from(0),
  };

  const now = (await provider.getBlock(await provider.getBlockNumber())).timestamp;
  const in_2_mins = now + 2 * 60;  // offer expires in 2 minutes
  const salt = `0x${Buffer.from(randomBytes(8)).toString('hex').padStart(64, '0')}`

  const orderComponents = {
    offerer: signer.address,
    zone: constants.AddressZero,
    offer: [ offerItem ],
    consideration: [ considerationItem ],
    orderType: OrderType.FULL_OPEN,
    startTime: now,
    endTime: in_2_mins,
    zoneHash: constants.HashZero,
    salt: salt,
    conduitKey: constants.HashZero,
  };

  // create order signature
  const ORDER_TYPES = {
    OrderComponents: [
      { name: 'offerer', type: 'address' },
      { name: 'zone', type: 'address' },
      { name: 'offer', type: 'OfferItem[]' },
      { name: 'consideration', type: 'ConsiderationItem[]' },
      { name: 'orderType', type: 'uint8' },
      { name: 'startTime', type: 'uint256' },
      { name: 'endTime', type: 'uint256' },
      { name: 'zoneHash', type: 'bytes32' },
      { name: 'salt', type: 'uint256' },
      { name: 'conduitKey', type: 'bytes32' },
      { name: 'counter', type: 'uint256' },
    ],
    OfferItem: [
      { name: 'itemType', type: 'uint8' },
      { name: 'token', type: 'address' },
      { name: 'identifierOrCriteria', type: 'uint256' },
      { name: 'startAmount', type: 'uint256' },
      { name: 'endAmount', type: 'uint256' },
    ],
    ConsiderationItem: [
      { name: 'itemType', type: 'uint8' },
      { name: 'token', type: 'address' },
      { name: 'identifierOrCriteria', type: 'uint256' },
      { name: 'startAmount', type: 'uint256' },
      { name: 'endAmount', type: 'uint256' },
      { name: 'recipient', type: 'address' },
    ],
  };

  // see https://docs.ethers.org/v5/api/signer/#Signer-signTypedData
  const signature = await signer._signTypedData(
    {},  // domain data is optional
    ORDER_TYPES,
    orderComponents
  );
  // Use EIP-2098 compact signatures to save gas.
  const splitSig = splitSignature(signature);
  const ethSignature = new EthSignature({
    r: arrayify(splitSig.r),
    s: arrayify(splitSig.s),
    v: arrayify(splitSig.v),
  });

  // convert order fields to H types
  const offerItem_H = new OfferItem({
    itemType: offerItem.itemType,
    token: toH160(offerItem.token),
    identifierOrCriteria: toH256(offerItem.identifierOrCriteria),
    startAmount: toH256(offerItem.startAmount),
    endAmount: toH256(offerItem.endAmount),
  });
  const considerationItem_H = new ConsiderationItem({
    itemType: considerationItem.itemType,
    token: toH160(considerationItem.token),
    identifierOrCriteria: toH256(considerationItem.identifierOrCriteria),
    startAmount: toH256(considerationItem.startAmount),
    endAmount: toH256(considerationItem.endAmount),
    recipient: toH160(considerationItem.recipient),
  });
  const order_H = new Order({
    offerer: toH160(orderComponents.offerer),
    zone: toH160(orderComponents.zone),
    offers: [offerItem_H],
    considerations: [considerationItem_H],
    orderType: orderComponents.orderType,
    startTime: toH256(orderComponents.startTime),
    endTime: toH256(orderComponents.endTime),
    zoneHash: toH256(orderComponents.zoneHash),
    salt: toH256(orderComponents.salt),
    conduitKey: toH256(orderComponents.conduitKey),
  });

  const signedOrder_H = new SignedOrder({
    parameters: order_H,
    signature: ethSignature,
  });

  // construct a quote response
  const quoteResponse = new QuoteResponse({
    ulid: quoteRequest.ulid,
    makerAddress: toH160(signer.address),
    order: signedOrder_H,
  });

  return quoteResponse;
};
```