# dev-researcher — accumulated learnings

_Per-session notes on the library/pattern landscape this project has explored. Read at session start; append at shutdown._

## Project shape (as of 2026-05-11)

- **supabase-migration** — single-user self-hosted .NET 8/9 REST API replacing Supabase (gotrue + Postgres + Storage + Realtime). Browser frontend (React + TS). No mobile clients in scope. Single Postgres instance, single deploy. "Phase 3 multi-user hosted" is deferred.
- Implication: many "scale" defaults (stateless tokens, distributed caches, eventual-consistency dance) are noise for this app. Recommend the simpler-with-state pattern unless there's a non-scale reason to go stateless.

## Settled recommendations

### Auth — refresh token pattern (2026-05-11, Task #7)
- **Recommended:** short-lived JWT access (~15 min) + **rotating opaque refresh token, DB-backed, SHA-256-hashed at rest, with reuse-detection chain (`replaced_by` self-fk)**.
- **Rejected:** stateless JWT refresh tokens. The only win (no DB hit at scale) is moot for single-user self-host, and you lose instant logout + reuse-detection.
- **Rejected:** JWT refresh + denylist. Pays for state without getting reuse-detection or clean revocation.
- **Transport:** refresh token in `HttpOnly; Secure; SameSite=Strict` cookie scoped to `/auth/refresh`. Access token in-memory `Authorization: Bearer`. No refresh-token-in-localStorage.
- **JWT signing:** prefer EdDSA (Ed25519) or ES256 over HS256. Asymmetric is cheap and key rotation is cleaner.
- **Why this fits the project:** mirrors gotrue's own pattern (Supabase Auth `refresh_tokens` table) → minimum behavioral drift for the React cutover.
- **Key references:** RFC 9700 (Jan 2025), OWASP OAuth 2.0 Cheat Sheet, draft-ietf-oauth-v2-1.
- **Open to revisit:** if non-browser clients (CLI, mobile, third-party) enter scope, DPoP-sender-constrained tokens deserve a fresh look.

## Patterns to default to for this project

- **State is cheap, statelessness is not free.** For single-user self-host, prefer the simpler state-bearing pattern. The "scale" justification for statelessness doesn't apply here.
- **Like-for-like replacement of Supabase primitives reduces cutover risk.** When in doubt, mirror what gotrue / PostgREST / Realtime did, then improve incrementally after migration.
- **OWASP / IETF BCP first, vendor blogs second.** Auth0 and Okta blogs are useful but trail the RFCs by months. RFC 9700 (Jan 2025) is the current ground truth for OAuth security BCP.

## Areas not yet researched (flag if asked)

- Storage replacement for Supabase Storage (object store choice, signed URL strategy).
- Realtime replacement (SignalR vs raw WebSockets vs SSE vs polling).
- Password hashing parameters (argon2id params for single-user are different from a multi-tenant SaaS).
- Email delivery for the notes-app email loop — providers and DKIM/SPF strategy.

## Don't re-research

- Stateless JWT refresh vs DB-backed rotating refresh — settled above. Need new evidence (e.g., scope change to multi-region) to revisit.
- "Should we use OAuth 2.1?" — yes, for the patterns. We are not a third-party authorization server, so the full grant-flow surface is overkill; we cherry-pick the security BCPs (PKCE-equivalence not relevant for first-party password login; rotation + reuse detection are).
