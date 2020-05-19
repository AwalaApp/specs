---
permalink: /RS-019
---
# General Security Threats
{: .no_toc }

- Id: RS-019.
- Status: Working draft.
- Type: Informational.
- Issue tracking label: [`spec-threats`](https://github.com/relaynet/specs/labels/spec-threats).

## Abstract
{: .no_toc }

This document describes the general security threats that end users, service providers, couriers and software vendors should be aware of when implementing and using Relaynet. The likelihood and impact of each threat will depend entirely on the context where Relaynet is used, so this document does not constitute a threat model but it can be used in a threat modelling process.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

Nothing has ever been more important throughout the design of Relaynet than the security, safety and privacy of end users -- After all, it was conceived to help vulnerable people fight back against repressive regimes. However, no system is impenetrable and Relaynet may be targeted by powerful adversaries as its userbase grows, so the best way to mitigate the risks is by being upfront and open about the threats, which this document summarizes.

This is meant to be a living document, unlike other Relaynet specifications. Only threats that are inherent to the implementation and use of the technology are eligible to be listed here -- Therefore, specific security vulnerabilities should be fixed in their corresponding implementations and/or specifications.

The term "providers" is used throughout the specification to refer to service providers, couriers and Relaynet software vendors collectively. Providers should conduct their own threat modelling process as this document is only meant to be an input to their models. Two elements that are notably missing from this document are the likelihood and impact of each threat, which should be identified in each threat model.

Despite the recommendations to address the various threats, this specification remains informational and providers are not required to adopt such recommendations. Providers are strongly encouraged to adopt the most appropriate countermeasures to their specific circumstances.

## Threats

| Title | [STRIDE](https://en.wikipedia.org/wiki/STRIDE_(security)) Threat Type | Relaynet Node Type | Relevant Roles |
|:-:|:-:|:-:|:-:|
| [Private Keys Compromise](#private-keys-compromise) | Spoofing, Tampering, Repudiation, Information disclosure | All | All |
| [Software or Hardware Backdoor](#backdoor) | All | All | All |
| [Tampered Copy of Software](#tampered-software) | Spoofing, Tampering, Repudiation, Information disclosure, DoS | All | All |
| [DoS from Legitimate Node](#dos-legitimate-node) | DoS | All | All |
| [DoS via Binding](#dos-binding) | DoS | All | All |
| [Public Endpoint Address Exposed to Gateway](#public-endpoint-address) | Information disclosure, DoS | Public gateway | End users |

### Private Keys Compromise {#private-keys-compromise}

Relaynet uses asymmetric encryption extensively, with both long-term and ephemeral key pairs. An attacker may want to have access to the private keys so that they can impersonate the node and/or decrypt messages from other devices. They could use methods such as theft, evil maid attacks, cold boot attacks or [backdoors](#backdoors) to achieve this.

Full-disk encryption, which is generally advisable, is a good starting point to mitigate this attack. Hardware-level methods like security tokens and tamper-evidence technology may also be appropriate in some circumstances.

To protect private keys, software vendors should set appropriate file permissions to prevent unauthorized access, use encryption at rest and also securely remove private keys from RAM when they are no longer needed. In some cases, it may also be appropriate to offer a duress code mechanism that users can activate to remove any trace of the software and its data.

### Software or Hardware Backdoor {#backdoor}

A backdoor at the Operating System, firmware or hardware level could allow an attacker to take full control of the host system, including the local Relaynet node(s). Such backdoors can be difficult to identify and they could, for example, achieve the following:

- [Compromise the private keys](#private-keys-compromise).
- Copy the plaintext of incoming and outgoing messages.
- Drop incoming and outgoing messages.

If backdoors are part of the threat model, providers should incorporate appropriate countermeasures in their plans, which should include educating end users on how to avoid these attacks.

### Tampered Copy of Software {#tampered-software}

An attacker could distribute tampered copies of Relaynet libraries, applications, endpoints or gateways in order to have a backdoor in the affected node.

Service providers and Relaynet software vendors should distribute their software in such a way that their users can be certain of its authenticity and integrity. The specific methods will depend on the nature of the software (e.g., mobile app, server-side app), but it will typically involve digital signatures.

### Denial of Service from Legitimate Node {#dos-legitimate-node}

Attackers may use freshly-generated, private addresses to conduct a DoS attack against public nodes, or use [compromised private keys](#private-keys-compromise) to achieve the same attack against private and public nodes. This would be an attack at the [channel](rs000-core.md#messaging-protocols) level.

Providers should have monitoring in place to detect ongoing DoS attacks, as well as an appropriate response plan. They should also employ appropriate rate limiting constraints; for example, private endpoints could issue [Parcel Delivery Authorizations (PDAs)](rs002-pki.md#parcel-delivery-authorization-pda) that employ the [rate limiting extension](rs002-pki.md#rate-limiting-extension).

### Denial of Service via Message Transport Binding {#dos-binding}

Public endpoints and gateways are inherently susceptible to DoS attacks at the [binding](rs000-core.md#message-transport-bindings) level. Such attacks could be distributed (DDoS) or could come from a single origin. [SYN](https://en.wikipedia.org/wiki/SYN_flood) and [application-layer floods](https://en.wikipedia.org/wiki/Denial_of_Service_attack#Application-layer_floods) are examples of such attacks.

Providers should have the appropriate network-level protections in place, such as network- and application-level firewalls. They should also have adequate monitoring to detect the attack and an adequate plan to respond to it. Where possible, public gateways should only accept [Cargo Relay Connections](rs000-core.md#cargo-relay-binding) from a set of whitelisted IP addresses.

An attacker may also target the DNS records of endpoints and gateways, also causing a denial of service. Even though the sender will continue to retry the delivery of the parcel or cargo until the target server uses a valid TLS certificate, the attack may last until some messages expire, forcing the sender to drop them.

### Public Endpoint Address Exposed to Gateway {#public-endpoint-address}

Gateways have to know the sender and the recipient of the parcels they relay, which means that they could find out the service that a parcel belongs to by looking at the public endpoint address (e.g., `rne+https://relaynet.twitter.com/`). Thanks to the use of end-to-end encryption in Relaynet, gateways would not be able to see the contents of the parcel; however, exposing the endpoint address could be cause for concern in some circumstances.

This information could be persisted and used subsequently, causing privacy issues -- And even more severe risks to certain end users, such as journalists. A public gateway operator may also be coerced to use this information to censor a particular service.

Users that wish to conceal the address of the centralized services they use should consider using a public gateway that does not persist or use that information in any way. They may do so by using the services of a third party provider or by hosting the gateway on a server under their control.

## Acknowledgements

This work is based on a lightweight threat analysis by [Include Security](http://www.includesecurity.com/) as part of a [security audit commissioned by the Open Technology Fund](https://relaynet.network/archives/security-audit-2019-03.pdf).
