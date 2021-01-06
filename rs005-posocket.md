---
nav_exclude: true
permalink: /RS-005
---
# PoSocket: Parcel Delivery over TPC/Unix Sockets

- Id: RS-005.
- Status: Placeholder.
- Type: Implementation.

## Abstract

This document will describe _PoSocket_, a binding for Gateway Synchronization Connections on top of TCP or Unix sockets. As a purpose-built [Application Layer](https://en.wikipedia.org/wiki/Application_layer) protocol, this is the most efficient binding to deliver parcels.

## Open Questions

- Is this binding really worth having? [PoWeb](rs016-poweb.md) can achieve exactly the same, although not very efficiently because implementations typically load the whole message in memory and the protocol adds some overhead (especially due to the lack of HTTP/2 support). Would these theoretical performance gains be noticeable? And if so, do they justify having a purpose-built Layer 7 protocol?
