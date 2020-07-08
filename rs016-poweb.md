---
permalink: /RS-016
---
# PoWeb: Parcel Delivery over HTTP and WebSockets
{: .no_toc }

- Id: RS-016.
- Status: Working draft.
- Type: Implementation.
- Proof of concept: https://github.com/relaynet/poc/tree/master/PoWebSocket

## Abstract
{: .no_toc }

This document describes _PoWebSockets_, a binding for [Parcel Delivery Connections (PDC)](rs000-core.md#parcel-delivery-binding) on top of the HTTP and [WebSockets (RFC-6455)](https://tools.ietf.org/html/rfc6455) protocol.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

PoWeb establishes local and Internet-based PDCs where remote procedure calls are done over HTTP and streams are delivered over WebSockets.

The messages sent over WebSockets are serialized with [Protocol Buffers](https://developers.google.com/protocol-buffers/).

## Handshake

Per Relaynet Core, the handshake involves three steps:

1. The gateway _challenges_ the endpoint to sign a nonce with its key(s).
1. The endpoint signs the nonce with each of its keys and sends the signatures to the gateway.
1. The gateway verifies the signatures and confirms the end of the handshake.

These messages are serialized using the following Protocol Buffers definition:

```proto
syntax = "proto3";

package relaynet.powebsockets.handshake;

message Challenge {
    // Sent by the gateway at the start of the connection

    string gateway_nonce = 1;
}

message Response {
    // Sent by the endpoint in response to Challenge

    // The gateway's nonce signed by each endpoint certificate.
    repeated bytes gateway_nonce_signatures = 1;
}

message Complete {
    // Sent by the gateway after successfully validating Response

    bool success = 1;
}
```

## Operations

The connection can be used to exchange parcels and perform other actions as soon as the handshake is completed successfully.

### Parcel Collection Request

The endpoint will send a _parcel collection request_ message when it is ready to collect parcels from the gateway. Per Relaynet Core, the gateway would not attempt to deliver parcels until then.

### Parcel Delivery

To deliver a parcel, the sending node MUST encapsulate it in a _parcel delivery_ message. The receiving node MUST respond with a _parcel delivery acknowledgement_ once the parcel has been safely stored/forwarded.

When the gateway does not have any further parcels to deliver, it MUST send a _parcel delivery complete_ message. This may be useful in cases where the application behind the endpoint is short-lived and is meant to exit as soon as it has collected all the parcel from the gateway.

### Endpoint Certificate Request

To request a certificate from the gateway, the endpoint MUST send an _endpoint certificate request_ message with the endpoint's public key attached to it. This is analogous to a [Certificate Signing Request](https://en.wikipedia.org/wiki/Certificate_signing_request).

Once the certificate has been generated, the server MUST attach it to an _endpoint certificate response_ message.

### Parcel Delivery Deauthorization

To revoke all or some PDAs, the endpoint MUST send a _parcel delivery deauthorization_ message with the fields specified in [Relaynet Core](rs000-core.md#pdd).

## Message Serialization

The following Protocol Buffers file defines the messages in this connection:

```proto
syntax = "proto3";

package relaynet.powebsockets;

import "google/protobuf/timestamp.proto";

message ParcelDelivery {
    string id = 1;
    bytes parcel = 2;
}

message ParcelDeliveryAck {
    string id = 1;
}

message ParcelCollectionRequest {}

message ParcelDeliveryComplete {}

message EndpointCertificateRequest {
    bytes public_key = 1;
}

message EndpointCertificateResponse {
    bytes certificate = 1;
}

message ParcelDeliveryDeauthorization {
    string endpoint_address = 1;
    repeated string pda_serial_numbers = 2;
    google.protobuf.Timestamp expiry = 3;
}
```

And since Protocol Buffers messages do not contain any information to identify their message type, each message MUST be prefixed with a corresponding 8-bit integer to specify its type:

- `0x0`: `ParcelDelivery`.
- `0x1`: `ParcelDeliveryAck`.
- `0x2`: `ParcelCollectionRequest`.
- `0x3`: `ParcelDeliveryComplete`.
- `0x4`: `EndpointCertificateRequest`.
- `0x5`: `EndpointCertificateResponse`.
- `0x6`: `ParcelDeliveryDeauthorization`.

## WebSocket Considerations

Clients and servers implementing this specification MUST support HTTP version 1.1.

Both the client and the server SHOULD send each other _ping_ messages and they MUST respond with a _pong_ message for every ping they receive.

## TLS Considerations

[Server Name Identification](https://en.wikipedia.org/wiki/Server_Name_Indication) MUST be supported by clients and MAY be used by servers.

## Relevant Specifications

[Relaynet Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoWeb. [Relaynet PKI (RS-002)](rs002-pki.md) is also particularly relevant to this specification.

## Open Questions

- Should this spec be future-proof with regards to HTTP/2 support in WebSockets?
- Should [close codes](https://www.iana.org/assignments/websocket/websocket.xml#close-code-number) other than `1000` (Normal Closure) be used? Note that [there is a question on errors in general in Relaynet Core](rs000-core.md#open-questions).
- Should there be a required or recommended frequency for ping messages?
