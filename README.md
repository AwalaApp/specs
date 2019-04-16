# Relaynet Protocol Suite Specifications

This repository contains all the specifications part of [Relaynet](https://relaynet.link/). If you're new to Relaynet, you may want to start by watching the [demo of the proof of concept with Twitter](https://www.youtube.com/watch?v=fi_RKwmrXIY).

The following specifications provide the foundation of the network and are therefore the top priority of the project. At this point, the best way to contribute to the project is by providing feedback on these specs.

- [RS-000 (Relaynet Core)](rs000-core.md) defines the foundation of the protocol suite.
- [RS-001 (RAMF)](rs001-ramf.md) defines the _Relaynet Abstract Message Format_, an efficient binary format used to serialize messages.
- [RS-002 (Relaynet PKI)](rs002-pki.md) defines how to use the certificates for endpoints and gateways.
- [RS-003 (Key Agreement)](rs003-key-agreement.md) defines the key agreement protocol to establish and protect sessions.
- [RS-004 (CoSocket)](rs004-cosocket.md) is the part of the technology that helps transport the data using alternative methods like sneakernets.
- [RS-016 (PoWebSocket)](rs016-powebsocket.md) defines a protocol that connects applications to the Relaynet network.
- [RS-007 (PoHTTP)](rs007-pohttp.md) defines a protocol that connects Relaynet to the Internet.
- [RS-018 (Cryptographic Algorithms)](rs018-algorithms.md) defines the cryptographic algorithms that can be used in Relaynet.
- [RS-014 (Ping)](rs014-ping.md) defines a trivial service to test end-to-end the implementation and integration of Relaynet components.

On the other hand, [RS-012 (Service Integration Scale)](rs012-service-integration.md) categorizes the degrees to which Relaynet can be integrated in a service. This can be useful to understand the vision of the project and how future applications could be built on top of Relaynet.

The following documents are placeholders for future extensions:

- [RS-017 (Adaptive Relay)](rs017-adaptive-relay.md) will keep latencies low when the underlying network (e.g., the Internet) is available.
- [RS-010](rs010-pdc-browser.md) will define a JavaScript interface that browsers or browser extensions can expose to make it easier and safer for client-side apps to send and receive parcels.
- [RS-011 (AsyncRPC)](rs011-asyncrpc.md) will define a service that encapsulates RPCs in Relaynet messages. Only meant as a steppingstone until the actual service supports Relaynet.
- [RS-013 (Message Broadcast)](rs013-pubsub.md) will add support for the [Publish-Subscribe pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html).
