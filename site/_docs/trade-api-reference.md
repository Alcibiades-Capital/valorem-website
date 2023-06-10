---
date: 2023-06-07 00:00:00
title: Trade API Reference
description: Reference documentation for the Valorem Trade API. 
---

The Valorem Trade API enables peer-to-peer, signature based, noncustodial
digital asset trading via low latency gRPC protobuf `proto3` interfaces, with order
settlement via the [Seaport smart contracts](https://github.com/ProjectOpenSea/seaport).
The complete protobuf definitions can be found
in [this repository](https://github.com/valorem-labs-inc/exchange-proto/blob/main/quay).

## User roles

There are two principal user roles in the Valorem Trade API:

- **Maker**: Makers are users who signed offers in response to a request for quote.
  They are responsible for fulfilling orders when a taker agrees to their quotes.

- **Taker**: Takers are users who request quotes from makers and optionally
  execute signed offers via the Seaport smart contracts.

## API overview

The Valorem Trade API is composed of an RFQ (request-for-quote) service, and an Auth service
using [SIWE (Sign-In with Ethereum)](https://docs.login.xyz/general-information/siwe-overview).
The Auth service enables users to authenticate themselves and obtain the necessary
credentials to access the other services provided by the API. The RFQ service
allows authenticated takers to request quotes from makers, and authenticated
makers to return signed offers. Takers can then execute those signed offers
via Seaport.

## Primitive data types

The trade API defines some primitive data types mirroring the Ethereum ABI:

`H40`: A 40 bit data type

```protobuf
message H40 {
  uint32 hi = 1;
  // Note: lo is really a uint8, however the closest type in Protocol Buffers is uint32. Parsing needs
  //       to take this into consideration.
  uint32 lo = 2;
}
```

`H96`: A 96 bit data type

```protobuf
message H96 {
  uint64 hi = 1;
  uint32 lo = 2;
}
```

`H128`: A 128 bit data type

```protobuf
message H128 {
  uint64 hi = 1;
  uint64 lo = 2;
}
```

`H160`: A 160 bit data type

```protobuf
message H160 {
  H128 hi = 1;
  uint32 lo = 2;
}
```

`H256`: A 256 bit data type

```protobuf
message H256 {
  H128 hi = 1;
  H128 lo = 2;
}
```

`Empty`: an empty message type

```protobuf
message Empty {}
```

`EthSignature`: An Ethereum signature

ECDSA signatures in Ethereum consist of three parameters: v, r and s. The signature is always 65-bytes in length.

- r = first 32 bytes of signature
- s = second 32 bytes of signature
- v = final 1 byte of signature

Because protobuf doesn't support uint8, we use a boolean for v, which is always 8 bits.

```protobuf
message EthSignature {
  bytes r = 1;
  bytes s = 2;
  bytes v = 3;
}
```

## Auth service

The Authentication Service in Valorem Trade API enables users to authenticate
themselves via SIWE, and receive the necessary credentials to access the other
services provided by the API. The Auth service uses session cookies to store
authentication information. The session cookies are implemented
using [axum-sessions](https://docs.rs/axum-sessions/latest/axum_sessions/),
and all information is verified server-side. This provides compatibility with
both broswer and non-browser clients out of the box.

```protobuf
service Auth {
    ...
}
```

### Cookie format

### Methods

#### `Nonce`

Returns an [EIP-4361](https://eips.ethereum.org/EIPS/eip-4361) nonce for the
session and invalidates any existing session. This method resets session cookie,
which is passed back on the request.

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

- `nonce` (string): a randomized token typically chosen by the Trade API, and
  used to prevent replay attacks, at least 8 alphanumeric characters UTF-8 encoded.

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

- `body` (string): a JSON-encoded ECDSA signature following the SIWE format.
  ECDSA signatures in Ethereum consist of three parameters: v, r and s.
  The signature is always 65-bytes in length.
    - r = first 32 bytes of signature
    - s = second 32 bytes of signature
    - v = final 1 byte of signature

**Response**

The response is the verified 160-bit address as an `H160`.

```protobuf
message H160 {
}
```

#### `Authenticate`

Checks if a given connection is authenticated and returns the authenticated
address for a session cookie.

```protobuf
rpc Authenticate (Empty) returns (H160);
```

**Request**

```protobuf
message Empty {}
```

**Response**

The response is the verified 160-bit address as an `H160`.

```protobuf
message H160 {
}
```

## RFQ service

The RFQ (Request for Quote) Service in Valorem Trade API allows authenticated
takers to request quotes from makers, for those makers to respond with signed
offers, and for those traders to receive those signed offers for executing
trades on the Seaport smart contracts.

```protobuf
service RFQ {
    ...
}
```

### Authentication

Only authenticated users can access the RFQ service.

### Methods

#### `Taker`

Request quotes from makers via a stream of `QuoteRequest` messages and receive
a stream of `QuoteResponse` messages.

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