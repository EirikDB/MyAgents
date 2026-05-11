# dev-qa — accumulated learnings

_Per-agent notes. Written at shutdown. Append or revise; don't bloat._

## Coverage patterns seen in this codebase

### Auth endpoints (POST /auth/login, POST /auth/refresh) — fictional design exercise, 2026-05-11

The .NET notes-app auth flow surfaced a reusable P0 checklist for any token-issuing endpoint pair. Capture it here so future auth work doesn't re-derive it:

**Login P0s that are always overlooked:**
- Non-existent-user vs. wrong-password **body equality** (enumeration defense — easy to get wrong because the natural code path returns different exceptions).
- Non-existent-user vs. wrong-password **timing parity** (requires a dummy BCrypt.Verify on the not-found branch; without it, the timing oracle is wide open).
- Field-level 400 vs. 401 distinction — a blank password is a validation error, not an auth error. Confusing the two leaks "user exists" information.

**Refresh P0s that are always overlooked:**
- Token-type confusion (access token presented as refresh token).
- Rotation replay (single-use contract).
- **Family revocation on reuse detection** — this is the security-critical one. Replaying an old refresh token must invalidate the entire descendant chain, forcing the legitimate user to re-auth. Without this, a stolen refresh token grants attackers indefinite access as long as they outrace the legit user.
- **Concurrent refresh race** — N parallel requests with the same refresh token must serialize at the data layer so exactly one wins. Only works with a row-lock or `UPDATE ... WHERE used_at IS NULL` pattern inside a transaction.

### Standing escalation list for auth designs

Before writing P0 auth tests, four pre-reqs must be settled by the team or the tests are speculative:

1. Rotation strategy (rotate-on-every-refresh + family-revocation is the default I push for).
2. DB-level serialization mechanism for refresh-token consumption.
3. Dummy-hash on the not-found branch of login.
4. Refresh-token storage shape (opaque + DB row vs. pure JWT with revocation table).

If any of these are unstated, flag in the release-gate verdict — do not paper over with a speculative test that will need to be rewritten.

## Framework defaults for .NET in this org

- **xUnit + FluentAssertions + `WebApplicationFactory<T>`** for integration.
- Default to integration tests for auth flows (wiring-bug magnet); carve out unit tests only for logic that's hard to drive end-to-end (clock-based expiry, family-revocation invariants).
- Always inject `IClock` so expiry tests don't have to `Thread.Sleep`.

## Recurring untested patterns to watch for

- **Happy-path-only auth suites.** A login test that only asserts the 200 path is a coverage trap — failure paths are where the real bugs live.
- **Tests that assert HTTP 401 but never inspect the body.** Username enumeration via response-body diffs slips through every time.
- **"Mock the entire token service" anti-pattern** — over-mocked rotation tests pass while the real DB serialization is broken. The R8 (concurrency) test only has meaning at the integration level.

## Escalations resolved

_(none yet — first session)_

## Accepted coverage gaps

_(none yet — first session)_
