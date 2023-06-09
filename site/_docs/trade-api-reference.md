---
date: 2023-06-07 00:00:00
title: Trade API Reference
description: Reference documentation for the Valorem Trade API. 
---

The Valorem Trade API allows options market makers and takers to trade options via
an RFQ (Request for Quote) service authenticated via [SIWE (Sign-In with Ethereum)](https://docs.login.xyz/general-information/siwe-overview).
It is comprised of two services: the Auth Service and the RFQ Service.
The Authentication Service enables users to authenticate themselves and obtain 
the necessary credentials to access the other services provided by the API. The 
RFQ Service allows authenticated takers to request quotes from makers, and 
authenticated makers to return signed offers. Takers can then execute those 
signed offers via the Seaport smart contracts.

The complete protobuf definitions can be found
in [this repository](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay).

## Makers and Takers

There are two principal user roles in the Valorem Trade API:

- **Maker**: Makers are users who signed offers in response to a request for quote.
  They are responsible for fulfilling orders when a taker agrees to their quotes.

- **Taker**: Takers are users who request quotes from makers and optionally 
  execute signed offers via the Seaport smart contracts.

## Auth Service

The Authentication Service in Valorem Trade API enables users to authenticate 
themselves via SIWE and obtain the necessary credentials to access the other 
services provided by the API.

```protobuf
service Auth {
  ...
}
```

### Methods

#### `Nonce`

Returns an [EIP-4361](https://eips.ethereum.org/EIPS/eip-4361) nonce for the session and invalidates any existing session.

```protobuf
rpc Nonce (Empty) returns (NonceText);
```

**Request**

```protobuf
message Empty {}
```

**Response**

```protobuf
message NonceText {
  string nonce = 1;
}
```

- `nonce` (string): The generated nonce string.

#### `Verify`

Verifies an [EIP-191](https://eips.ethereum.org/EIPS/eip-191) signed message 
following the SIWE format and returns the verified address.

```protobuf
rpc Verify (VerifyText) returns (H160);
```

**Request**

```protobuf
message VerifyText {
  string body = 1;
}
```

- `body` (string): JSON-encoded string of the `SignedMessage` structure.

**Response**

```protobuf
message H160 {}
```

- (Response does not contain any additional fields)

#### `Authenticate`

Checks if a given connection is authenticated and returns the authenticated address for a nonce cookie.

```protobuf
rpc Authenticate (Empty) returns (H160);
```

**Request**

```protobuf
message Empty {}
```

**Response**

```protobuf
message H160 {}
```

- (Response does not contain any additional fields)

## RFQ Service

The RFQ (Request for Quote) Service in Valorem Trade API allows authenticated users to request quotes from makers and
execute trades on the Seaport exchange.

### Authentication

Only authenticated users can access the RFQ service. Users need to obtain authentication credentials from the
Authentication Service before making RFQ requests.

### Methods

#### `Taker`

Request quotes from makers via a stream of `QuoteRequest` messages and receive a stream of `QuoteResponse` messages.

```protobuf
rpc Taker (stream QuoteRequest) returns (stream QuoteResponse);
```

**Request**

```protobuf
message QuoteRequest {
  H128 ulid = 1;
  H160 taker_address = 2;
  ItemType item_type = 3;
  H160 token_address = 4;
  H256 identifier_or_criteria = 5;
  H256 amount = 6;
  Action action = 7;
}
```

- `ulid` (H128, optional): The unique identifier for the quote request.
- `taker_address` (H160, optional): The address of the taker.
- `item_type` (ItemType): The type of item for which a quote is being requested.
- `token_address` (H160, optional): The token address for which a quote is being requested.
- `identifier_or_criteria` (H256, optional): The identifier or criteria for the item.
- `amount` (H256): The amount of the item.
- `action` (Action): The action (BUY or SELL) for the quote request.

**Response**

```protobuf
message QuoteResponse {
  H128 ulid = 1;
  H160 maker_address = 2;
  SignedOrder order = 3;
}
```

- `ulid` (H128): The unique identifier for the quote request.
- `maker_address` (H160): The address of the maker making the offer.
- `order` (SignedOrder): The order and signature from the maker.

#### `Maker`

Send quotes to takers via a stream of `QuoteResponse` messages and receive a stream of `QuoteRequest` messages.

```protobuf
rpc Maker (stream QuoteResponse) returns (stream QuoteRequest);
```

**Request**

```protobuf
message QuoteResponse {
  H128 ulid = 1;
  H160 maker_address = 2;
  SignedOrder order = 3;
}
```

- `ulid` (H128): The unique identifier for the quote request.
- `maker_address` (H160): The address of the maker making the offer.
- `order` (SignedOrder): The order and signature from the maker.

**Response**

```protobuf
message QuoteRequest {
  H128 ulid = 1;
  H160 taker_address = 2;
  ItemType item_type = 3;
  H160 token_address = 4;
  H256 identifier_or_criteria = 5;
  H256 amount = 6;
  Action action = 7;
}
```

- `ulid` (H128, optional): The unique identifier for the quote request.
- `taker_address` (H160, optional): The address of the taker.
- `item_type` (ItemType): The type of item for which a quote is being requested.
- `token_address` (H160, optional): The token address for which a quote is being requested.
- `identifier_or_criteria` (H256, optional): The identifier or criteria for the item.
- `amount` (H256): The amount of the item.
- `action` (Action): The action (BUY or SELL) for the quote request.

#### `WebTaker`

Request

quotes from makers via a single `QuoteRequest` message and receive a stream of `QuoteResponse` messages for use by
gRPC-web clients.

```protobuf
rpc WebTaker (QuoteRequest) returns (stream QuoteResponse);
```

**Request**

```protobuf
message QuoteRequest {
  H128 ulid = 1;
  H160 taker_address = 2;
  ItemType item_type = 3;
  H160 token_address = 4;
  H256 identifier_or_criteria = 5;
  H256 amount = 6;
  Action action = 7;
}
```

- `ulid` (H128, optional): The unique identifier for the quote request.
- `taker_address` (H160, optional): The address of the taker.
- `item_type` (ItemType): The type of item for which a quote is being requested.
- `token_address` (H160, optional): The token address for which a quote is being requested.
- `identifier_or_criteria` (H256, optional): The identifier or criteria for the item.
- `amount` (H256): The amount of the item.
- `action` (Action): The action (BUY or SELL) for the quote request.

**Response**

```protobuf
message QuoteResponse {
  H128 ulid = 1;
  H160 maker_address = 2;
  SignedOrder order = 3;
}
```

- `ulid` (H128): The unique identifier for the quote request.
- `maker_address` (H160): The address of the maker making the offer.
- `order` (SignedOrder): The order and signature from the maker.