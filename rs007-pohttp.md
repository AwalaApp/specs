# PoHTTP: Parcel Delivery over HTTP

- Id: RS-007.
- Status: Working draft.
- Type: Implementation.

## Abstract

This document describes PoHTTP, a [parcel delivery binding](rs000-core.md#parcel-delivery-binding) based on HTTP. This is an alternative to [PoSocket](rs005-posocket.md), meant to help adopt Relaynet in existing systems.

## Introduction

PoHTTP was specifically designed to make it easier for maintainers of existing systems to support Relaynet. For example, an organization maintaining a REST API could collect parcels by implementing a new API endpoint (e.g., `/relaynet`), or they could integrate an [Enterprise Service Bus](https://en.wikipedia.org/wiki/Enterprise_service_bus) to convert Relaynet service messages to API calls.

In some cases, it may also be desirable for a relaying gateway to expose an HTTP interface so it can collect parcels from certain endpoints. For example, a Shell script in a legacy system could use `curl` to deliver parcels.

Given the nature of the underlying protocol, this binding can only be used when the target is a [public node](rs000-core.md#addressing) -- That is, the HTTP server can only be a public endpoint or gateway.

## Binding hint

The [hint](rs000-core.md#addressing) for this binding MUST be `http`. For example, `rne+http://api.example.com/relaynet` would be a valid [public endpoint](rs000-core.md#endpoint-messaging-protocol) address and it would correspond to the HTTP URL `https://api.example.com/relaynet` (note the `https` scheme).

## Operations

### Parcel Collection

A user gateway MAY allow its endpoints to collect parcels using this binding. Public endpoints and relaying gateways MUST NOT support this functionality.

Collecting parcels involves two steps:

1. Retrieving the set of URLs where parcels can be found. This is done by making a `GET` request to the HTTP URL corresponding to the node address.
   - If the request can be fulfilled and there are parcels available, the server MUST return a `200` (OK) response with the Content Type `text/plain` and a plain text body where each line is the HTTP(S) URL to each parcel.
   - If the request can be fulfilled but there are no parcels available, the server MAY return a `204` (No Content) response instead of a `200` (OK) with an empty body.
   - The results MAY be paginated using the [`Link` header (RFC-5988)](https://tools.ietf.org/html/rfc5988) with the `rel` parameter set to `"next"`. E.g., `Link: <https://api.example.com/?page=2>; rel="next"`.
1. Collecting the parcels by making a `GET` request to download each parcel. When the request can be fulfilled, the server MUST return a `200` (OK) response with the content type `application/vnd.relaynet.parcel` and the parcel as the body.

When using HTTP v2, [Server Push](https://en.wikipedia.org/wiki/HTTP/2_Server_Push) SHOULD be used instead: The server MUST return a `204` (No Content) response, followed by a `200` (OK) response for each parcel, with the content type `application/vnd.relaynet.parcel` and the parcel as the body.

Servers SHOULD add the header `Cache-Control: private` to the responses described above.

### Parcel Collection Acknowledgement

The client MUST make a `DELETE` request to the URL of each parcel that is successfully downloaded and [stored/forwarded](https://en.wikipedia.org/wiki/Store_and_forward).

### Parcel Delivery

To deliver each parcel, the client MUST make a `POST` request to the HTTP URL corresponding to the node address, with `application/vnd.relaynet.parcel` as the content type and the parcel as the body.

The server MUST return a `202` (Accepted) response as soon as it successfully [stores or forwards](https://en.wikipedia.org/wiki/Store_and_forward) the parcel. A relaying gateway MUST return a `507` (Insufficient Storage), it MUST respond with an appropriate status code in the range 400-599, including `507` (Insufficient Storage) .

If per [Parcel Delivery Binding](rs000-core.md#parcel-delivery-binding) in Relaynet Core, the client is a relaying gateway and it is required to provide the target endpoint with its address. PoHTTP gateways MUST do so with the request header `X-Relaynet-Gateway`. For example, `X-Relaynet-Gateway: rng+http://gateway.humanitarian.org`.

## Client Authentication

A user gateway MAY require its endpoints to provide authentication information in the `Authorization` request header and/or require the use of a client-side TLS certificate issued by a trusted Certificate Authority. Defining how that would work would be the responsibility of another Relaynet Specification.

Public endpoints and relaying gateways MUST NOT require their clients to provide any authentication- or authorization-related information in the request headers, nor require the use of TLS-certificates. 

Regardless of the type of Relaynet node they represent, servers MAY still refuse requests from suspicious and/or ill-behaved clients.

## HTTP Considerations

Clients and servers implementing this specification MUST support HTTP version 1.1, and they SHOULD also support HTTP version 2.

Clients and servers MUST NOT use cookies.

Servers MAY support [rate limiting](https://tools.ietf.org/html/rfc6585#section-4).

Clients MUST follow redirects and they SHOULD support rate limiting.

Status codes or headers other than the ones described in this specification MAY be used, but defining how they should be handled at the other end is outside the scope of this specification.

## TLS Considerations

Since [message transport bindings](rs000-core.md#message-transport-bindings) MAY NOT use TLS when communication happens via the loopback network interface per [Relaynet Core](rs000-core.md), clients MUST fall back to plain HTTP without TLS in that case if the server does not support TLS. This constraint also applies to any request that the client makes following instructions from the server, such as redirects and pagination.

Clients MUST support [Server Name Identification](https://en.wikipedia.org/wiki/Server_Name_Indication).

## Relevant Specifications

[Relaynet Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoHTTP.

## Open Questions

- This binding is meant for public endpoints and relaying gateways. User gateways should be using PoSocket or some other IPC-based binding, so should they be forbidden from supporting this binding? If so, this binding would not support parcel collection.
