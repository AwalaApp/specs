---
permalink: /RS-014
---
# Ping Service, Version 1
{: .no_toc }

- Id: RS-014.
- Status: Working draft.
- Type: Implementation.

## Abstract
{: .no_toc }

This document describes _Ping_, a trivial service used to test the underlying implementation and deployment of Relaynet, thus mitigating integration issues.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

Relaynet consists of endpoints, gateways and relayers, which may run on different operating systems, on different hardware and in different physical environments. These components support different messaging protocols and message transport bindings, and may be built and run by different organizations.

This specification mitigates the integration issues that could arise from the implementation and deployment of Relaynet by offering a very simple service that can be used to test that applications can exchange messages.

Ping is a trivial service: Given applications A and B, application A sends a _ping message_ to application B, to which application B has to respond with a _pong message_.

## Transactions

A _transaction_ is initiated with a ping message and is completed when the pong message reaches its destination.

The two applications MAY continue to run subsequent transactions. This could be useful to test the stability of the channel and the underlying session provided by the [Channel Session Protocol](rs003-key-agreement.md).

## Messages

This service employs the following messages.

### Ping

This is the message that initiates a transaction. Its type MUST be `application/vnd.relaynet.ping-v1.ping`, and its payload MUST be a binary stream with the following structure:

1. The ping id: A sequence of exactly 36 octets. It SHOULD be a UUID4 value.
1. The DER serialization of the [Parcel Delivery Authorization](rs002-pki.md#parcel-delivery-authorization-pda) (PDA) to use to reply with a pong message.

### Pong

This message MUST be sent by the application receiving a ping message. Its type MUST be `application/vnd.relaynet.ping-v1.pong`. The payload MUST be a sequence of 36 octets representing the ping id.
