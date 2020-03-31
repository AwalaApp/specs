---
permalink: /RS-008
---
# CogRPC: Cargo Relay over gRPC
{: .no_toc }

- Id: RS-008.
- Status: Draft.
- Type: Implementation.

## Abstract
{: .no_toc }

This document describes CogRPC (pronounced _Co-Jee-Arr-Pee-See_), a [cargo relay binding](rs000-core.md#cargo-relay-binding) on top of [gRPC](https://grpc.io/).

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

As a cargo relay binding, CogRPC's objective is to establish a Cargo Relay Connection (CRC) between a courier and a gateway so they can exchange cargo.

When a CRC is established between a private gateway and a courier, the gateway and the courier act as the client and the server, respectively. Similarly, when a CRC is established between a public gateway and a courier, they act as the server and the client, respectively.

This binding defines multiple message types and RPCs under a single gRPC service called `CargoRelay`. The gRPC interface described by this binding is defined in full in the form of Protocol Buffers at the end of this document.
 
The client is responsible for initiating the delivery and collection of parcels using the corresponding RPC. Note that per [RS-000](./rs000-core.md), it is recommended that the client sends the cargo before attempting to collect any cargo.

## Remote Procedure Calls (RPCs) {#rpcs}

The messages sent to and received from the gRPC services below MUST be serialized with Protocol Buffers v3.

### `DeliverCargo`

This bidirectional streaming RPC MUST be used to deliver cargo to the server. The client MUST send zero or more [RAMF](rs001-ramf.md)-serialized cargo messages and the server MUST acknowledge the receipt of each cargo per the requirements and recommendations in RS-000.

The cargo sent to the server MAY originate in different gateways.

### `CollectCargo`

This bidirectional streaming RPC MUST be used to collect cargo from the server. The server MUST send zero or more [RAMF](rs001-ramf.md)-serialized cargo messages and the client MUST acknowledge the receipt of each cargo per the requirements and recommendations in RS-000.

This call MUST be authenticated by setting the `Authorization` metadata to the string `Relaynet-CCA ${crcBase64}`, where `${crcBase64}` is the Base64-encoded serialization of a valid [Cargo Collection Authorization (CCA)](./rs000-core.md#cca). As a consequence, each call is bound to exactly one target gateway.

## Acknowledgement Messages

A node sending cargo MUST attach a UUID4 to each individual message. The recipient MUST attach such identifier to each acknowledgement message.

## Protocol Buffers Package Definition

```proto
{% include_relative diagrams/rs008/cogrpc.proto %}
```

## Relevant Specifications

[Relaynet Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [cargo relay bindings](rs000-core.md#cargo-relay-binding) specifically, all of which apply to CogRPC. Amongst other things, it defines the use Transport Layer Security (TLS) or equivalent, as well as when it is safe to return acknowledgement messages.
