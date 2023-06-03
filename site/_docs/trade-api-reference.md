---
date: 2023-05-29 00:00:00
title: Trade API Reference
description: Developer documentation for the Valorem Exchange API.
---

## Introduction

Valorem Trade is an expanding ecosystem of high-performance options trading infrastructure. The API currently supports RFQ (Request for Quote) services and returns signed offers for execution on [Seaport](https://docs.opensea.io/reference/seaport-overview). We utilise the [gRPC](https://grpc.io/docs/what-is-grpc/introduction/) framework and use protocol buffers for data serialization for fast and efficient communication.

### API Endpoint

`https://exchange.valorem.xyz/`


### Proto Files

[rfq.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/rfq.proto)

[seaport.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/seaport.proto)

[session.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/session.proto)

[types.proto](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay/types.proto)

 

## Example Usage

We will use [Connect-Node](https://connect.build/docs/node/getting-started) to make requests to the gRPC API in the following examples.


## RFQ Taker



### 1. Connect and authenticate with Quay





```javascript


  const client = new quay.session.SessionClient(
    new tonic.transport.Channel(quay_uri, {
      transport: tonic.transport.HttpTransport
      })
  );
  const nonce_response = await client.nonce(new quay.session.Empty());

  // Fetch the session cookie for all future requests
  const session_cookie = nonce_response.metadata[settings.session_cookie_key];
  const nonce = nonce_response.nonce;

  // Verify & authenticate with Quay before connecting to RFQ endpoint.
  const client_auth = new quay.session.SessionClient(
    new tonic.transport.Channel(quay_uri, {
      transport: tonic.transport.HttpTransport
    }),
    new quay.utils.session_interceptor.SessionInterceptor(session_cookie)
  );

  // Create a sign in with ethereum message
  const message = new siwe.Message({
    domain: new siwe.Domain('localhost.com'),
    address: wallet.address,
    statement: null,
    uri: new siwe.Uri('http://localhost/'),
    version: siwe.Version.V1,
    chain_id: provider.getNetwork().chainId,
    nonce,
    issued_at: siwe.TimeStamp.fromDate(new Date()),
    expiration_time: null,
    not_before: null,
    request_id: null,
    resources: [],
  });

  // Generate a signature
  const message_string = message.toString();
  const signature = await wallet.signMessage(message_string);

  // Create the SignedMessage
  const signature_string = signature.toString();
  const signed_message = {
    signature: signature_string,
    message: message_string,
  };

  // Verify the session with Quay
  try {
    await client_auth.verify(new quay.session.VerifyText({ body: JSON.stringify(signed_message) }));
  } catch (error) {
    console.error('Error: Unable to verify client. Reported error:');
    console.error(error);
    process.exit(1);
  }

  // Check that we have an authenticated session
  try {
    await client_auth.authenticate(new quay.session.Empty());
  } catch (error) {
    console.error('Error: Unable to check authentication with Quay. Reported error:');
    console.error(error);
    process.exit(1);
  }

```


### 2. Create an Option Type

```


```


### 3. Request a buy quote from the Maker


### 4. If the Maker offers a quote:


### 5. Accept and fulfill the Order from the Maker


### 6. Request a sell quote from the Maker


### 7. Accept and fulfill the Order from the Maker






## RFQ Maker
