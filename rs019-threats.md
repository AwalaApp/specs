---
permalink: /RS-019
---
# General Security Threats
{: .no_toc }

- Id: RS-019.
- Status: Working draft.
- Type: Informational.

## Abstract
{: .no_toc }

This document describes the general security threats that end users, service providers, relayers and software vendors should be aware of when implementing and using Relaynet. The likelihood and impact of each threat will depend entirely on the context where Relaynet is used, so this document does not constitute a threat model but it can be used in a threat modelling process.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction



Assumptions:

- The relevant specs were implemented as intended.

This document uses the term "providers" to refer to service providers, relayers and Relaynet software vendors collectively. Providers should conduct their own threat modelling process. as this document is only meant to be an input to their models.

This specification is a living document.

## Threats

| Summary | [STRIDE](https://en.wikipedia.org/wiki/STRIDE_(security)) Threat Type | Relaynet Node Type | Relevant Roles |
|:-:|:-:|:-:|:-:|
| [Software or Hardware Backdoor](#backdoor) | All | All | All |
| [Evil Maid Attack](#evil-maid) | All | All | All |
| [Detected, Unauthorized Physical Access](#detected-physical-access) | Spoofing, Tampering, Repudiation, Information disclosure, DoS | All | All |
| [Tampered Copy of Software](#tampered-software) | Spoofing, Tampering, Repudiation, Information disclosure, DoS | All | All |
| [DoS from Compromised, Legitimate Node](#dos-legitimate-node) | Spoofing, Tampering, Repudiation, Information disclosure, DoS | All | All |
| [DoS via Binding](#dos-binding) | DoS | All | All |
| [Public Endpoint Address Exposed to Gateway](#public-endpoint-address) | Information disclosure, DoS | Relaying gateway | End users |

### Software or Hardware Backdoor {#backdoor}

A backdoor at the Operating System, firmware or hardware level could allow an attacker to take full control of the host system, including the local Relaynet node(s). Such backdoors can be difficult to identify and they could, for example, achieve following:

- Compromise private keys to impersonate the node and decrypt messages, even from other devices.
- Copy the plaintext of incoming and outgoing messages.
- Drop incoming and outgoing messages.

On the other hand, some backdoors could have limited access to the host system or the Relaynet node. For example, it could be a process running as an unprivileged user different from the one running the Relaynet node. The damage that such backdoors can cause will depend on the host system and the implementation of the node, but one likely target of such attacks will be private keys. Consequently, software vendors should ensure that only the intended process has access to private keys; at a minimum, file permissions should prevent unauthorized access to the keys and they should also be encrypted at rest.

If backdoors are part of the threat model, providers should incorporate appropriate countermeasures in their plans, which should include educating end users on how to avoid these attacks.

### Evil Maid Attack {#evil-maid}

Could also lead to backdoor. The device should be tamper-evident, tamper-resistant or tamper-proof. Security token.

### Detected, Unauthorized Physical Access {#detected-physical-access}

Coercion or theft.

Self-destruct. Encryption at rest. Securely erase private keys and passwords from RAM when no longer needed. Security token.

### Tampered Copy of Software {#tampered-software}

An attacker could distribute tampered copies of Relaynet libraries, applications, endpoints or gateways in order to have a backdoor in the affected node.

Service providers and Relaynet software vendors should distribute their software in such a way that their users can be certain of its authenticity and integrity. The specific methods will depend on the nature of the software (e.g., mobile app, server-side app), but it will typically involve digital signatures.

### Denial of Service from Compromised, Legitimate Node {#dos-legitimate-node}

Rate limiting.

### Denial of Service via Message Transport Binding {#dos-binding}

Public endpoints and gateways are inherently susceptible to DoS attacks.

Monitoring.

### Public Endpoint Address Exposed to Gateway {#public-endpoint-address}

Gateways have to know the sender and the recipient of the parcels they relay, which means that they could find out the service that a parcel belongs to by looking at the public endpoint address (e.g., `rne+https://relaynet.twitter.com/`). Thanks to the use of end-to-end encryption in Relaynet, gateways would not be able to see the contents of the parcel; however, exposing the endpoint address could be cause for concern in some circumstances.

This information could be persisted and used subsequently, causing privacy issues -- And even more severe risks to certain end users, such as journalists. A relaying gateway operator may also be coerced to use this information to censor a particular service.

Users that wish to conceal the address of the centralized services they use should consider using a relaying gateway that does not persist or use that information in any way. They may do so by using the services of a third party provider or by hosting the gateway on a server under their control.

## Acknowledgements

Lightweight threat model in security audit by Include Security.
