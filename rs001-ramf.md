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
   1. Prefix (8 octets): "Relaynet" in ASCII (hex: "52 65 6c 61 79 6e 65 74").
   1. Concrete message format signature (1 octet).
   1. Format version (1 octet). An 8-bit unsigned integer.
1. Signature hashing algorithm identifier. Defined early to allow the recipient to start calculating the message digest as the message is being streamed. The algorithm MUST be valid per [RS-018](rs018-algorithms.md). This value MUST be DER-encoded as an ASN.1 Object Identifier; for example, SHA-256 (OID `2.16.840.1.101.3.4.2.1`) would be encoded as `06 09 60 86 48 01 65 03 04 02 01`. It MUST also have a fixed length of 16 octets, right padded with `0x00`.
1. Recipient address. UTF-8 encoded, and length-prefixed with a 16-bit unsigned integer (2 octets). Consequently, the address can be as long as 255 characters.
1. Sender certificate. It MUST be DER-encoded and length-prefixed with 12-bit unsigned integer (2 octets), so the maximum length is ~4kib. It MUST also comply with the [Relaynet PKI](rs002-pki.md).
1. Message id. Unique to the sender. This is an opaque value, so it has no structure or semantics. This field is ASCII encoded and length-prefixed with 16-bit unsigned integer (2 octets).
1. Date. Creation date of the message (in UTC), represented as the number of seconds since Unix epoch. This is serialized as a 32-bit unsigned integer (4 octets), so it is not susceptible to the [Year 2038 Problem](https://en.wikipedia.org/wiki/Year_2038_problem).
1. Time to live (TTL). Number of seconds during which the message is valid, starting from the date in the Date field. Zero (`0`) means the message does not expire. The value MUST be encoded as a 24-bit, unsigned integer (3 octets), so maximum TTL is just over 6 months.
1. Payload. Contains the [service data unit](https://en.wikipedia.org/wiki/Service_data_unit) encoded with the [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652). The [ciphertext](https://en.wikipedia.org/wiki/Ciphertext) MUST be length-prefixed with a 32-bit unsigned integer (4 octets), so the maximum length is ~3.73GiB.
1. Signature. The sender's [detached signature](https://en.wikipedia.org/wiki/Detached_signature) to validate the integrity and authenticity of the message. This is at the bottom to make it easy to generate and process messages with a single pass.
   - The plaintext MUST be the entire RAMF message before the signature.
   - The ciphertext MUST be encapsulated as a valid [CMS signed data](https://tools.ietf.org/html/rfc5652#section-5) value where:
     - `digestAlgorithms`, the collection of message digest algorithm identifiers, MUST contain exactly one OID and it MUST correspond to the signature hashing algorithm specified in the message.
     - `encapContentInfo`, the signed content, MUST NOT include the content itself, since this is a detached signature.
     - `certificates` SHOULD contain the chain of the sender certificate, if applicable. The sender certificate itself SHOULD NOT be included.
     - `crls` MUST be empty, since certificate revocation is part of the [Relaynet PKI](rs002-pki.md).
     - `signerInfos` MUST contain exactly one signer (`SignerInfo`), where `digestAlgorithm` MUST match that of the RAMF message and `signatureAlgorithm` MUST be valid per [RS-018](rs018-algorithms.md).
   - The CMS value MUST be length-prefixed with a 13-bit unsigned integer (2 octets), so the maximum length is 8kib.

## Post-Deserialization Validation

Recipients and brokers of a RAMF message MUST validate the message as soon as it is received, before any further processing or relay. The message MUST be refused when any of the conditions below is not met:

- The sender certificate MUST be valid per [Relaynet PKI](rs002-pki.md).
- The message date MUST NOT be more than 5 minutes in the future.
- The message TTL MUST NOT be more than 5 minutes in the future.
- The message date MUST be within the period of time during which the sender certificate was valid.
- The signature MUST be valid according to the [CMS verification process](https://tools.ietf.org/html/rfc5652#section-5.6) and the specified signature algorithm. Additionally, the signature MUST be deemed invalid if the signature algorithm is unsupported, the hashing algorithm is unsupported or the hashing algorithm does not match that of the RAMF message.

The purpose of the grace period in the date and TTL fields is to account for a potential clock drift.

## Security Considerations

To avoid replay attacks, the message id SHOULD be persisted until the TTL expires, and until then, reject any incoming message from the same sender and the same id.

Nodes can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md).

Note that all cryptographic algorithms MUST comply with [RS-018](rs018-algorithms.md).

## Reserved Concrete Message Format Signatures

The following concrete signatures have been reserved by other Relaynet specifications:

- `0x10` for [certificate rotation](rs002-pki.md#certificate-and-key-rotation).
- `0x11` for [gateway certificate revocation](rs002-pki.md#gateway-certificate-revocation-gcr).
- `0x43` ("C" in ASCII) for [cargoes](rs000-core.md#cargo).
- `0x44` for [cargo collection authorizations](rs000-core.md#cargo-collection-authorization-cca).
- `0x50` ("P" in ASCII) for [parcels](rs000-core.md#parcel).

## Open Questions

- Should the payload ciphertext be detached from the CMS EnvelopedData? That'd make it easier to encrypt/decrypt as a stream.
