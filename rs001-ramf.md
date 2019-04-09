---
permalink: /RS-001
---
# Relaynet Abstract Message Format, Version 1
{: .no_toc }

- Id: RS-001.
- Status: Working draft.
- Type: Implementation.
- Proof of concept: https://github.com/relaynet/poc/blob/master/core/serialization.js

## Abstract
{: .no_toc }

This document defines version 1 of the Relaynet Abstract Message Format (RAMF), a binary format used to serialize messages in the endpoint and gateway messaging protocols of Relaynet. It also defines some basic requirements for any recipient of the message.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

RAMF messages are optimized to be processed on the fly with a single pass, without having to hold the entire message in memory. To achieve this, fields are framed with [length prefixes instead of delimiters](https://blog.stephencleary.com/2009/04/message-framing.html), and those fields that are used for routing, authentication and authorization are available near the start of the message.

## Format

A message is serialized using the following byte sequence ([little-endian](https://en.wikipedia.org/wiki/Endianness)):

1. [File format signature](https://en.wikipedia.org/wiki/List_of_file_signatures) (10 octets):
   - Prefix (8 octets): "Relaynet" in ASCII (hex: "52 65 6c 61 79 6e 65 74").
   - Concrete message format signature (1 octet).
   - Format version (1 octet). An 8-bit unsigned integer.
1. Signature hashing algorithm identifier. Defined early to allow the recipient to start calculating the message digest as the message is being streamed. This is an 8-bit unsigned integer (1 octet). See [Signature Hashing Algorithms](#signature-hashing-algorithms).
1. Recipient address. UTF-8 encoded, and length-prefixed with a 16-bit unsigned integer (2 octets). Consequently, the address can be as long as 255 characters.
1. Sender certificate (chain).
   - DER encoded.
   - Length-prefixed with 13-bit unsigned integer (2 octets), so the maximum length is ~8kib.
1. Message id. Unique to the sender. This is an opaque value, so it has no structure or semantics. This field is ASCII encoded and length-prefixed with 16-bit unsigned integer (2 octets).
1. Date. Creation date of the message (in UTC), represented as the number of seconds since Unix epoch. This is serialized as a 32-bit unsigned integer (4 octets), so it is not susceptible to the [Year 2038 Problem](https://en.wikipedia.org/wiki/Year_2038_problem).
1. Time to live (TTL). When the message expires.
   - Number of seconds since Date.
   - Zero means the message does not expire.
   - 24-bit, unsigned integer (3 octets). So maximum is over 6 months.
1. Payload. Contains the [service data unit](https://en.wikipedia.org/wiki/Service_data_unit) encoded with the [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652). The [ciphertext](https://en.wikipedia.org/wiki/Ciphertext) MUST be length-prefixed with a 32-bit unsigned integer (4 octets), so the maximum length is ~3.73GiB.
1. Signature. The sender's [detached signature](https://en.wikipedia.org/wiki/Detached_signature) for all the preceding octets in the message, from the format signature to the payload. This is at the bottom to make it easy to generate and process messages with a single pass.
   - Serialized as a [CMS signed data](https://tools.ietf.org/html/rfc5652#section-5) value where:
     - `digestAlgorithms`, the collection of message digest algorithm identifiers, MUST contain exactly one OID and it MUST correspond to the signature hashing algorithm specified in the message. See [Signature Hashing Algorithms](#signature-hashing-algorithms).
     - `encapContentInfo`, the signed content, MUST NOT include the content itself, since this is a detached signature.
     - `certificates` MUST be empty, since the sender's certificate is available above.
     - `crls` MUST be empty, since certificate revocation is part of the [Relaynet PKI](rs002-pki.md).
     - `signerInfos` MUST contain exactly one signer (`SignerInfo`), where:
       - `sid`
       - `digestAlgorithm`
       - `signedAttrs`
       - `signatureAlgorithm`
       - `signature`
       - `unsignedAttrs`
   - The ciphertext is length-prefixed with a 12-bit unsigned integer (2 octets), so the maximum length is 4kib.

### Signature Hashing Algorithms

RAMF messages MUST be signed using one of the following hashing algorithms. The sender MAY only support one of those algorithms, but any implementation that processes RAMF messages MUST support all the algorithms below.

| Algorithm | 8-Bit Identifier in RAMF | OID in CMS |
| --- | --- | --- |
| SHA-256 | `0x00` | [2.16.840.1.101.3.4.2.1](http://www.oid-info.com/get/2.16.840.1.101.3.4.2.1) |
| SHA-512 | `0x01` | [2.16.840.1.101.3.4.2.3](http://www.oid-info.com/get/2.16.840.1.101.3.4.2.3) |

## Post-Deserialization Validation

Recipients and brokers of a RAMF message MUST validate the message as soon as possible, before any further processing or relay. At a minimum, they MUST:

- Check the date and TTL to make sure the message is still valid (and mitigate replay attacks).
- Check that the sender certificate is valid per [Relaynet PKI](rs002-pki.md).
- Check that the date is within the period of time during which the certificate was valid.
- The signature MUST be valid according to the specified signature algorithm. The signature MUST be deemed invalid if the signature algorithm is unsupported, the hashing algorithm is unsupported or the hashing algorithm does not match that of the RAMF message.

## Security Considerations

To avoid replay attacks, the message id SHOULD be persisted until the TTL expires, and until then, reject any incoming message from the same sender and the same id.

Nodes can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md). 

## Reserved Concrete Message Format Signatures

The following concrete signatures have been reserved by other Relaynet specifications:

- `0x10` for [certificate rotation](rs002-pki.md#certificate-and-key-rotation).
- `0x11` for [gateway certificate revocation](rs002-pki.md#gateway-certificate-revocation-gcr).
- `0x43` ("C" in ASCII) for [cargoes](rs000-core.md#cargo).
- `0x44` for [cargo collection authorizations](rs000-core.md#cargo-collection-authorization-cca).
- `0x50` ("P" in ASCII) for [parcels](rs000-core.md#parcel).

## Open Questions

- PCKS7 is much more widely supported than CMS. Should we downgrade to PCKS7? If so, then this format will have to be updated to hold the [key agreement](rs003-key-agreement.md) information (more specifically, the public component of the ephemeral key and some metadata), since PKCS7 enveloped-data does not support encryption using key agreement algorithms.
