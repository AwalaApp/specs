# Service Message Broadcast

- Id: RS-013.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This specification will extend [Relaynet Core (RS-000)](rs000-core.md) to support the [Publish-Subscribe pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) in centralized and decentralized services. This document is a just placeholder because this functionality is not a top priority as of this writing.

## Introduction

- Endpoints will tell their corresponding gateways which topics they want to be subscribed to.
- Gateways will push _broadcast parcels_ to a distributed database, for other gateways in the network to consume such parcels and pass them on to their endpoints.
- The database will be analogous to a "mutable blockchain". Obviously, blockchains are immutable by definition, so the term isn't technically correct. This would be a system with the following properties:
  - Peer-to-peer. No central authority.
  - Nodes will be responsible for storing valid parcels until they expire.
  - Node operators should ideally be rewarded for hosting parcels.
  - It may be necessary to support sharding, especially if the parcels aren't limited to just a few kilobytes.
- Use cases
  - A social network could use this mechanism to distribute posts, especially those that have to reach millions of users.
  In a world where the Twitter service is Relaynet-native, it could represent a tweet as a [SDU](https://en.wikipedia.org/wiki/Service_data_unit) (that is, one tweet is the sole content of a parcel). This would allow a political dissident to broadcast a message to their followers, so that the message can both (1) reach Twitter and (2) be read by anyone who gets hold of the parcel.
  - Software update notifications.

## Addressing

Syntax: `rnt:serviceId[/topicId[?queryString]]`. Useful to broadcast messages. The format of the serviceId depends on whether the service is centralised or decentralised:

- Centralised: A domain name or IP address.
- Decentralised: A random, 128-bit hex string.

A _relaying gateway_ could subscribe to the topics relevant to its peer gateways so they can relay relevant parcels.

## Messaging Protocols

- Broadcast parcels could be unencrypted (CMS type "data"), or encrypted with one or more certificates (CMS type "enveloped data").
- The [Cargo Collection Authorization](rs000-core.md#cargo-collection-authorization-cca) MUST include zero or more topic subscriptions.
- A topic subscription is a structure that contains a topic address (potentially using glob patterns) and any conditions that the broadcast parcel must meet, such as having a specific origin endpoint (by address).
