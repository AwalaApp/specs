# Relaynet Core

- Id: RS-000.
- Status: Working draft.
- Type: Implementation.

## Abstract

This document describes the core elements of the Relaynet protocol suite, whose purpose is to make software tolerant to potentially large network latencies through the use of [asynchronous messaging](https://www.techopedia.com/definition/26454/asynchronous-messaging).

## Introduction

Software running on different computers is typically integrated using some form of [Remote Procedure Call (RPC)](https://en.wikipedia.org/wiki/Remote_procedure_call), a seemingly simple and familiar pattern that resembles local function calls in programming. Services running on HTTP, such as RESTful or gRPC APIs, employ this pattern.

RPCs work well in a reliable network -- One with a low [round-trip time (RTT)](https://en.wikipedia.org/wiki/Round-trip_delay_time) and an adequate [throughput](https://en.wikipedia.org/wiki/Throughput). The higher the RRT or the lower throughput can be, the more complicated an RPC implementation becomes. And with an extremely high RRT and/or an extremely low throughput, RPCs do not work at all.

In contrast to RPCs, asynchronous messaging does not depend on a reliable network. In fact, it does not require the ability for one node to reach another node directly. It does, however, require the introduction of a [broker](https://en.wikipedia.org/wiki/Message_broker) to queue and dispatch messages.

Given the ubiquity of the RPC integration style, societies with access to computers or smartphones but no (reliable) Internet access are kept from using the Internet. The best they can hope for is a [sneakernet](https://en.wikipedia.org/wiki/Sneakernet) that provides them with limited, curated content.

Relaynet is designed to change that through the use of asynchronous messaging, and by leveraging sneakernets to transport data to and from the Internet in a secure manner.

## Concepts

- A **service** is a collection of _applications_ that communicate amongst themselves. A service can be centralized (client-server) or decentralized (peer-to-peer).
- **Applications** exchange _messages_ amongst themselves, and because they can't communicate directly, they each use an _endpoint_ as a broker.
- A **message** is serialized in the format determined by the service and doesn't have to be encrypted or signed.
- An **endpoint** receives a message from its application and converts it into a _parcel_ for the target application's endpoint, and because they still can't communicate directly, they each use a _gateway_ as a broker. When an endpoint receives a parcel from the gateway, it has to decrypt the message and pass it to its application.
- A **parcel** encapsulates a message by encrypting it with the target endpoint's certificate and signing it with the origin endpoint's key.
- A **gateway** receives parcels from endpoints and puts them into cargo for another gateway, using a _relayer_ as a broker. When a gateway receives cargo from a relayer, it decrypts the parcels and delivers them to their corresponding target endpoints.
- A **cargo** encapsulates one or more parcels
- A **relayer** _relays_ cargo from one gateway to one or more gateways.

The following diagram shows how these entities interact with each other:

![](assets/rs000/protocol-layers.png)

In Relaynet, [same-layer and adjacent-layer interactions](https://upskilld.com/learn/same-layer-and-adjacent-layer-interactions/) are defined by _messaging protocols_ and _message transport bindings_, respectively.

For example, if Twitter supported Relaynet, Twitter would be the _service_, the Twitter mobile apps would be _applications_, the Twitter API would also be an _application_. The _endpoints_ in the mobile apps could simply be Java (Android) or Swift (iOS) libraries, whilst the _endpoint_ in the Twitter API could be a new API endpoint (e.g., `https://api.twitter.com/relaynet`).

## Addressing

This document only defines [point-to-point](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PointToPointChannel.html) message delivery. [Service Message Broadcast (RS-013)](rs013-pubsub.md) defines a [publish-subscribe](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) protocol.

Each node in Relaynet MUST have a unique address, and the type of address is determined by how it is accessed in its message transport binding:

- _Host nodes_ can be reached by host/port, which are included in the address in the form `scheme[+bindingHint]:domainNameOrIpAddress[:port][/extra]`, where:
  - `scheme` is determined by the messaging protocol as defined below.
  - `bindingHint` specifies the message transport binding. If absent, [Application-Level Protocol Negotiation](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation) MUST be done.
  - `domainNameOrIpAddress` is the host name as a DNS record or IPv4/IPv6 address. If using a DNS record, the binding SHOULD specify the type of allowed DNS records (e.g., A, CNAME, SRV).
  - `port` is the Layer 4 (e.g., TCP) port on which the host listens. This does not apply when using SRV records.
  - `extra` is any additional information necessary to reach the node. For example, it could be a URL path in bindings using HTTP.
- A _private node_ has the digest of its public key as its address (`scheme:publicKeyDigest`), and its peer in the message transport binding knows how to reach it. The digest uses SHA-256 and is hex-encoded. It MUST also be prefixed with a zero `0` to denote the version of the address, as a different algorithm to calculate the address can be defined in the future.

## Messaging Protocols

These protocols define the [same-layer interactions](https://upskilld.com/learn/same-layer-and-adjacent-layer-interactions/) between gateways, endpoints and applications.

The [Relaynet PKI](rs002-pki.md) defines the use of certificates in these protocols. The [Internet PKI](https://tools.ietf.org/html/rfc5280) does not apply here.

### Service Messaging Protocol

This protocol defines the interactions between two applications in a centralized service, or amongst two or more applications in a decentralized service.
 
The service has full control over this protocol, including the types of messages that its applications exchange (their contents, serialization format, etc).

### Endpoint Messaging Protocol

This protocol determines the interactions between two endpoints.

In terms of addressing, _host endpoints_ and _private endpoints_ MUST use the schemes `rneh` and `rnep`, respectively. For example, `rneh:example.com` or `rneh+grpc:example.com` (if using the [gRPC binding](rs009-pogrpc.md)).

#### Parcel

A parcel encapsulates a service message and encrypts it with the target endpoint's certificate. 

Parcels are serialized with the [Relaynet Abstract Message Format (RAMF)](rs001-ramf.md), using the ASCII character "P" (for "parcel") as its _concrete message format signature_. Gateways and the target endpoint MUST follow the post-deserialization validation listed in the RAMF specification.

The sender certificate is either self-signed or issued by the recipient (see _Parcel Delivery Authorization_).

Gateways MUST override any previously queued parcel with the same id. Endpoints can use this to replace stale messages in the same relay.

The payload [plaintext](https://en.wikipedia.org/wiki/Plaintext) contains the service message and its media type, and is serialized with the following binary sequence (with little-endian encoding):

1. An 8-bit unsigned integer (1 octet) representing the length of the service message type.
1. A UTF-8 encoded string representing the type of the service message. For example, `application/x-protobuf; messageType="twitter.Tweet"`.
1. A 32-bit unsigned integer (4 octets) representing the length of the service message.
1. The service message serialized in the format dictated by the service.

#### Parcel Delivery Authorization

A X.509 certificate chain whereby Endpoint A instructs its gateway and its relaying gateway to accept parcels from Endpoint B to Endpoint A.

Applications behind private endpoints are expected to include this certificate in their service messages where they request to subscribe to messages from the target app.

Service applications are responsible for the issuance, transmission, storage and revocation of Parcel Delivery Authorizations. Gateways are responsible for enforcing them.

### Gateway Messaging Protocol

This protocol defines the interactions between two gateways.

In terms of addressing, _host gateways_ and _private gateways_ MUST use the schemes `rngh` and `rngp`, respectively. For example, `rngh:example.com` or `rngh+grpc:example.com` (if using the [gRPC binding](rs008-cogrpc.md)).

#### Cargo

Its purpose is to encapsulate one or more parcels, encrypting them with the target gateway's certificate.

Cargoes are serialized with the [Relaynet Abstract Message Format (RAMF)](rs001-ramf.md), using the ASCII character "C" (for "cargo") as its _concrete message format signature_. Relayers and gateways MUST follow the post-deserialization validation listed in the RAMF specification.

The payload [plaintext](https://en.wikipedia.org/wiki/Plaintext) contains one or more parcels, and is serialized with the following binary sequence (with little-endian encoding), which is repeated for each parcel:

1. A 32-bit unsigned integer (4 octets) representing the length of the parcel.
1. The parcel serialized in the RAMF.

#### Cargo Collection Authorization

A RAMF message whereby Gateway A allows a relayer to collect cargo on its behalf from Gateway B. This is to be eventually used as described in the [cargo relay binding](#cargo-relay-binding).

Its payload MUST be encrypted with Gateway B's certificate and its plaintext MUST contains the following information:

- The (issuing endpoint address, certificate serial number) tuple for each _Parcel Delivery Deauthorization_ issued by Gateway A's endpoints.
- Binding-level constraints to authenticate the relayer, like expecting a specific _Distinguished Name_ in its client-side TLS certificate.

The payload plaintext MUST be serialized with [Protocol Buffers v3](https://developers.google.com/protocol-buffers/docs/proto3) using the `CargoCollectionAuthorization` message as defined below:

```proto
syntax = "proto3";

package relaynet.messaging.gateway;

import "google/protobuf/any.proto";

message CargoCollectionAuthorization {
    repeated ParcelDeliveryDeauthorization parcel_delivery_deauthorizations = 1;

    // The key MUST be the name of the binding (lower case) and the value is defined by the binding.
    map<string, google.protobuf.Any> relayer_constraints = 2;
}

message ParcelDeliveryDeauthorization {
    string endpoint_address = 1;
    string endpoint_certificate_serial_number = 2;
}
```

## Message Transport Bindings

Bindings define the [adjacent-layer interactions](https://upskilld.com/learn/same-layer-and-adjacent-layer-interactions/) between endpoints/gateways and gateways/relayers. Bindings are protocols that leverage pre-existing [Layer 7](https://en.wikipedia.org/wiki/Application_layer) protocols (e.g., HTTP) or they can be purpose-built.

This document describes the requirements applicable to all bindings, but does not define any concrete binding as they are defined in separate documents.

A binding MUST define its clients and servers, and how they implement this specification. Typically, the server listens on an address/port location, and the client initiates the communication. A peer-to-peer model could be represented with nodes that act as both clients and servers.

The client MUST authenticate the server with the following constraints:

- TLS MUST be used if the Layer 4 protocol is TCP, unless communication happens via the loopback network interface.
- DTLS MUST be used if the Layer 4 protocol is UDP, unless communication happens via the loopback network interface.
- When using the loopback network interface (i.e., localhost or `127.0.0.0/8`), server authentication MAY be skipped if the server listens on a system port (i.e., a port in the range 0-1023).
- When using Unix sockets, the client MUST check that the expected user owns the file.

Likewise, the server MUST authenticate the client and the binding MUST specify the mechanism(s) to do so, including whether or how to do client registration.

Clients and servers MUST comply with the [Internet PKI](https://tools.ietf.org/html/rfc5280) when using TLS. When not using TLS, they SHOULD NOT use [Relaynet PKI](rs002-pki.md) certificates for client/server authentication because they are only meant to be used in messaging protocols.

For performance reasons, the client and the server SHOULD not use the loopback network interface when they are on the same computer, and SHOULD instead use Unix sockets or any other Inter-Process Communication (IPC) mechanism supported by the host system.

Bindings MAY extend this specification, but they MUST NOT override it.

### Parcel Delivery Binding

This is a protocol that establishes a _Parcel Delivery Network_ (PDN) between an endpoint and a gateway, so that the two can exchange parcels.

The binding MUST support the following:

- The endpoint can send parcels to the gateway, and vice versa. The node delivering the parcel MUST NOT remove it until the target node has acknowledged it.
- A private endpoint can ask its gateway to issue a certificate for the endpoint, so that it can be subsequently used to issue a _Parcel Delivery Authorization_.
- A private endpoint can send a _Parcel Delivery Deauthorization_ to instruct its gateway not accept incoming parcels with that parcel delivery authorization.

### Cargo Relay Binding

This is a protocol that establishes a _Cargo Relay Network_ (CRN) between an a gateway and a relayer, so that the two can exchange cargo.

The binding MUST support the following:

- The relayer can send parcels to the gateway, and vice versa. The node delivering the parcel MUST NOT remove it until the target node has acknowledged it.
- A relayer can request a [cargo collection authorization](#cargo-collection-authorization) from the current gateway, so that the relayer can collect messages for the current gateway at the other end. The binding MAY allow the gateway to place restrictions on its use, using the appropriate field in the cargo collection authorization.

A user gateway MAY require the relayer to provide a Cargo Collection Authorization (CRA) from the relaying gateway. A relaying gateway MUST require at least one CRA because:

- It needs the user gateway's certificate to identify the parcels that should be delivered to that gateway. Such parcels use a [parcel delivery authorization](#parcel-delivery-authorization) as the sender certificate chain, which MUST contain the user gateway's certificate.
- The relaying gateway should have some degree of trust that the relayer will actually send the cargo to the target gateway.

The relayer SHOULD follow the following process when it interacts with a gateway:

1. Relayer delivers cargo(es) and if necessary waits for all ACKs to arrive.
1. Request cargo collection authorization for target gateway.
1. Wait a few seconds in case there are responses to the cargo(es) that were delivered earlier.
1. Collect cargoes.
