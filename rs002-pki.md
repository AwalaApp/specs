---
permalink: /RS-002
---
# Relaynet Public Key Infrastructure and Revocation Profile
{: .no_toc }

- Id: RS-002.
- Status: Working draft.
- Type: Implementation.
- Issue tracking label: [`spec-pki`](https://github.com/relaynet/specs/labels/spec-pki).

## Abstract
{: .no_toc }

This document describes how to issue, distribute, store, revoke and interpret X.509 certificates in Relaynet [messaging protocols](rs000-core.md#messaging-protocols). Despite the use of X.509 certificates, this PKI profile is independent of and incompatible with the [Internet PKI profile](https://tools.ietf.org/html/rfc5280) as used in the TLS protocol.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## General Constraints and Attributes

Certificates in this PKI profile MUST be represented as [X.509 v3 certificates](https://www.itu.int/rec/T-REC-X.509/en).

The _Distinguished Name_ MUST only contain the _Common Name_ (CN) attribute, and it MUST be set to the node's private address (e.g., `CN=0b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c`).

A certificate MUST NOT be valid before its issuer is valid or after its issuer expires.

The serial number of the certificate MUST NOT be negative (in order to make it easier to convert between little endian and big endian representations).

## Certificate Types

Endpoints and gateways can use the following types of certificates.

### Endpoint Certificate

An endpoint certificate MUST be issued by one of the following Certificate Authorities (CAs):

- Itself, if it is a public endpoint.
- Its local gateway, if it is a private endpoint.
- Another endpoint, resulting in a [_parcel delivery authorization_](#parcel-delivery-authorization-pda).

### Parcel Delivery Authorization (PDA)

Given private Endpoint A and Endpoint B, Endpoint A MAY instruct its gateway and its relaying gateway to accept parcels from Endpoint B by signing Endpoint B's certificate, which will result in a special endpoint certificate called _Parcel Delivery Authorization_ (PDA).

The certification path of a PDA is formed of the following sequence (from end entity to root):

1. Endpoint B's certificate.
1. Endpoint A's certificate.
1. Endpoint A's local gateway.
1. Endpoint A's relaying gateway.

When relaying parcels where the recipient is a private endpoint, gateways MUST refuse those where the certificate for the sender of the parcel was not issued by the target endpoint. In other words, the Common Name of the second certificate MUST match the recipient of the RAMF-serialized parcel.

#### Rate Limiting Extension

Endpoint A MAY rate limit the volume of parcels that Endpoint B may send with the PDA by including the non-critical extension _PDA Rate Limiting_.

Gateways SHOULD enforce the rate limiting specified by the extension, if present. When evaluating the eligibility of a message for rate limiting purposes, relaying gateways MUST use the time when the message was received, whilst user gateways MUST use the date specified in the RAMF message.

The target endpoint (Endpoint A) MAY enforce the rate limiting.

The [ASN.1](https://www.itu.int/ITU-T/studygroups/com17/languages/X.680-0207.pdf) Object Identifier of this extension is defined as follows:

```
PKIPDARateLimitId OBJECT IDENTIFIER ::= {
    itu-t(0) identified-organization(4) etsi(0) reserved(127) etsi-identified-organization(0)
        relaycorp(17) relaynet(0) pki(0) 0
    }
```

The ASN.1 value of the extension is defined as follows:

```
PKIPDARateLimit ::= SEQUENCE {
    limit  INTEGER,
    period INTEGER
}
```

Where, `limit` specifies how many parcels can be sent within a given number of seconds (`period`). For example, a `limit` of `1` and a `period` of `86400` allow a maximum of one parcel a day.

### Gateway Certificate

Given Gateway A and Gateway B, Gateway A's certificate MUST be either self-issued or issued by Gateway B, forming the certification path below (from end entity to root):

1. Gateway A's certificate, issued by Gateway B.
1. One or more certificates representing the certificate chain for Gateway B.

A certificate issued by another gateway MUST NOT be used to issue additional gateway certificates.

## Certificate and Key Rotation

Endpoints and gateways MAY use multiple certificates, with the same or different asymmetric keys (and therefore different addresses), at any point in time, in order to facilitate certificate or key rotation. 

An endpoint or gateway initiating a certificate rotation MUST share the new certificate using a _Certificate Rotation Message_ (CRM) through the appropriate [messaging channels](rs000-core.md#messaging-protocols). Such a message MUST:

- Be serialized with the [Relaynet Abstract Message Format (RAMF)](rs001-ramf.md), using the octet `0x10` as its _concrete message type_.
- Be signed with a certificate that the target endpoint/gateway already trusts.
- Have their payload encrypted as specified in the [Core](rs000-core.md) and [RAMF](rs001-ramf.md) specifications.
- Have its payload plaintext contain only the new certificate.

CRMs MUST be top-level messages in the [endpoint channel](rs000-core.md#endpoint-messaging-protocol), but they MUST be encapsulated in cargo in the [gateway channel](rs000-core.md#gateway-messaging-protocol) (along with parcels) to prevent malicious couriers from identifying and dropping such messages.

Since a recipient could have multiple keys at any point in time, endpoints and gateways MUST include the appropriate metadata to identify the correct certificate in any [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652) enveloped-data values that they generate.

## Certificate Revocation

Certificates MUST be revoked when their private keys are compromised, when a [rotation](#certificate-and-key-rotation) is complete, or when deemed appropriate by their subject or issuer.

### Endpoint Self-Signed Certificate Revocation

The endpoint MUST:
 
- Issue [_parcel delivery deauthorizations_ (PDDs)](#parcel-delivery-deauthorization-pdd) for all its active PDAs.
- Securely destroy its private key.

### Parcel Delivery Deauthorization (PDD)

A Parcel Delivery Deauthorization (PDD) revokes one or more [PDAs](#parcel-delivery-authorization-pda). It may be requested by the endpoint or its gateway.

An endpoint MUST use the [message transport binding](rs000-core.md#message-transport-bindings) to instruct its gateway to revoke its self-issued certificate or a specific PDA. Such a request MUST include the following data, serialized in the format determined by the specific binding:

- (Required) The endpoint address affected by the deauthorization. This is needed in case the endpoint has multiple active certificates as a result of a key rotation.
- (Optional) Serial numbers of the PDAs to revoke. It may be absent to revoke all the PDAs issued by the endpoint.
- (Required) Expiry date of the deauthorization. If revoking all PDAs from the endpoint, this MUST be the expiry date of the endpoint certificate. If revoking a specific PDA, this MUST be the expiry date of the PDA.

Gateways MUST include all their active PDDs in their [_Cargo Collection Authorizations_](rs000-core.md#cca), and they MUST enforce PDDs for as long as they are active.
 
Relaying gateways MAY cache PDDs until they expire in order to refuse future parcels whose PDA has been revoked.

### Gateway Certificate Revocation (GCR)

A gateway MAY revoke its own certificate by issuing a GCR message to its peer gateway(s). These messages MUST be serialized with RAMF, using the octet `0x11` as its _concrete message type_, and have an empty payload (i.e., one of length zero).

GCRs MUST be sent in the payload plaintext of a cargo, along with parcels, in order to prevent a malicious courier from identifying and dropping such messages.

## X.509 Extensions

### Basic Constraints

Each certificate MUST have its Basic Constraints extension as defined in the X.509 v3 specification. The extension MUST be marked as critical and its attributes MUST be set as follows depending on the type of certificate:

| Certificate type | `cA` | `pathLenConstraint` |
| --- | --- | --- |
| Self-issued gateway certificate | `true` | `2` |
| Non-self-issued gateway certificate | `true` | `1` |
| Endpoint certificate (excluding PDAs) | `true` | `0` |
| PDA | `false` | `0` |

### Authority Key Identifier

Except for self-issued certificates, all certificates MUST include the Authority Key Identifier extension as defined in the X.509 v3 specification.

### Subject Key Identifier

All certificates MUST include the Subject Key Identifier extension as defined in the X.509 v3 specification.
