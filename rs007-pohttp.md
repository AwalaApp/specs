# PoHTTP: Parcel Delivery over HTTP
{: .no_toc }

- Id: RS-007.
- Status: Working draft.
- Type: Implementation.

## Abstract
{: .no_toc }

This document describes PoHTTP, a [parcel delivery binding](rs000-core.md#parcel-delivery-binding) for external Parcel Delivery Connections (PDC) based on HTTP.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

PoHTTP was specifically designed to make it easier for maintainers of existing systems to support Relaynet. For example, an organization maintaining a REST API could collect parcels by implementing a new API endpoint (e.g., `/relaynet`), or they could integrate an [Enterprise Service Bus](https://en.wikipedia.org/wiki/Enterprise_service_bus) to convert Relaynet service messages to API calls.

In some cases, it may also be desirable for a relaying gateway to expose an HTTP interface so it can receive parcels from certain endpoints. For example, a Shell script in a legacy system could use `curl` to deliver parcels.

As an external PDC, the only operation supported by this binding is [parcel delivery](#parcel-delivery).

## Binding hint

The [hint](rs000-core.md#addressing) for this binding MUST be `http`. For example, `rne+http://api.example.com/relaynet` would be a valid [public endpoint](rs000-core.md#endpoint-messaging-protocol) address and it would correspond to the HTTP URL `https://api.example.com/relaynet` (note the `https` scheme).

## Parcel Delivery

To deliver each parcel, the client MUST make a `POST` request to the HTTP URL corresponding to the node address, with `application/vnd.relaynet.parcel` as the content type and the parcel as the body. If the client is a relaying gateway, it MUST provide the target endpoint with its address using the request header `X-Relaynet-Gateway`. For example, `X-Relaynet-Gateway: rng+http://gateway.humanitarian.org`.

The server MUST respond with one of the following status codes:

- `202` (Accepted) response as soon as it successfully [stores or forwards](https://en.wikipedia.org/wiki/Store_and_forward) the parcel.
- `307` (Temporary Redirect) per [RFC-7231](https://tools.ietf.org/html/rfc7231#section-6.4.7) or `308` (Permanent Redirect) per [RFC-7238](https://tools.ietf.org/html/rfc7238), in which case the client MUST repeat the `POST` request against the specified URL.
- Any other standard status code in the range 400-599 that the server regards applicable.
- A relaying gateway MAY return a `507` (Insufficient Storage) response when it no longer has the capacity to accept parcels for the target endpoint or its gateway.

## HTTP Considerations

Clients and servers implementing this specification MUST support HTTP version 1.1, and they SHOULD also support HTTP version 2.

HTTP connections MUST be sessionless, so clients and servers MUST NOT use cookies.

[Rate limiting](https://tools.ietf.org/html/rfc6585#section-4) SHOULD be supported by clients as servers MAY use it.

Headers other than the ones described in this specification MAY be used, but defining how they should be handled at the other end is outside the scope of this specification.

## TLS Considerations

Since [message transport bindings](rs000-core.md#message-transport-bindings) MAY NOT use TLS when communication happens via the loopback network interface per [Relaynet Core](rs000-core.md), clients MUST fall back to plain HTTP without TLS in that case if the server does not support TLS. This constraint also applies to redirects.

[Server Name Identification](https://en.wikipedia.org/wiki/Server_Name_Indication) MUST be supported by clients and MAY be used by servers.

## Relevant Specifications

[Relaynet Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoHTTP.

## Open Questions

- This is basically a webhook. Should it be renamed to _PoWebhook_?
