# PRD: Vendor Inventory Sync

**Author:** Priya Subramanian (Supply Chain PM)
**Date:** 2026-05-30
**Status:** Awaiting eng design

## Background

We sell ~140K SKUs across 12 vendors. Each vendor exposes their inventory
via different mechanisms: 8 of them have a REST API we can poll, 3 of them
drop a CSV to an SFTP nightly, and 1 of them sends us a daily email
attachment (yes, really — that one is Vendor K, see ticket VENDOR-2341).

Currently inventory is updated manually by the ops team using a Google
Sheet, which is at least 24 hours stale and frequently wrong. We lose
~$80K/month in customer-facing "out of stock after order placed"
incidents according to last quarter's CX report.

We want an automated sync that pulls vendor inventory and updates our
product catalog so the storefront shows accurate availability.

## Goals

- For each of the 12 vendors, pull current inventory at a vendor-
  appropriate cadence.
- Update our internal product catalog (existing service, Postgres-backed,
  has a REST update API).
- Detect and alert on stale data (a vendor hasn't sent us new data in
  >24h means something is broken).
- Keep an audit log of every inventory change with source and timestamp.

## Non-goals

- Two-way sync (we don't push our sales numbers back to vendors).
- Real-time inventory (the underlying vendor data is at best hourly;
  we're aiming for "fresh enough" not "live").
- Replacing the ops team's Google Sheet workflow on day 1 — we'll run
  the new system in shadow mode for two weeks first.

## Functional requirements

- Per-vendor configuration: source type (REST / SFTP / email), endpoint
  / credentials, poll frequency, field mapping.
- Process each vendor's data, normalize to our SKU model.
- Diff against current catalog; only apply changes.
- Handle the messy cases: a vendor briefly returns empty / zero
  inventory; a vendor's API is down; a CSV is malformed; a SKU appears
  that we don't recognize.

## Constraints

- Engineering team is Python-shop (Django + Celery elsewhere in the
  org).
- We're on AWS (us-east-2). Catalog service is in the same VPC.
- 2 backend engineers can be allocated for ~6 weeks.
- The email-attachment vendor (K) is non-negotiable on their delivery
  mechanism — they're a small supplier, take it or leave it. They're
  ~3% of revenue.
- Vendors' APIs have varying rate limits (one of them, Vendor F,
  limits us to 60 requests/hour total).

## Open from PM side

- How do we surface a "this SKU was just discontinued" event to the
  storefront team? (They have their own pipeline.)
- What happens during the shadow-mode period if the new system disagrees
  with the Google Sheet — who arbitrates?
