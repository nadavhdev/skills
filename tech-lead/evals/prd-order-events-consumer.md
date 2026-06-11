# PRD: Real-time Order Status Projections

**Author:** Marcus Webb (Platform PM)
**Date:** 2026-06-02
**Status:** Eng review

## Background

We're an e-commerce platform (~50K orders/day, 5K/day on peaks during
flash sales — we hit 18K/day on Black Friday last year). Our order
service publishes domain events to Kafka topic `orders.events.v1` whenever
an order state changes (created, payment_authorized, payment_captured,
fulfilled, partially_refunded, refunded, cancelled, etc. — ~14 event
types). The topic is partitioned by `order_id` with 24 partitions.

Currently, the customer-facing app polls the order service every 5
seconds for status updates. This is expensive (~120 RPS just from polling)
and laggy. The customer-app team wants a sub-second order-status feed.

We want to build a new "order projections" service that consumes the
events topic and maintains a denormalized read model in a low-latency
store. The customer app will read from this store instead of polling the
order service.

## Goals

- Maintain a per-order denormalized state record with the latest known
  status, line items, payment summary, fulfillment summary, refund summary.
- Read latency p99 < 50ms.
- End-to-end "event published → projection updated" p95 < 2s.
- Handle full back-fill (re-process all events from topic start) for
  initial seed and for recovery.

## Non-goals

- Replacing the order service as source of truth.
- Historical analytics (a separate batch pipeline handles that).
- Cross-order aggregations (per-customer order list is a separate
  projection that can be built later — call it out in the design).

## Functional requirements

- Consume `orders.events.v1`.
- Apply event → state transition. Bad transitions (e.g. "fulfilled" on
  a never-created order) are logged and DLQ'd, do not stop the consumer.
- Expose a read endpoint `GET /orders/{order_id}/projection`.
- Support a manual replay tool for ops to rebuild a single order's
  projection.

## Constraints

- We're on GCP (events are in Pub/Sub-mirrored Kafka via Confluent). The
  rest of the platform is on GKE.
- Team is split Java (Spring Boot) and Kotlin. Order service is Kotlin.
- Pub/Sub-mirrored Kafka has at-least-once delivery; we have to deal
  with duplicates.
- Order events sometimes arrive out of order across partitions during
  broker rebalancing (we've seen this).
- We need this live by end of Q3 (Sept 30) for the holiday season ramp.

## Notes from initial discussion with engineering

- Schema is Avro on Confluent Schema Registry, BACKWARDS compatibility
  enforced.
- The customer app currently calls our internal API gateway; the new
  endpoint should sit behind the same gateway for auth consistency.
- Storage choice is open — eng has floated DynamoDB, Postgres, and
  Redis. PM doesn't have an opinion.
