---
permalink: /RS-002
---
# Awala Public Key Infrastructure and Revocation Profile
{: .no_toc }

- Id: RS-002.
- Status: Working draft.
- Type: Implementation.
- Issue tracking label: [`spec-pki`](https://github.com/AwalaNetwork/specs/labels/spec-pki).

## Abstract
{: .no_toc }

This document describes how to issue, revoke and process X.509 certificates in Awala [messaging protocols](rs000-core.md#messaging-protocols). Despite the use of X.509 certificates, this Public Key Infrastructure (PKI) profile is independent of and incompatible with the [Internet PKI profile](https://tools.ietf.org/html/rfc5280) as used in the TLS protocol.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

Awala relies extensively on its PKI in order to authenticate and authorize nodes without a real-time connection to an external authentication/authorization server, as well as to encrypt payloads when the [Channel Session Protocol](./rs003-key-agreement.md) is not employed.

One prominent use of the Awala PKI is in the [Awala Abstract Message Format (RAMF)](./rs001-ramf.md), where certificates are used to authenticate the sender of the message and ensure the integrity of the message. Any valid certificate can be used to sign a message bound for a public node, but every message bound for a private node has to be signed with a certificate issued by the recipient (in which case the certificate will be called a _delivery authorization_).

This PKI applies to the long-term keys that identify endpoints and gateways in Awala, and it also serves as the basis for issuing certificates for initial keys in the Channel Session Protocol. The requirements and recommendations in this document do not apply to the Internet PKI certificates used in [Message Transport Bindings](./rs000-core.md#message-transport-bindings).

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
        relaycorp(17) awala(0) pki(0) 0
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

Each gateway has at least two certificates for the same long-term key pair: One self-issued and one certificate issued by each of its peers. Consequently, every private gateway has exactly two certificates because it has exactly one peer, while a public gateway may have more certificates.

Self-issued certificates MUST only be used to issue certificates to peers, and therefore such certificates will be the root for a PDA or a [Cargo Delivery Authorization (CDA)](#cargo-delivery-authorization-cda). Self-issued certificates MUST NOT be used to sign channel or binding messages. Peers MAY use the self-issued certificate to encrypt payloads when not using the Channel Session Protocol.

Certificates issued by peers MUST be used to sign channel and binding messages like cargoes. A certificate issued by a private gateway to its public peer is known as a CDA, and additional requirements and recommendations apply.

### Cargo Delivery Authorization (CDA)

Any certificate issued by a private gateway to a public one is regarded as a Cargo Delivery Authorization (CDA), and it authorizes the public gateway to send cargo to the private gateway.

CDAs SHOULD be valid for at least 24 hours.

## Certificate Validity Period

Certificates MUST NOT be valid for more than 180 days.

## Certificate Rotation

Each node SHOULD rotate its certificate once half of the validity period has elapsed, in order to avoid disruption. For example, a certificate valid for 180 days should be rotated 90 days before its expiry date or soon thereafter.

Certificate rotation may cause a node to have multiple valid certificates for the same channel. When that happens, they should be used as follows:

- The certificate with the latest expiry date MUST be used to produce new digital signatures (e.g., certificate issuance, RAMF message signing).
- All valid certificates MUST be used to verify digital signatures (e.g., certification path verification, RAMF message integrity/authentication checks).

Nodes SHOULD delete certificates that are no longer valid.

A certificate rotation message MUST be serialized as a control message of concrete type `0x10` and be followed by the DER representation of the `CertificateRotation` ASN.1 type defined below:

```
CertificateRotation ::= SEQUENCE
{
  subjectCertificate OCTET STRING,
  chain              SET OF OCTET STRING
}
```

Where:

- `subjectCertificate` is the DER serialization of the newly-issued certificate for the peer.
- `chain` is the set of DER serializations for all the certificates in the chain. At a minimum, it MUST contain the issuer's certificate.

## Key Rotation

Endpoints and gateways MAY use multiple certificates with different asymmetric keys (and therefore different addresses), at any point in time, in order to facilitate key rotation.

## Certificate Revocation

Certificates MUST be revoked when their private keys are compromised, when a certificate/key rotation is complete, or when deemed appropriate by their subject or issuer.

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
