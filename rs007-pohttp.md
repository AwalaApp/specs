---
permalink: /RS-007
---
# PoHTTP: Parcel Delivery over HTTP, Version 1
{: .no_toc }

- Id: RS-007.
- Status: Working draft.
- Type: Implementation.
- Issue tracking label: [`spec-pohttp`](https://github.com/relaynet/specs/labels/spec-pohttp).

## Abstract
{: .no_toc }

This document describes PoHTTP, a [parcel delivery binding](rs000-core.md#parcel-delivery-binding) for Internet-based Parcel Delivery Connections (PDC) based on HTTP.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

PoHTTP was specifically designed to make it easier for maintainers of existing systems to support Relaynet. For example, an organization maintaining a REST API could collect parcels by implementing a new API endpoint (e.g., `/relaynet`), or they could integrate an [Enterprise Service Bus](https://en.wikipedia.org/wiki/Enterprise_service_bus) to convert Relaynet service messages to API calls.

In some cases, it may also be desirable for a public gateway to expose an HTTP interface so it can receive parcels from certain endpoints. For example, a Shell script in a legacy system could use `curl` to deliver parcels.

As an Internet-based PDC, the only operation supported by this binding is [parcel delivery](#parcel-delivery).

## Parcel Delivery

To deliver each parcel, the client MUST make a `POST` request to the HTTP URL corresponding to the node address, with the parcel as the body and the following headers:

- `Content-Type` MUST be set to `application/vnd.relaynet.parcel`.
- If the client is a public gateway, `X-Relaynet-Gateway` MUST provide the target endpoint with its address using the request header `X-Relaynet-Gateway`. For example, `X-Relaynet-Gateway: https://gateway.humanitarian.org`.

The server MUST respond with one of the following status codes:

- `202` (Accepted) response as soon as it successfully [stores or forwards](https://en.wikipedia.org/wiki/Store_and_forward) the parcel.
- `307` (Temporary Redirect) per [RFC-7231](https://tools.ietf.org/html/rfc7231#section-6.4.7) or `308` (Permanent Redirect) per [RFC-7238](https://tools.ietf.org/html/rfc7238), in which case the client MUST repeat the `POST` request against the URL specified in the `Location` response header. Clients MUST honor up to 3 consecutive redirects; additional consecutive redirects MAY be honored.
- `400` if the body of the request is not a valid RAMF-serialized parcel.
- A public gateway MAY return a `507` (Insufficient Storage) response when it no longer has the capacity to accept parcels for the target endpoint or its gateway. In this case, the client SHOULD retry to deliver the parcel at a later point but before it expires.
- Any other standard status code in the range 400-599 that the server regards applicable. For example, a `415 Unsupported Media Type` code could be returned if the `Content-Type` request header did not match `application/vnd.relaynet.parcel`.

The response MAY contain a body, but defining how to process it is outside this specification.

## HTTP Considerations

Clients and servers implementing this specification MUST support HTTP version 1.1, and they SHOULD also support HTTP version 2.

HTTP requests MUST set the `Host` header accordingly.

HTTP connections MUST be sessionless, so clients and servers MUST NOT use cookies.

[Rate limiting](https://tools.ietf.org/html/rfc6585#section-4) SHOULD be supported by clients as servers MAY use it.

Headers other than the ones described in this specification MAY be used, but defining how they should be handled at the other end is outside the scope of this specification.

## Relevant Specifications

[Relaynet Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoHTTP.
