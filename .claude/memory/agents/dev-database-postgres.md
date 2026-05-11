# dev-database-postgres — accumulated notes

_Written at agent shutdown. Append or revise; don't truncate._

## Project-specific decisions

### supabase-migration: auth schema (Task #5, 2026-05-11)

- **Extensions required on target Postgres:** `pgcrypto` (for `gen_random_uuid()`) and `citext` (for case-insensitive email). Both are bundled with stock Postgres ≥ 13; no managed-service issues. Supabase-only extensions (`pgsodium`, `vault`, `supabase_vault`, `pgjwt`) are explicitly NOT used in the new schema.
- **`users` table lives in `public`, not `auth`.** The Supabase `auth` schema is abandoned at migration. `public.users.id` MUST equal Supabase `auth.users.id` (UUID v4) to preserve every FK from `notes.user_id`, etc. Migration uses explicit `INSERT ... SELECT id, ...` — never let DEFAULT fire on the migration insert.
- **Email column is `citext`** with a CHECK constraint that the value equals its own lowercase. Cheaper than `lower(email)` expression indexes; survives `UNIQUE`.
- **Password hash column accepts bcrypt format only** via regex CHECK `^\$2[aby]\$\d{2}\$.{53}$`. Bcrypt is 100% portable from Supabase gotrue → .NET BCrypt.Net-Next: same algorithm, cost embedded in hash, no rehash needed at migration time. Cost ≥ 10 enforced by regex; allows future upgrade-on-login without schema change.
- **`refresh_tokens` uses hashed-at-rest tokens (`bytea`, SHA-256 of opaque token).** Plus `used_at`, `revoked_at`, `family_id`, `parent_id` to support token rotation with replay detection. Cleanup retention ≥ 90 days — deleting used tokens kills replay-detection signal.
- **RLS dropped on all auth tables.** Replaced by Postgres role grants (`notes_app` role gets DML; `notes_readonly` role gets SELECT minus `password_hash`). App-layer enforces user-scoped queries via explicit `WHERE user_id = @currentUserId` — discipline requirement, no DB safety net.

## Patterns adopted

- **UUID v4 from `gen_random_uuid()` (pgcrypto), not `uuid_generate_v4()` (uuid-ossp).** pgcrypto is already in use for password hashing; one less extension to track.
- **Updated-at via trigger, not application code.** Single `set_updated_at()` function reused per table; reduces drift risk between code paths.
- **Partial indexes for hot paths.** `refresh_tokens (user_id, expires_at) WHERE used_at IS NULL AND revoked_at IS NULL` — the active-token lookup is the dominant query and warrants its own slimmer index.
- **CHECK constraints encode invariants the app could violate.** Email lowercase, bcrypt format, expires_at > issued_at. Cheap insurance against bad inserts from forgotten code paths.

## Migration-tooling choice

- **Not yet settled.** Candidates: Flyway (mature, SQL-first), dbmate (lightweight, SQL-first), EF Core migrations (canonical when .NET model leads). Pending backend's call on whether the .NET model or raw SQL is canonical.

## Open questions sent to peers

- **dev-backend-dotnet (Task #5):** Confirmed `revoked_at` column (not separate revocation table) is sufficient for refresh tokens. Open: do they want DB-level reuse-detection (trigger that revokes family on used-token re-presentation) or pure C# state machine? Affects whether `family_id` stays in schema.
