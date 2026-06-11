# T03 — Code generator (8-char Crockford base32, retry-on-collision)

**Estimate:** 1 day
**Depends on:** T01
**TDD ref:** §4.3

## Outcome

A pure function / module that produces an 8-character Crockford base32
referral code, with a documented contract and unit tests covering the
collision-retry behavior.

## Scope

- Generator emits 8-character Crockford base32 (excludes `I`, `L`, `O`, `U`).
- A `generate(check_exists)` API where `check_exists(code) -> bool` is
  injected, so the DB lookup is the caller's concern.
- Bounded retry loop: max 5 attempts; raise a typed `CodeCollisionError`
  if exhausted (this is the "backstop" path called out in the TDD).
- Use a cryptographically strong RNG (`secrets`-equivalent in the chosen
  language), not a seeded PRNG.

## Out of scope

- Persisting the code (T04).
- Metric for collision rate (T09 will add a counter that wraps this).

## Acceptance criteria

- [ ] Unit test: 10k generated codes are all 8 chars, all in the Crockford
      alphabet, no excluded letters.
- [ ] Unit test: with a stub `check_exists` that returns `True` 4 times
      then `False`, generator succeeds on the 5th attempt.
- [ ] Unit test: with a stub `check_exists` that always returns `True`,
      generator raises `CodeCollisionError` after exactly 5 attempts.
- [ ] No more than 5 calls to `check_exists` per invocation.
