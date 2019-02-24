# Relaynet Public Key Infrastructure and Revocation Profile

- Id: RS-002.
- Status: Working draft.
- Type: Implementation.

## Abstract

This document describes how to issue, distribute, store, revoke and interpret X.509 v3 certificates in Relaynet [messaging protocols](rs000-core.md#messaging-protocols). Despite the use of X.509 certificates, this PKI profile is independent of and incompatible with the [Internet PKI profile](https://tools.ietf.org/html/rfc5280).

## Basic Attributes and Constraints

The _Distinguished Name_ MUST only contain the _Common Name_ (CN), which MUST be set to the node's address (including its schema; e.g., `CN=rne://example.com`, `CN=rng:0b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c`).

A certificate MUST NOT be valid before its issuer is valid or after its issuer expires.

Certificates in this PKI profile MUST always be encoded with the [Distinguished Encoding Rules (DER)](https://en.wikipedia.org/wiki/X.690#DER_encoding).

## Certificate Types

### Endpoint Certificate

An endpoint certificate MUST be issued by one of the following:

- Itself (a self-signed certificate).
- A peer endpoint when the certificate is a [_parcel delivery authorization_](#parcel-delivery-authorization-pda).

### Parcel Delivery Authorization (PDA)

A specific type of endpoint certificate whereby Endpoint A instructs its gateway and its relaying gateway to accept parcels from Endpoint B to Endpoint A. If Endpoint A is private, the endpoint and its gateways MUST refuse parcels where the sender certificate is not a valid PDA.

Each PDA is a certificate chain formed of the following sequence (from leaf to root):

1. (Optional) Endpoint B's _signature-only certificate_. A certificate that MUST only be used to sign messages in the [endpoint channel](rs000-core.md#endpoint-messaging-protocol) (e.g., parcels) on behalf of Endpoint B.
1. Endpoint B's certificate.
1. (Optional) Endpoint A's _signature-only certificate_. A certificate that MUST only be used to issue PDAs.
1. Endpoint A's certificate, issued by its [gateway](#gateway-certificate).

Gateways MUST refuse PDAs whose issuing endpoint's _Common Name_ does not match the recipient of the parcel.

#### Rate Limiting Extension

Endpoint B's certificate MAY contain the non-critical extension _PDA Rate Limiting_, so that Endpoint A can instruct its gateways to limit the volume of parcels that Endpoint B can send. The extension has the following attributes:

- _Limit_: How many parcels can be sent.
- _Period_: Number of seconds during which the limit applies.

For example, Endpoint A could specify a limit of one parcel every 86400 seconds (one day).

The eligibility of a message is determined by its own date, not the date when the message was received.

Gateways SHOULD support this extension, whilst endpoints MAY support it.

### Gateway Certificate

A gateway's certificate MUST be either self-signed or issued by its peer gateway, forming the chain below (from leaf to root):

1. (Optional) The gateway's _signature-only certificate_. A certificate that MUST only be used to issue [endpoint certificates](#endpoint-certificate).
1. The gateway's certificate.
1. (Optional) The peer gateway's _signature-only certificate_. A certificate that MUST only be used to issue gateway certificates.
1. (Optional) The peer gateways's certificate.

A certificate issued by another gateway MUST NOT be used to issue additional gateway certificates.

## Certificate Extensions

### Certificate Type Extension

Every certificate in this PKI MUST use the critical extension _PDA Certificate Type_ to specify its type. The value MUST correspond to the [ASN.1](https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One) enumeration below:

```asn1
PDACertType ::= ENUMERATED {
    senderSignOnly        (0),   -- A sender's signature-only certificate
    sender                (1),   -- A sender's certificate
    recipientSignOnly     (2),   -- A recipient's signature-only certificate
    recipient             (3),   -- A recipient's certificate
    gatewaySignOnly       (4),   -- A gateway's signature-only certificate
    gateway               (5),   -- A gateway's certificate
    peerGatewaySignOnly   (6),   -- A peer gateway's signature-only certificate
    peerGateway           (7) }  -- A peer gateway's certificate
```

## Certificate and Key Rotation

Endpoints and gateways MAY support multiple certificates, with the same or different asymmetric keys (and potentially different addresses), at any point in time, in order to facilitate certificate or key rotation. 

An endpoint or gateway initiating a certificate rotation MUST share the new certificate using a _Certificate Rotation Message_ (CRM) through the appropriate [messaging channels](rs000-core.md#messaging-protocols). Such a message MUST:

- Be serialized with the [Relaynet Abstract Message Format (RAMF)](rs001-ramf.md), using the octet `0x10` as its _concrete message format signature_.
- Be signed with a certificate that the target endpoint/gateway already trusts.
- Have their payload encrypted as specified in the [Core](rs000-core.md) and [RAMF](rs001-ramf.md) specifications.
- Have their payload plaintext contain only the new certificate.

CRMs MUST be top-level messages in the [endpoint channel](rs000-core.md#endpoint-messaging-protocol), but they MUST be encapsulated in cargo in the [gateway channel](rs000-core.md#gateway-messaging-protocol) (along with parcels) to prevent malicious relayers from identifying and dropping such messages.

All [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652) values in Relaynet MUST include the appropriate metadata to identify the correct certificate.

## Certificate Revocation

Certificates MUST be revoked when their private keys are compromised, when a [rotation](#certificate-and-key-rotation) is complete, or when deemed appropriate by their subject or issuer.

### Endpoint Self-Signed Certificate Revocation

The endpoint MUST:
 
- Issue [_parcel delivery deauthorizations_ (PDDs)](#parcel-delivery-deauthorization-pdd) for all its active PDAs.
- Securely destroy its private key.

### Parcel Delivery Deauthorization (PDD)

A Parcel Delivery Deauthorization (PDD) revokes one or more [PDAs](#parcel-delivery-authorization-pda). It may be requested by the endpoint or its gateway.

An endpoint MUST use the [message transport binding](rs000-core.md#message-transport-bindings) to instruct its gateway to revoke its self-signed certificate or a specific PDA. Such a request MUST include the following data, serialized in the format determined by the binding:

- (Optional) Serial numbers of the PDAs to revoke. It may be absent to revoke all the PDAs issued by the endpoint.
- (Required) Expiry date of the deauthorization. If revoking all PDAs from the endpoint, this MUST be the expiry date of the endpoint certificate. If revoking a specific PDA, this MUST be the expiry date of the PDA.

Gateway MUST include all their active PDDs in their [_Cargo Collection Authorizations_](rs000-core.md#cargo-collection-authorization-cca), and they MUST enforce PDDs for as long as they are active.
 
Relaying gateways MAY cache PDDs until they expire in order to refuse future parcels whose PDA has been revoked.

### Gateway Certificate Revocation (GCR)

A gateway can revoke its own certificate by issuing a GCR message to its peer gateway(s). These messages MUST be serialized with RAMF, using the octet `0x11` as its _concrete message format signature_, and have an empty payload (i.e., one of length zero).

GCRs MUST be sent in the payload plaintext of a cargo, along with parcels, in order to prevent a malicious relayer from identifying and dropping such messages.

## Security and Scalability Considerations

An endpoint or gateway implemented as a distributed system SHOULD use different keys (and therefore different certificates) for signing and decrypting, as supported by this specification. This might not be necessary when the endpoint/gateway is a monolith.
