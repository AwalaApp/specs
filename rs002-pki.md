# Relaynet Public Key Infrastructure and Revocation Profile

- Id: RS-002.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This document describes how to issue, distribute, store, revoke and interpret X.509 certificates in Relaynet [messaging protocols](rs000-core.md#messaging-protocols). Despite the use of X.509 certificates, this PKI profile is independent of and incompatible with the [Internet PKI profile](https://tools.ietf.org/html/rfc5280).

## Attributes

The _Distinguished Name_ MUST only contain the _Common Name_ (CN), which MUST be set to the node's address (including its schema).

## Certificate Chains

An endpoint certificate MAY be issued by its gateway to generate a _parcel delivery authorization_. The gateway-endpoint relationship MUST be represented as a critical X.509 extension.

Endpoints and gateways MAY use a different keys for signing and decryption, which may be necessary in distributed systems. In this case, the certificate used for signing MUST be issued by the certificate used for decryption. This relationship MUST also be represented as a critical X.509 extension. 

## Certificate Rotation and Revocation

TODO
