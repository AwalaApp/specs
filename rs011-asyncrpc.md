---
nav_exclude: true
permalink: /RS-011
---
# AsyncRPC: RPC Encapsulation in Awala
{: .no_toc }

- Id: RS-011.
- Status: Working draft.
- Type: Implementation.

## Abstract
{: .no_toc }

_AsyncRPC_ is a Awala service whose messages are [Application Layer](https://en.wikipedia.org/wiki/Application_layer) packets such as HTTP requests, HTTP responses, SMTP messages, amongst others. Its purpose is to facilitate the integration of Awala in pre-existing, centralised services by keeping the server-side unchanged with the use of a server-side adapter.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

## Encapsulated Protocols

### HTTP

Messages:

- `HTTPv1Request`.
- `HTTPv1Response`.
- `HTTPv2Request`.
- `HTTPv2ResponseSet`.

### SMTP

Messages:

- `SMTPMessage`.
- `SMTPServerResponse`.

### POP3

Messages:

- `POP3Stat`.
- `POP3List`.
- `POP3Retrieve`. Retrieves only the specified email(s).
- `POP3RetrieveNew`. Retrieves only the emails that haven't been retrieved (but requires listing all the emails that have been retrieved already).
- `POP3RetrieveAll`.
- `POP3RetrieveAllThenDelete`.
- `POP3Delete`. Deletes the specified email(s).

### TCP Dump

Connects to the specified netloc over TCP, dumps the specified payload, saves any output from the server and finally waits for the server to close the connection (unless the specified timeout is reached). The output from the server is returned to the origin application as another message.

Messages:

- `TCPDump`.

### UDP Dump

Connects to the specified netloc over UDP and dumps the specified payload.

Messages:

- `UDPDump`.

## TLS and DTLS

(D)TLS can be enforced. Server certificate validation can't be disabled, but a custom certificate store can be specified.

The client may optionally request that the following data be returned in order to check that the server indeed returned the specified response:

- All the relevant parameters established in the TLS handshake. For example, the server's certificate chain, nonces and the master secret.
- The raw response signed by the server.

(Inspired by [TLSNotary](https://tlsnotary.org/))

Client-side certificates can be specified.
