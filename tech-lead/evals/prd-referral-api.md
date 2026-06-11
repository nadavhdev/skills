# PRD: Referral Code API

**Author:** Sarah Chen (Growth PM)
**Date:** 2026-06-04
**Status:** Draft for engineering review

## Background

Our consumer mobile app (~2.1M MAU, growing 8% MoM) currently has no referral
program. The growth team has been waiting on this for a year. Marketing
has committed to a Q3 launch tied to our back-to-school campaign on
August 25.

We want users to be able to share a personal referral code with friends.
When the friend installs the app and completes onboarding, both users get
a $5 credit toward their next subscription month.

## Goals

- Users can generate / view their referral code in the app.
- Users can share the code via the OS share sheet (deep link).
- When a new user enters a code (or installs from a deep link), we
  attribute the install to the referrer.
- Both users receive credit once the new user completes onboarding
  (defined as: first paid subscription start).
- Marketing can pull a dashboard of code → installs → activations.

## Non-goals

- Multi-tier referral (referrer-of-referrer credit).
- B2B / enterprise referrals.
- Cross-platform attribution beyond what the mobile SDK gives us.

## User stories

1. As a user, I can tap "Invite friends" in settings and see my code.
2. As a user, I can share my code via the share sheet.
3. As a new user, when I install the app via a referral link, my account is
   automatically attributed.
4. As a new user, I can also enter a code manually during onboarding.
5. As marketing, I can see a daily dashboard of referrals by source.

## Functional requirements

- Each user gets one persistent referral code (6 alphanumeric chars, no
  ambiguous chars like 0/O/I/1).
- A code-redemption must be tracked even if the friend signs up days later.
- Credits are issued only when the new user reaches "activated" status.
- A user can't redeem their own code, and one new user can only be
  attributed to one referrer.

## Constraints

- Must integrate with our existing billing system (Stripe, with our own
  credit ledger in Postgres).
- Mobile team needs the API contract by July 1 to hit August 25.
- We use AWS (us-east-1 primary). Backend team is Python (FastAPI) and Go.
- Marketing dashboard can be Looker / Snowflake — data team owns this part.

## Open from PM side

- Should codes ever expire? (Currently assuming no.)
- What's the fraud story — what if someone creates 1000 fake accounts
  to farm credits? (Legal flagged this.)
