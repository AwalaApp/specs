# Relaynet Public Key Infrastructure (RPKI) and Revocation Profile

- Id: RS-002.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This document describes how to issue, revoke and interpret X.509 certificates for endpoints and gateways in Relaynet.

## Attributes

The _Distinguished Name_ MUST only contain the _Common Name_ (CN), which MUST be set to the node's address (including its schema).

## Certificate Chains

An endpoint certificate MAY be issued by its gateway to generate a _parcel delivery authorization_. The gateway-endpoint relationship MUST be represented as a critical X.509 extension.

Endpoints and gateways MAY use a different keys for signing and decryption, which may be necessary in distributed systems. In this case, the certificate used for signing MUST be issued by the certificate used for decryption. This relationship MUST also be represented as a critical X.509 extension. 

## Certificate Rotation and Revocation

TODO
