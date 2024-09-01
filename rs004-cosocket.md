---
nav_exclude: true
permalink: /RS-004
---
# Cargo Relay over TCP/Unix Sockets (CoSocket)
{: .no_toc }

- Id: RS-004.
- Status: Abandoned.
- Type: Implementation.
- Proof of concept: https://github.com/AwalaApp/poc/tree/master/CoSocket

## Abstract
{: .no_toc }

This document describes CoSocket, a [cargo relay binding](rs000-core.md#cargo-relay-binding) on top of TCP or Unix sockets. As a purpose-built [Application Layer](https://en.wikipedia.org/wiki/Application_layer) protocol, this is the most efficient binding to relay cargo.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

As a cargo relay binding, CoSocket's objective is to provide the basis for a gateway to exchange cargo with a courier or another gateway.

Gateways and couriers can act as client and servers. One of them has to be the server so that the other can connect to it via TCP or a Unix socket, but once communication has been established, they become peers and can send [packets](#packets) to each other indistinctively.

CoSocket is a [binary protocol](https://en.wikipedia.org/wiki/Binary_protocol) with [little-endian](https://en.wikipedia.org/wiki/Endianness#Little-endian) byte order.

## Primitives

The packets in this protocol use the following primitives:

- _Varchar_: A UTF-8 string, length-prefixed with a 8-bit unsigned integer (1 octet). Consequently, the string can have a length of up to 255 octets.
- _Payload length_: A 32-bit unsigned integer (4 octets), used as a length-prefix for a payload.

## Packets

Each packet starts with a tag that identifies the type of packet. The tag itself is serialized as a 16-bit unsigned integer (2 octets).

### Handshake Packets

Per Awala Core, the handshake involves three steps:

1. The courier _challenges_ the gateway to sign a nonce with its key(s).
1. The gateway signs the nonce with each of its keys and sends the signatures to the courier.
1. The courier verifies the signatures and confirms the end of the handshake.

#### Handshake Challenge

This packet contains the nonce that the gateway has to sign. The packet comprises the following sequence:

- Tag: `0xf0`.
- The nonce, as a random sequence of exactly 32 octets.

#### Handshake Response

This packet contains the signatures for the nonce and comprises the following sequence:

- Tag: `0xf1`.
- The total _payload length_ for all the signatures and their length prefixes included in the packet.
- The payload with the sequence of signatures, where each is prefixed with its payload length.

#### Handshake Complete

This packet is sent by the courier when the signatures were successfully verified. This packet is empty -- It only contains its tag (`0xf2`).

### Operation Packets

#### Cargo Collection Request

This packet encapsulates a [Cargo Collection Authorization (CCA)](rs000-core.md#cca) and represents a request to collect cargo for a specific gateway.

A courier MUST send this packet to a gateway to indicate it is ready to receive cargo and to prove it is authorized to receive cargo for the gateway in the CCA.

The packet comprises the following sequence:

1. Tag: `0x00`.
1. The _payload length_ for the CCA.
1. The CCA.

#### Cargo Delivery

This packet encapsulates a cargo and has the following sequence:

1. Tag: `0x01`.
1. A string that uniquely identifies this cargo delivery, serialized as a varchar. This MAY be different from the cargo id, so it could be a [UUID4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) value, for example.
1. The payload length for the cargo.
1. The cargo.

#### Cargo Delivery (File Descriptor Version)

This is an alternative to [Cargo Delivery](#cargo-delivery) using file descriptors, so this is only available on Unix sockets. The sequence is as follows:

1. Tag: `0x02`.
1. The string that uniquely identifies this cargo delivery, serialized as a varchar.
1. (TODO: Define how the file descriptor is actually passed)

#### Cargo Collection

This packet represents an acknowledgement that a cargo delivery was successfully received and has the following sequence:

1. Tag: `0x03`.
1. The cargo delivery id, serialized as varchar.

#### Cargo Delivery Completion

Used to signal that there are no further cargoes for the specified gateway. It has the following sequence:

1. Tag: `0x04`.
1. The address of the gateway for which there are no further cargoes, serialized as varchar.

The sender of this packet MUST also quit if its peer has already confirmed that it will not send any further cargoes.

#### Quit

Used by a peer to indicate that it is about to close the connection. The sender MUST close the connection immediately after sending this packet.

The packet has the following sequence:

1. Tag: `0xff`.
1. Reason: A 16-bit unsigned integer representing the reason why the connection was terminated:
   - `0x00`: Operation completed without errors.
   - `0x01`: Invalid packet.
   - `0x02`: Handshake error.
   - `0x03`: Quota reached.
   - `0x04`: Internal error.

## Persistent Connections

Two gateways MAY maintain a persistent connection to exchange cargo in near-real time. This could be necessary when the target gateway is not a server that can be reached by the other gateway (e.g., the target is behind a [NAT gateway](https://en.wikipedia.org/wiki/Network_address_translation)).

Couriers MUST always quit the connection as soon as no further cargoes are expected in either direction.

## Examples

### Courier as TCP Server

TODO: Upload sequence diagram.

### Courier as Unix Socket Client

TODO: Upload sequence diagram.

### Cargo Forwarding

TODO: Upload sequence diagram.

## Relevant Specifications

[Awala Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [cargo relay bindings](rs000-core.md#cargo-relay-binding) specifically, all of which apply to CoSocket. Amongst other things, it defines the use Transport Layer Security (TLS).
