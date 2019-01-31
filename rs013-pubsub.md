# Service Message Broadcast

- Id: RS-013.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This specification extends [Relaynet Core (RS-000)](rs000-core.md) to support the [Publish-Subscribe pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) in centralized and decentralized services.

## Introduction

- Think of this as a public message queue, with producers, consumers and queues/topics. Endpoints will tell their corresponding gateways which queues/topics they want to be subscribed to — So that means the cargo should also keep track of the queues/topics it contains.
- Use cases
  - In a world where the Twitter service is DTN-native, it could represent a tweet as a SDU (that is, one tweet is the sole content of a parcel). This would allow a political dissident to broadcast a message to their followers, so that the message can both (1) reach Twitter and (2) be read by anyone who gets hold of the parcel.
  - File distribution, especially software updates.

## Addressing

Syntax: `rnt:serviceId[/topicId[?queryString]]`. Useful to broadcast messages. The format of the serviceId depends on whether the service is centralised or decentralised:

- Centralised: A domain name or IP address.
- Decentralised: A random, 128-bit hex string.

PubSub parcels could be pushed to a blockchain. And a _relaying gateway_ could subscribe to the topics relevant to its peer gateways so they can relay relevant parcels.

## Messaging Protocols

- Broadcast parcels could be unencrypted (CMS type "data"), or encrypted with one or more certificates (CMS type "enveloped data").
- The [Cargo Collection Authorization](rs000-core.md#cargo-collection-authorization) MUST include zero or more topic subscriptions.
- A topic subscription is a structure that contains a topic address (potentially using glob patterns) and any conditions that the broadcast parcel must meet, such as having a specific origin endpoint (by address).
