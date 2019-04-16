---
nav_exclude: true
permalink: /RS-009
---
# PogRPC: Parcel Delivery over gRPC

- Id: RS-009.
- Status: Placeholder.
- Type: Implementation.
- Proof of concept: https://github.com/relaynet/poc/tree/master/PogRPC

## Abstract

This document describes PogRPC, a [parcel delivery binding](rs000-core.md#parcel-delivery-binding) for external Parcel Delivery Connections (PDC) based on [gRPC](https://grpc.io/).

## Open Questions

- Should this be deprecated in favour of [PoHTTP](rs007-pohttp.md)? PogRPC is only useful when the client can/should deliver parcels in batch to the server, but implementing that functionality in the client will be complicated and the benefits are likely to be minimal.
