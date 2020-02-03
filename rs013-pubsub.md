---
nav_exclude: true
permalink: /RS-013
---
# Service Message Broadcast
{: .no_toc }

- Id: RS-013.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This specification will extend [Relaynet Core (RS-000)](rs000-core.md) to support the [Publish-Subscribe pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) through a [Distributed Hash Table (DHT)](https://en.wikipedia.org/wiki/Distributed_hash_table). This document is a just placeholder because this functionality is not a top priority as of this writing.

## Overview

- Endpoints will tell their corresponding gateways which senders and/or topics they want to be subscribed to.
- Gateways will publish _broadcast parcels_ for other gateways in the network to consume such parcels and pass them on to their endpoints.
- Parcels will be deleted from the DHT as soon as they expire.
- Use cases:
  - A social network could use this mechanism to distribute posts, especially those that have to reach millions of users.
  - Software update notifications.

## Addressing

Syntax: `rnt:serviceId[/topicId[?queryString]]`. The format of the `serviceId` depends on whether the service is centralised or decentralised:

- Centralised: A domain name or IP address.
- Decentralised: A random, 128-bit hex string.

A _relaying gateway_ could subscribe to the topics relevant to its peer gateways so they can relay relevant parcels.

## Messaging Protocols

- Broadcast parcels could be unencrypted (CMS type "data"), or encrypted with one or more certificates (CMS type "enveloped data").
- The [Cargo Collection Authorization](rs000-core.md#cca) MUST include zero or more topic subscriptions.
- A topic subscription is a structure that contains a topic address (potentially using glob patterns) and any conditions that the broadcast parcel must meet, such as having a specific origin endpoint (by address).
