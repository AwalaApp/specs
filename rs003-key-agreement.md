# Key Agreement Protocol

- Id: RS-003.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This document will describe a key agreement protocol to establish and protect sessions between gateways or between endpoints, in order to provide [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) and mitigate replay attacks.

It will be based on the [Extended Triple Diffie-Hellman protocol](https://signal.org/docs/specifications/x3dh/) and the [double ratchet algorithm](https://signal.org/docs/specifications/doubleratchet/), with two notable differences:

- There is no central server that can provide certificates or public keys for any node in Relaynet, but that is not necessary because peers always have each other's certificates.
- There MUST be **multiple** ephemeral keys that are valid at any point in time, because messages can arrive late or out of order, or they may be lost. 
