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

This document describes how to issue, distribute, store, revoke and interpret X.509 certificates in Relaynet [messaging protocols](rs000-core.md#messaging-protocols). Despite the use of X.509 certificates, this Public Key Infrastructure (PKI) profile is independent of and incompatible with the [Internet PKI profile](https://tools.ietf.org/html/rfc5280) as used in the TLS protocol.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

Relaynet relies extensively on its PKI in order to authenticate and authorize nodes without a real-time connection to an external authentication/authorization server, as well as to encrypt payloads when the [Channel Session Protocol](./rs003-key-agreement.md) is not employed.

The [Relaynet Message Abstract Format (RAMF)](./rs001-ramf.md) depends on Relaynet PKI certificates to authenticate the source of the message and ensure its integrity. Such certificates can be self-issued when the message recipient is a public node, but they have to be pre-authorized by the recipient if said recipient is a private node.

Private nodes express their willingness to accept messages from another node in the same channel by issuing a _delivery authorization_, which is simply a certificate issued for the sender. When relaying messages bound for private nodes, gateways and couriers have to make sure that the sender signed the message with an appropriate delivery authorization.

## General Constraints and Attributes

Certificates in this PKI profile MUST be represented as [X.509 v3 certificates](https://www.itu.int/rec/T-REC-X.509/en).

The _Distinguished Name_ MUST only contain the _Common Name_ (CN) attribute, and it MUST be set to the node's private address (e.g., `CN=0b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c`).

A certificate MUST NOT be valid before its issuer is valid or after its issuer expires.

## Certificate Types

Endpoints and gateways can use the following types of certificates.

### Endpoint Certificate

An endpoint certificate MUST be issued by one of the following Certificate Authorities (CAs):

- Itself, if it is a public endpoint.
- Its private gateway, if it is a private endpoint.
- Another endpoint, resulting in a [_parcel delivery authorization_](#parcel-delivery-authorization-pda).

### Parcel Delivery Authorization (PDA)

Given private Endpoint A and Endpoint B, Endpoint A MAY instruct its gateway and its public gateway to accept parcels from Endpoint B by signing Endpoint B's certificate, which will result in a special endpoint certificate called _Parcel Delivery Authorization_ (PDA).

The certification path of a PDA is formed of the following sequence (from end entity to root):

1. Endpoint B's certificate.
1. Endpoint A's certificate.
1. Endpoint A's private gateway.
1. Endpoint A's public gateway.

When relaying parcels where the recipient is a private endpoint, gateways MUST refuse those where the certificate for the sender of the parcel was not issued by the target endpoint. In other words, the Common Name of the second certificate MUST match the recipient of the RAMF-serialized parcel.

#### Rate Limiting Extension

Endpoint A MAY rate limit the volume of parcels that Endpoint B may send with the PDA by including the non-critical extension _PDA Rate Limiting_.

Gateways SHOULD enforce the rate limiting specified by the extension, if present. When evaluating the eligibility of a message for rate limiting purposes, public gateways MUST use the time when the message was received, whilst private gateways MUST use the date specified in the RAMF message.

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

### Cargo Delivery Authorization (CDA)

Any certificate issued by a private gateway to a public one is regarded as a Cargo Delivery Authorization (CDA), and it authorizes the public gateway to send messages to the issuer.

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
 
- Issue [_parcel delivery deauthorizations_ (PDDs)](rs000-core.md#pdd) for all its active PDAs.
- Securely destroy its private key.

### Gateway Certificate Revocation (GCR)

A gateway MAY revoke its own certificate by issuing a GCR message to its peer gateway(s). These messages MUST be serialized with RAMF, using the octet `0x11` as its _concrete message type_, and have an empty payload (i.e., one of length zero).

GCRs MUST be sent in the payload plaintext of a cargo, along with parcels, in order to prevent a malicious courier from identifying and dropping such messages.

## X.509 Extensions

### Basic Constraints

Each certificate MUST have its Basic Constraints extension as defined in the X.509 v3 specification. The extension MUST be marked as critical and its attributes MUST be set as follows depending on the type of certificate:

| Certificate type | `cA` | `pathLenConstraint` |
| --- | --- | --- |
| Self-issued gateway certificate | `true` | `2` |
| Non-self-issued gateway certificate (excluding CDAs) | `true` | `1` |
| Endpoint certificate (excluding PDAs) | `true` | `0` |
| Delivery authorization (PDAs and CDAs) | `false` | `0` |

### Authority Key Identifier

Except for self-issued certificates, all certificates MUST include the Authority Key Identifier extension as defined in the X.509 v3 specification.

### Subject Key Identifier

All certificates MUST include the Subject Key Identifier extension as defined in the X.509 v3 specification.
