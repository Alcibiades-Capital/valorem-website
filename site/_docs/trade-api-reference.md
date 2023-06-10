---
date: 2023-06-07 00:00:00
title: Trade API Reference
description: Reference documentation for the Valorem Trade API. 
---

The Valorem Trade API enables peer-to-peer, signature based, noncustodial
digital asset trading via low latency gRPC protobuf `proto3` interfaces, with order
settlement via the [Seaport smart contracts](https://github.com/ProjectOpenSea/seaport).
The complete protobuf definitions can be found
in [this repository](https://github.com/valorem-labs-inc/trade-interfaces).

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

As well as a few utility types:

`Empty`: an empty message type

```protobuf
message Empty {}
```

`EthSignature`: An Ethereum signature

ECDSA signatures in Ethereum consist of three parameters: v, r and s. The signature is always 65-bytes in length.

- r = first 32 bytes of signature
- s = second 32 bytes of signature
- v = final 1 byte of signature

```protobuf
message EthSignature {
  bytes r = 1;
  bytes s = 2;
  bytes v = 3;
}
```

## Seaport data types

Valorem Trade uses the Seaport smart contracts for order settlement. This section
describes the data types used for a protobuf wire format relating to the Seaport
smart contracts.

**For a full reference on the seaport smart contracts and interfaces, see
the [Seaport documentation](https://docs.opensea.io/reference/seaport-overview).**

`ItemType`: The ItemType designates the type of item, with valid types being Ether
(or other native token for the given chain), ERC20, ERC721, ERC1155,
ERC721 with "criteria" (explained below), and ERC1155 with criteria.

```protobuf
enum ItemType {
  NATIVE = 0;
  ERC20 = 1;
  ERC721 = 2;
  ERC1155 = 3;
  ERC721_WITH_CRITERIA = 4;
  ERC1155_WITH_CRITERIA = 5;
}
```

`ConsiderationItem`: An item required in exchange for an offer.

```protobuf
message ConsiderationItem {
  ItemType item_type = 1;
  H160 token = 2; // address
  H256 identifier_or_criteria = 3; // uint256
  H256 start_amount = 4; // uint256
  H256 end_amount = 5; // uint256
  H160 recipient = 6;
}
```

`OfferItem`: An item offered in exchange for consideration.

```protobuf
message OfferItem {
  ItemType item_type = 1;
  H160 token = 2;
  H256 identifier_or_criteria = 3;
  H256 start_amount = 4;
  H256 end_amount = 5; // uint256
}
```

- `item_type`: The item_type designates the type of item.
- `token`: Designates the account of the item's token contract (with the null  
  address used for Ether or other native tokens).
- `identifier_or_criteria`: The identifier_or_criteria represents either the ERC721 or ERC1155
  token identifier or, in the case of a criteria-based item type, a
  merkle root composed of the valid set of token identifiers for
  the item. This value will be ignored for Ether and ERC20 item types,
  and can optionally be zero for criteria-based item types to allow
  for any identifier.
- `start_amount`: The start_amount represents the amount of the item in question that
  will be required should the order be fulfilled at the moment the
  order becomes active.
- `end_amount`: The end_amount represents the amount of the item in question that
  will be required should the order be fulfilled at the moment the
  order expires. If this value differs from the item's start_amount,
  the realized amount is calculated linearly based on the time elapsed
  since the order became active.

`OrderType`: designates one of four types for the order depending on two distinct preferences:

```protobuf
enum OrderType {
  FULL_OPEN = 0;
  PARTIAL_OPEN = 1;
  FULL_RESTRICTED = 2;
  PARTIAL_RESTRICTED = 3;
}
```

- `FULL` indicates that the order does not support partial fills,
  whereas `PARTIAL` enables filling some fraction of the order, with the
  important caveat that each item must be cleanly divisible by the supplied
  fraction (i.e. no remainder after division). `OPEN` indicates that the call to
  execute the order can be submitted by any account, whereas `RESTRICTED` requires 
  that the order either be executed by the offerer or the zone of the order, or 
  that a magic value indicating that the order is approved is returned upon 
  calling validateOrder on the zone.

`Order`: A Seaport order. Each order contains ten key components.

```protobuf
message Order {
  H160 offerer = 1;
  H160 zone = 2;
  repeated OfferItem offers = 3;
  repeated ConsiderationItem considerations = 4;
  OrderType order_type = 5;
  H256 start_time = 6;
  H256 end_time = 7;
  H256 zone_hash = 8;
  H256 salt = 9;
  H256 conduit_key = 10;
}
```

- `offerer`: The offerer of the order supplies all offered items and must either
  fulfill the order personally (i.e. msg.sender == offerer) or approve
  the order via signature (either standard 65-byte EDCSA, 64-byte
  EIP-2098, or an EIP-1271 isValidSignature check) or by listing the order
  on-chain (i.e. calling validate).
- `zone`: The zone of the order is an optional secondary account attached to the
  order with two additional privileges:
    - The zone may cancel orders where it is named as the zone by calling
      cancel. (Note that offerers can also cancel their own orders, either
      individually or for all orders signed with their current counter at
      once by calling incrementCounter).
    - "Restricted" orders (as specified by the order type) must either be
      executed by the zone or the offerer, or must be approved as indicated
      by a call to an validateOrder on the zone.
- `offers`: The offers array contains an array of items that may be transferred
  from the offerer's account.
- `considerations`: The consideration contains an array of items that must be received
  in order to fulfill the order. It contains all of the same components
  as an offered item, and additionally includes a recipient that will
  receive each item. This array may be extended by the fulfiller on
  order fulfillment so as to support "tipping" (e.g. relayer or
  referral payments)
- `order_type`: The order type indicates whether the order supports partial fills
  and whether the order can be executed by any account or only by the
  offerer or zone.
- `start_time`: The start_time indicates the block timestamp at which the order
  becomes active.
- `end_time`: The end_time indicates the block timestamp at which the order expires.
  This value and the startTime are used in conjunction with the
  start_amount and end_amount of each item to derive their current amount.
- `zone_hash`: The zoneHash represents an arbitrary `bytes32` value that will be
  supplied to the zone when fulfilling restricted orders that the zone
  can utilize when making a determination on whether to authorize the order.
- `salt`: The salt represents an arbitrary source of entropy for the order.
- `conduit_key`: The conduit_key is a `bytes32` value that indicates what conduit,
  if any, should be utilized as a source for token approvals when
  performing transfers. By default, i.e. when conduitKey is set to the
  zero hash, the offerer will grant ERC20, ERC721, and ERC1155 token
  approvals to Seaport directly so that it can perform any transfers
  specified by the order during fulfillment. In contrast, an offerer
  that elects to utilize a conduit will grant token approvals to the
  conduit contract corresponding to the supplied conduit key, and
  Seaport will then instruct that conduit to transfer the respective
  tokens.

`SignedOrder`: A signed order ready for execution via Seaport.

```protobuf
message SignedOrder {
  Order parameters = 1;
  EthSignature signature = 2;
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