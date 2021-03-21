---
permalink: /RS-007
---
# PoHTTP: Parcel Delivery over HTTP, Version 1
{: .no_toc }

- Id: RS-007.
- Status: Working draft.
- Type: Implementation.
- Issue tracking label: [`spec-pohttp`](https://github.com/AwalaNetwork/specs/labels/spec-pohttp).

## Abstract
{: .no_toc }

This document describes PoHTTP, a [parcel delivery binding](rs000-core.md#parcel-delivery-binding) based on HTTP.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

PoHTTP was specifically designed to make it easier for maintainers of existing systems to support Awala. For example, an organization maintaining a REST API could collect parcels by implementing a webhook or they could integrate an [Enterprise Service Bus](https://en.wikipedia.org/wiki/Enterprise_service_bus) to convert Awala service messages to API calls.

In some cases, it may also be desirable for a public gateway to expose an HTTP interface so it can receive parcels from certain endpoints. For example, a Shell script in a legacy system could use `curl` to deliver parcels.

As a PDC, the only operation supported by this binding is [parcel delivery](#parcel-delivery).

## Parcel Delivery

To deliver each parcel, the client MUST make a `POST` request to the HTTP URL corresponding to the node address, with the parcel as the body and the following headers:

- `Content-Type` MUST be set to `application/vnd.awala.parcel`.
- If the client is a public gateway, `X-Awala-Gateway` MUST provide the target endpoint with its address using the request header `X-Awala-Gateway`. For example, `X-Awala-Gateway: https://gateway.humanitarian.org`.

The server MUST respond with one of the following status codes:

- `202` (Accepted) response as soon as it successfully [stores or forwards](https://en.wikipedia.org/wiki/Store_and_forward) the parcel.
- `307` (Temporary Redirect) per [RFC-7231](https://tools.ietf.org/html/rfc7231#section-6.4.7) or `308` (Permanent Redirect) per [RFC-7238](https://tools.ietf.org/html/rfc7238), in which case the client MUST repeat the `POST` request against the URL specified in the `Location` response header. Clients MUST honor up to 3 consecutive redirects; additional consecutive redirects MAY be honored.
- `400` if the request does not meet the requirements specified in this document.
- `403` if the overall request meets the requirements in this document but the parcel specified in the request body is invalid (e.g., it expired, it is bound for another address). The client MUST NOT try to send the parcel again.
- A public gateway MAY return a `507` (Insufficient Storage) response when it no longer has the capacity to accept parcels for the target endpoint or its gateway. In this case, the client SHOULD retry to deliver the parcel at a later point but before it expires.
- Any other standard status code in the range 400-599 that the server regards applicable. For example, a `415 Unsupported Media Type` code could be returned if the `Content-Type` request header did not match `application/vnd.awala.parcel`.

An error response (status code 40X or 50X) MAY have a body, in which case it will be subject to the value of the request header `Accept`:

- If `application/json` is among the values in `Accept`, the body SHOULD be serialized as a JSON document with a property `message` that explains what went wrong. Additionally, the response `Content-Type` MUST be `application/json`. The document MAY contain additional properties, but their semantics would be outside the scope of this specification.
- If the `Accept` header is missing or does not contain `application/json`, the response body SHOULD be plain text and use the `text/plain` Content Type.

## Public Node SRV records

Because this binding uses the Transmission Control Protocol (TCP; [RFC 793](https://tools.ietf.org/html/rfc793)), SRV records for PDC servers implementing this binding MUST use the `tcp` protocol.

For example, a public endpoint like `example.social` could specify an SRV record as follows:

```
_awala-pdc._tcp.example.social. 300 IN SRV 0 1 443 pohttp.example.social.
```

## HTTP Considerations

Clients and servers implementing this specification MUST support HTTP version 1.1, and they SHOULD also support HTTP version 2.

HTTP requests MUST set the `Host` header accordingly.

HTTP connections MUST be sessionless, so clients and servers MUST NOT use cookies.

[Rate limiting](https://tools.ietf.org/html/rfc6585#section-4) SHOULD be supported by clients as servers MAY use it.

Headers other than the ones described in this specification MAY be used, but defining how they should be handled at the other end is outside the scope of this specification.

## Relevant Specifications

[Awala Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoHTTP.
