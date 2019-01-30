# Relaynet Core

- Id: RS-000.
- Status: Working draft.
- Type: Implementation.

## Abstract

## Introduction

Concepts / nodes, etc.

## Addressing

Supports the [point-to-point channel pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PointToPointChannel.html).

- Host nodes have addresses in the form `scheme[+hint]:domainNameOrIpAddress[:port][/extra_params]`.
  - If there's no binding hint, Application-Level Protocol Negotiation MUST be done.
  - If using a domain name, the binding will determine the type of allowed DNS records (e.g., A, CNAME, SRV). (Purpose-built protocols may prefer SRVs)
- Private nodes have opaque address (`scheme:certificateDigest`). Necessary for nodes unreachable from the current network — that is, endpoints only accessible via gateways and gateways only accessible via relayers.
  - MUST be the SHA-256, hex-encoded fingerprint of the node's certificate.
  - The routing node knows how to deliver the message (but the delivery configuration is managed out of band).
  - The first character is alphadecimal (digits and lower case letters) and denotes the version of the scheme. Starts at 0.

## Messaging Protocols

### Service Messaging Protocol

The service has full control over the types of messages that its applications exchange (their contents, serialization format, etc).

Service applications are responsible for the issuance, transmission, storage and revocation of Parcel Delivery Authorizations. Gateways are responsible for enforcing them.

### Endpoint Messaging Protocol

#### Parcel

A parcel encapsulates a service message and encrypts it with the target endpoint's certificate. Parcels are serialized with the [Relaynet Abstract Message Format](rs001-ramf.md), and as such it has the following fields:

- File format signature.
  - Concrete message format signature (1 octet): ASCII "P" for parcel.
  - Full signature for v1: "52 65 6c 61 79 6e 65 74 50 01" (hex).
- Recipient address. The target endpoint's address.
- Sender certificate. The certificate (chain) of the endpoint producing the parcel. It is either self-signed or issued by the recipient (see _Parcel Delivery Authorization_).
- Id. The unique id for the parcel. Gateways MUST override any previously queued parcel with the same id. Endpoints can use this to replace stale messages in the same relay.
- Date. When the parcel was created.
  - The origin gateway MUST reject back-dated parcels (with some wiggle room, ~5 mins).
  - The target gateway/endpoint MAY reject post-dated parcels.
  - MUST be within the validity of the response authorization (if present).
- TTL. The gateway should discard the parcel when it's expired. Maybe generate an undelivered parcel (see below).
  - Used in conjunction with Date to mitigate replay attacks before a session has been established. 
  - MUST be less than or equal to the expiry date of the response authorization, if present.
- Payload. Per RMAF, this is serialized with CMS. Its plaintext contains the service message and its media type, and is serialized with the following binary sequence (with little-endian encoding):
  1. An 8-bit unsigned integer (1 octet) representing the length of the service message type.
  1. A UTF-8 encoded string representing the type of the service message. For example, `application/x-protobuf; messageType="twitter.Tweet"`.
  1. A 32-bit unsigned integer (4 octets) representing the length of the service message.
  1. The service message serialized in the format dictated by the service.

Relaying gateways and target endpoints MUST validate incoming parcels:

- The recipient's address MUST match the target endpoint's.
- The sender's certificate MUST be valid at the time it is received.
- To mitigate replay attacks, the parcel MUST be valid per its date and TTL fields.
- To avoid replay attacks, the parcel id SHOULD be persistent until its TTL expires, and until then, any incoming parcel from the same endpoint and with the same id MUST be rejected.
- The signature MUST be valid.

Endpoints can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md). 

#### Parcel Delivery Authorization
A X.509 certificate chain whereby Endpoint A instructs its gateway and its relaying gateway to accept parcels from Endpoint B to Endpoint A.

Applications behind private endpoints are expected to include this certificate in their service messages where they request to subscribe to messages from the target app.

### Gateway Messaging Protocol

#### Cargo

Its purpose is to encapsulate one or more parcels, encrypting them with the target gateway's certificate. Cargoes are serialized with the [Relaynet Abstract Message Format](rs001-ramf.md), and as such it has the following fields:

- File format signature.
  - Concrete message format signature (1 octet): ASCII "C" for cargo.
  - Full signature for v1: "52 65 6c 61 79 6e 65 74 43 01" (hex).
- Recipient. The target gateway's address.
- Payload. Per RMAF, this is serialized with CMS. Its plaintext contains one or more parcels, and is serialized with the following binary sequence (with little-endian encoding), which is repeated for each parcel:
  1. An 32-bit unsigned integer (4 octets) representing the length of the parcel.
  1. The parcel serialized in RAMF.

Gateways MUST validate incoming cargo:

- The recipient's address MUST match the current gateway's.
- The sender's certificate MUST be valid at the time it is received.
- To mitigate replay attacks, the cargo MUST be valid per its date and TTL fields.
- To avoid replay attacks, the cargo id SHOULD be persistent until its TTL expires, and until then, any incoming cargo from the same gateway and with the same id MUST be rejected.
- The signature MUST be valid.

gateways can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md). 

#### Cargo Collection Authorization

A RAMF message whereby Gateway A allows Relayer R to collect cargo on its behalf from Gateway B. Its payload contains the following information:

- Zero or more topic subscriptions.
- The (issuing endpoint address, certificate serial number) tuple for each Parcel Delivery Deauthorization issued by Gateway A's endpoints.
- Binding-level constraints to authenticate the relayer, like expecting a specific Distinguished Name in its client-side TLS certificate.

## Message Transport Bindings

A binding is a protocol that establishes a transport medium to exchange parcels (in a PDN) or cargoes (in a CRN). It MAY operate on top of a pre-existing, general purpose protocol (e.g., HTTP), or be purpose-built.

A binding MUST define its clients and servers, and how they implement this specification. Typically, the server listens on an address/port location, and the client initiates the communication. A peer-to-peer model could be represented with nodes that act as both clients and servers.

The client MUST authenticate the server with the following constraints:

- TLS MUST be used if the Layer 4 protocol is TCP, unless communication happens via the loopback network interface.
- DTLS MUST be used if the Layer 4 protocol is UDP, unless communication happens via the loopback network interface.
- When using the loopback network interface (i.e., localhost or 127.0.0.0/8), server authentication MAY be skipped if the server listens on a system port (i.e., a port in the range 0-1023).
- When using Unix sockets, the client MUST check that the expected user owns the file.

Likewise, the server MUST authenticate the client and the binding MUST specify the mechanism(s) to do so, including whether or how to do client registration.

When using X.509 certificates for client/server authentication, they SHOULD not be Relaynet PKI certificates or their corresponding keys because they are meant to be used for a different purpose.

For performance reasons, the client and the server SHOULD not use the loopback network interface when they are on the same computer, and SHOULD instead use Unix sockets or any other Inter-Process Communication (IPC) mechanism supported by the host system.

Bindings MAY extend this specification, but they MUST not override it.

### Parcel Delivery Binding

Addressing

- Private endpoints have opaque address. Scheme: "rneo:". Syntax: `rneo:certificateDigest`).
- Host endpoint address (syntax: "rneh:domainNameOrIpAddress"). For example, "rneh:twitter.com".

Abstract packets:

- Parcel delivery. The node delivering the parcel MUST NOT remove it until the target node has acknowledged it.
  - Delivery id.
  - Parcel.
  - Parcel size in octets.
- Parcel Delivery Acknowledgement.
  - Delivery id.
- Gateway Certificate. Requested by private endpoints to issue Parcel Delivery Authorization.
- Parcel Delivery Deauthorization. Revokes a Parcel Delivery Authorization by serial number.
- Topic subscription.

### Cargo Relay Binding

Addressing

- Host gateway address. Syntax "rngh:domainNameOrIPAddress". Can include binding hint; e.g., "rngh+grpc:domainNameOrIPAddress".
- Opaque address. Syntax: "rngo:gatewayAddress".

Abstract packets:

- Cargo delivery. The node delivering the cargo MUST NOT remove it until the target node has acknowledged it.
  - Delivery id.
  - Cargo.
  - Cargo size in octets.
- Cargo delivery acknowledgement.
- Cargo collection authorization delivery. Encapsulates a Cargo Collection Authorization.
- Cargo Collection. MAY include a Cargo Relay Authorization (CRA) for each supported gateway in the gRPC call metadata, but a relayer's gateway MUST require at least one CRA because:
  - it could have a potentially large number of queued cargoes for multiple relayers serving different user gateways.
  - The relayer gateway's should have some degree of trust that the relayer will actually send the cargo to the target gateway.

Suggested sequence:

1. Relayer delivers cargo(es) and if necessary waits for all ACKs to arrive.
1. Request cargo collection authorization for target gateway.
1. Wait a few seconds in case there are responses to the cargo(es) that were delivered earlier.
1. Collect cargoes.

#### Addressing
