---
permalink: /RS-018
---
# Relaynet Cryptographic Algorithms, Version 1
{: .no_toc }

- Id: RS-018.
- Status: Working draft.
- Type: Implementation.

## Abstract
{: .no_toc }

This document specifies the requirements and recommendations for the support and selection of cryptographic algorithms in Relaynet. Every implementation of the Relaynet protocol suite is required to comply with this specification.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

The primary purpose of this document is to prevent the use of insecure cryptographic algorithms and to limit the set of supported algorithms for interoperability reasons. This specification only requires or recommends algorithms that have no known vulnerabilities, are well supported and are unencumbered by patents.

In each category, exactly one algorithm is required in order to achieve interoperability. Required algorithms are [assumed to be secure until at least 2030](https://www.keylength.com/en/4/). Recommended algorithms are assumed to be secure beyond 2030, but are more expensive to run and/or not widely supported.

This document aims to reflect industry best practices as of 2019, and is expected to be replaced eventually to reflect future developments. One change that can be anticipated is the exclusive use of Elliptic Curve Cryptography (ECC) for asymmetric key cryptography once support for the recommended curves is widespread.

## RSA Key Constraints

Implementations MUST support 2048-bit RSA keys. They SHOULD also support 3072-bit and 4096-bit RSA keys. RSA keys with fewer than 2048 bits MUST NOT be supported.

## Required and Recommended Algorithms

Each supported algorithm is accompanied with its corresponding Object ID (OID) when available, as required by the [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652), which is extensively used in Relaynet.

For security and compatibility reasons, Relaynet implementations SHOULD NOT support any algorithm that is not explicitly required or recommended in this document, unless the implementer has a very strong cryptographic background and very strong reasons to use other algorithms.

### Cryptographic Hash Functions

Implementations MUST support SHA-256 (OID `2.16.840.1.101.3.4.2.1`) and they SHOULD also support SHA-384 (OID `2.16.840.1.101.3.4.2.2`) and SHA-512 (OID `2.16.840.1.101.3.4.2.3`). They MUST NOT support MD5 or SHA-1 for security reasons.

### Symmetric Ciphers

Implementations MUST support AES-128, and they SHOULD also support AES-192 and AES-256. They MUST NOT support DES for security reasons.

More specifically, [Key Wrap mode](https://tools.ietf.org/html/rfc3394.html) MUST be used when encrypting cryptographic key materials and [GCM](https://tools.ietf.org/html/rfc5084) MUST be used when encrypting payloads. Consequently, the following ciphers are required or recommended:

- AES-128-KW (required, OID `2.16.840.1.101.3.4.1.5`).
- AES-192-KW (recommended, OID `2.16.840.1.101.3.4.1.25`).
- AES-256-KW (recommended, OID `2.16.840.1.101.3.4.1.45`).
- AES-128-GCM (required, OID `2.16.840.1.101.3.4.1.6`).
- AES-192-GCM (recommended, OID `2.16.840.1.101.3.4.1.26`).
- AES-256-GCM (recommended, OID `2.16.840.1.101.3.4.1.46`).

### Asymmetric Ciphers

Implementations MUST support RSA-OAEP (OID `1.2.840.113549.1.1.7`). They SHOULD also support Curve25519 (OID `1.3.101.110`), and they MAY support Curve448 (OID `1.3.101.111`).

### Digital Signature Algorithms

Implementations MUST support RSA-PSS (OID `1.2.840.113549.1.1.10`). They SHOULD also support Ed25519 EdDSA keys (OID `1.3.101.112`), and they MAY support Ed448 EdDSA keys (OID `1.3.101.113`).

### Key Exchange Algorithms

Implementations MUST support Elliptic Curve Diffie-Hellman (ECDH; OID `1.3.132.1.12`) with the NIST P-256 curve (OID `1.2.840.10045.3.1.7`). They SHOULD also support the NIST curves P-384 (OID `1.3.132.0.34`) and P-521 (OID `1.3.132.0.35`).

Implementations MAY also support the curves X25519 (OID `1.3.101.110`) and X448 (OID `1.3.101.111`).

Finite field Diffie-Hellman (DH; OID `1.2.840.113549.1.3.1`) MAY also be supported, in which case implementations SHOULD only support DH groups from [RFC3526](https://tools.ietf.org/html/rfc3526) with at least 2048 bits. DH groups under 2048 bits MUST NOT be supported.

## Algorithm Selection

Relaynet-compliant software SHOULD default to the required algorithms for interoperability reasons, but they MAY allow systems administrators or advanced end users to use algorithms that offer better security guarantees.
