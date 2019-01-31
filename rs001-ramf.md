# Relaynet Abstract Message Format, Version 1

- Id: RS-001.
- Status: Working draft.
- Type: Implementation.
- Proof of concept: https://github.com/relaynet/poc/blob/master/core/serialization.js

## Abstract

This document defines version 1 of the Relaynet Abstract Message Format (RAMF), a binary format used to serialize messages in the endpoint and gateway messaging protocols of Relaynet. It also defines some basic requirements for any recipient of the message.

## Introduction

RAMF messages are optimized to be processed on the fly with a single pass, without having to hold the entire message in memory. To achieve this, fields are framed with [length prefixes instead of delimiters](https://blog.stephencleary.com/2009/04/message-framing.html), and those fields that are used for routing, authentication and authorization are available near the start of the message.

## Format

A message is serialized using the following byte sequence ([little-endian](https://en.wikipedia.org/wiki/Endianness)):

1. [File format signature](https://en.wikipedia.org/wiki/List_of_file_signatures) (10 octets):
   - Prefix (8 octets): "Relaynet" in ASCII (hex: "52 65 6c 61 79 6e 65 74").
   - Concrete message format signature (1 octet).
   - Format version (1 octet). An 8-bit unsigned integer.
1. Signature hashing algorithm, defined early to allow the recipient to start calculating the message digest as the message is being streamed. This is an ASCII with a fixed length of 8 octets, padded with `0x00` octets at the end if fewer octets are needed.
1. Recipient address. UTF-8 encoded, and length-prefixed with a 16-bit unsigned integer (2 octets). Consequently, the address can be as long as 256 characters.
1. Sender certificate (chain).
   - DER encoded.
   - Length-prefixed with 13-bit unsigned integer (2 octets), so the maximum length is ~8kib.
1. Message id. Unique to the sender. This is an opaque value, so it has no structure or semantics. This field is ASCII encoded and length-prefixed with 16-bit unsigned integer (2 octets).
1. Date. Creation date of the message (in UTC), represented as the number of seconds since Unix epoch. This is serialized as a 32-bit unsigned integer (4 octets), so it is not susceptible to the [Year 2038 Problem](https://en.wikipedia.org/wiki/Year_2038_problem).
1. Time to live (TTL). When the message expires.
   - Number of seconds since Date.
   - Zero means the message does not expire.
   - 24-bit, unsigned integer (3 octets). So maximum is over 6 months.
1. Payload. Encoded as a [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652) [enveloped data](https://tools.ietf.org/html/rfc5652#section-6) with exactly one recipient and using the [Relaynet Key Agreement protocol](rs003-key-agreement.md). The ciphertext is length-prefixed with a 32-bit unsigned integer (4 octets), so the maximum length is ~3.73GiB.
1. Signature.
   - As [CMS signed data](https://tools.ietf.org/html/rfc5652#section-5) structure with exactly one signer and zero embedded certificates. The signer is the sender, and its certificate is available above.
   - The cleartext to the signature should be the entire message, from the format signature to the payload.
   - This is at the bottom to make it easy to generate and consume messages with a single pass.
   - The ciphertext is length-prefixed with a 12-bit unsigned integer (2 octets), so the maximum length is 4kib.

## Post-deserialization validation

Recipients and brokers of a RAMF message MUST validate the message as soon as possible, before any further processing or relay. At a minimum, they MUST:

- Check the signature.
- Check that the sender certificate is valid.
- Check the date and TTL to make sure the message is still valid (and mitigate replay attacks).

To avoid replay attacks, the message id SHOULD be persistent until the TTL expires, and until then, reject any incoming message from the same sender and the same id.

Nodes can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md). 
