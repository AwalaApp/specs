---
permalink: /RS-001
---
# Relaynet Abstract Message Format, Version 1
{: .no_toc }

- Id: RS-001.
- Status: Working draft.
- Type: Implementation.
- Issue tracking label: [`spec-ramf`](https://github.com/relaynet/specs/labels/spec-ramf).

## Abstract
{: .no_toc }

This document defines version 1 of the _Relaynet Abstract Message Format_ (RAMF), a binary format used to serialize Relaynet [channel](./rs000-core.md#messaging-protocols) messages. RAMF is based on the [ASN.1 Distinguished Encoding Rules](https://www.itu.int/rec/T-REC-X.680-X.693-201508-I/en) (DER) and the [Cryptographic Message Syntax](https://tools.ietf.org/html/rfc5652). It also defines a series of requirements for recipients and intermediaries processing such messages.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

Messages exchanged within an endpoint or gateway channel require metadata attached to their payload so that gateways and couriers processing such messages can verify the validity of the message and then deliver the verified message to the next node on the route.

Every endpoint and gateway channel message is serialized as a _RAMF message_, which encapsulates the payload along with relevant metadata to be used for routing, authentication and authorization purposes. The message payload and metadata, collectively known as the _message fields_, are serialized as an ASN.1 DER value.

A RAMF message begins with a sequence of octets, collectively known as _the format signature_, which specify the type of the message (e.g., a parcel) and its format version (e.g., version 1). The format signature is followed by a DER-encoded [CMS Signed-data](https://tools.ietf.org/html/rfc5652#section-5) value that encapsulates the message fields and the digital signature (including the [Relaynet PKI](rs002-pki.md) certificate of the sender).

By specifying the message type and version in the format signature, future RAMF versions could use different serialization formats, including formats incompatible with ASN.1.

## Format

The format signature MUST span the first 10 octets of the message, representing the following sequence (encoded in little-endian):

1. Prefix (8 octets): "Relaynet" in ASCII (hex: `52 65 6c 61 79 6e 65 74`).
1. Concrete message type (1 octet).
1. Concrete message format version (1 octet). This MUST be an 8-bit unsigned integer. 

The format signature MUST be followed by a DER-encoded CMS signed-data value where:

  - `digestAlgorithms`, the collection of message digest algorithm identifiers, MUST contain exactly one OID and it MUST correspond to a valid algorithm per [RS-018](rs018-algorithms.md).
  - `encapContentInfo`, the signed content, MUST include signed ciphertext -- In this case, the message fields.
  - `certificates` MUST contain the sender's certificate and it SHOULD also include the rest of the certificates in the chain. All certificates MUST comply with the [Relaynet PKI](rs002-pki.md).
  - `crls` MUST be empty, since certificate revocation is part of the [Relaynet PKI](rs002-pki.md).
  - `signerInfos` MUST contain exactly one signer (`SignerInfo`), and whose `signatureAlgorithm` MUST be valid per [RS-018](rs018-algorithms.md).

The message fields MUST be represented as the DER serialization of the ASN.1 `RAMF` type below:

```
{% include_relative diagrams/rs001/ramf.asn1 %}
```

Where the items in the `RAMFMessage` sequence are defined as follows:

- `recipientAddress` MUST be the public or private address of the recipient. It MUST NOT span more than 1024 octets.
- `messageId` MUST be the unique identifier assigned to this message by its sender. It MUST NOT span more than 256 octets.
- `creationTimeUtc` MUST be the creation date of the message (in UTC).
- `ttl` MUST represent the time-to-live of the message -- That is, the number of seconds since `creationTimeUtc` during which the message is regarded valid. It MUST NOT be less than zero or greater than 15552000 (180 days).
- `payload` MUST be the [service data unit](https://en.wikipedia.org/wiki/Service_data_unit) encapsulated in a DER-encoded [Cryptographic Message Syntax (CMS)](https://tools.ietf.org/html/rfc5652) value (e.g., Enveloped-data). This field MUST not span more than 8388608 octets (8 MiB).

A RAMF message MUST NOT span more than 9437184 octets (9 MiB).

## Post-Deserialization Validation

Recipients and brokers of a RAMF message MUST validate the message as soon as it is received, before any further processing or relay. The message MUST be refused when any of the conditions below is not met:

- The message date MUST NOT be in the future.
- The message TTL MUST NOT resolve to a date in the past.
- The message date MUST be within the period of time during which the sender certificate was valid.
- All certificates MUST be valid per [Relaynet PKI](rs002-pki.md).
- The signature MUST be valid according to the [CMS verification process](https://tools.ietf.org/html/rfc5652#section-5.6) and the specified signature algorithm. Additionally, the signature MUST be deemed invalid if the signature algorithm is unsupported or the hashing algorithm is unsupported.

## Security Considerations

To avoid replay attacks, the message id SHOULD be persisted until the TTL expires, and until then, reject any incoming message from the same sender and the same id.

Nodes can further protect from replay attacks, amongst other attack vectors, by establishing a secure session with the [Relaynet Key Agreement Protocol](rs003-key-agreement.md).

Note that all cryptographic algorithms MUST comply with [RS-018](rs018-algorithms.md).

## Reserved Concrete Message Types

The following concrete types have been reserved by other Relaynet specifications:

- `0x10` for [certificate rotation](rs002-pki.md#certificate-and-key-rotation).
- `0x11` for [gateway certificate revocation](rs002-pki.md#gateway-certificate-revocation-gcr).
- `0x43` ("C" in ASCII) for [cargoes](rs000-core.md#cargo).
- `0x44` for [cargo collection authorizations](rs000-core.md#cca).
- `0x50` ("P" in ASCII) for [parcels](rs000-core.md#parcel).
- `0x51` for [parcel collection acknowledgments](rs000-core.md#pca).
