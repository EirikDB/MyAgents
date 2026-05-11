# dev-backend-dotnet — accumulated learnings

_Append on each shutdown. Don't overwrite still-valid notes._

## Project: supabase-migration (notes-app)

Target: single-user self-host first, multi-user hosted later (Phase 3). Replacing Supabase (gotrue + PostgREST + Postgres) with ASP.NET Core REST API on standalone Postgres.

### Stack choices (settled 2026-05-11)

- **Minimal APIs**, not controllers. Endpoint surface is small (~5-20 routes); controllers + filters earn nothing here. Revisit only if model binding gets complex or filter pipelines grow.
- **JWT bearer (HS256)** via `Microsoft.AspNetCore.Authentication.JwtBearer`. 256-bit secret from env/user-secrets/Key Vault — never `appsettings.json`. Asymmetric keys deferred to Phase 3 (multi-user); for single-user self-host the secret never leaves the box.
- **BCrypt.Net-Next** for password hashing. **Not ASP.NET Core Identity** — Identity is overkill for one user, drags in EF Core schema you don't need, and obscures the auth flow. Direct ownership wins.
- **Refresh tokens = opaque 256-bit random strings**, hashed at rest with SHA-256, stored in EF Core. Not JWTs. Opaque means single-row revocation; self-describing refresh tokens are an anti-pattern.
- **Rotation with reuse detection / family revocation.** Each refresh consumes old token (RevokedAt + ReplacedByTokenId), issues new with same FamilyId. Replay of a revoked token kills the whole family. OWASP standard.
- **EF Core + Npgsql** as default; Dapper only for measured hot paths.
- **ProblemDetails (RFC 7807)** for all error responses. `AddProblemDetails()` + `UseExceptionHandler()`. No bespoke error envelope.

### Auth flow — settled token transport (2026-05-11)

**httpOnly + Secure + SameSite=Strict cookies, NOT JSON-body / localStorage.**

Decision driver: notes-app's Ask & apply feature renders Claude-controlled strings into the DOM. Even with strict escaping, that's a meaningfully larger XSS surface than a normal CRUD SPA. Tokens in localStorage lose to XSS the moment one escape slips; httpOnly cookies survive a compromised renderer.

- Access token (`at`): `Path=/api`, `Max-Age=900` (15 min). **Requires API route convention: everything API under `/api/*`.**
- Refresh token (`rt`): `Path=/auth`, `Max-Age=2592000` (30 days). Only sent to `/auth/refresh` and `/auth/logout` — never to `/api/*`. Compromised API endpoint can't see it.
- `SameSite=Strict` handles CSRF for same-origin SPA. `Secure` mandatory in production.
- **CORS implication for dev:** frontend dev server must proxy to API origin (not run cross-origin), or `SameSite=Strict` won't send the cookie.

JWT middleware reads cookie via `JwtBearerEvents.OnMessageReceived`, with `Authorization: Bearer` fallback so future native clients work without server changes:

```csharp
opt.Events = new JwtBearerEvents
{
    OnMessageReceived = ctx =>
    {
        if (ctx.Request.Cookies.TryGetValue("at", out var token))
            ctx.Token = token;
        return Task.CompletedTask;
    }
};
```

`/auth/logout` is **mandatory** with cookies (was optional with localStorage). Server must clear cookies + revoke refresh token row.

### Token response body (settled 2026-05-11)

Return both `expiresIn` (OAuth 2.0 RFC 6749 conformance) AND `expiresAt` (ISO 8601 UTC) — frontend picks. Note for future debates:
- Tab-sleep drift is a frontend implementation bug (recomputing from `receivedAt + expiresIn` on each access), not a contract bug.
- `expiresAt` is *more* vulnerable to client/server clock skew than `expiresIn` — laptop clock 2 min fast → absolute timestamp appears expired early.
- Don't waste cycles re-litigating; just return both.

### Endpoint conventions

- POST `/auth/login` — body `{username, password}`; 200 sets cookies + returns metadata; 401 ProblemDetails on miss. **Always run BCrypt even when user not found** — defeats username enumeration via timing.
- POST `/auth/refresh` — empty body, reads `rt` cookie; 200 sets new cookies (rotation); 401 on invalid/expired. On reuse-after-revocation, kill family server-side, return plain 401 (don't leak the difference).
- POST `/auth/logout` — reads `rt`, revokes row, returns cleared cookies.
- API routes under `/api/*` and `.RequireAuthorization()`.

### Middleware order (people get this wrong)

```
UseExceptionHandler → UseStatusCodePages → UseAuthentication → UseAuthorization → endpoints
```

ProblemDetails before auth, auth before authz, both before endpoints. `ClockSkew = TimeSpan.FromSeconds(30)` — the default 5 minutes is too loose.

### Top recurring backend risks for this project

1. **JWT signing secret leakage** — env-only, rotation via `kid` header + dual `IssuerSigningKeys` overlap window.
2. **Refresh token theft (XSS or storage)** — mitigated by cookie strategy + rotation with reuse detection. Don't IP-bind on mobile (legit users get locked out).
3. **Login brute force on public self-host** — `AddRateLimiter` with independent IP and username buckets (so attacker can't lock legit user out by spamming their username).

### Migration sequencing (general)

- Build auth endpoints first; that unblocks everything else.
- Keep Supabase IDs stable across cutover (UUIDs) so FKs survive.
- Password hashes can't migrate from gotrue (bcrypt with its own pepper) — force reset or transparent re-auth on first login.

### Peer interaction patterns observed

- **dev-frontend-react** will ask for `expiresAt` (concede, return both — see above).
- **dev-security** will push httpOnly cookies hard whenever the frontend renders LLM-controlled content (concede; the XSS amplifier argument is real).
- **dev-database-postgres** owns the `RefreshToken` table schema; coordinate on indexes: `(TokenHash) UNIQUE`, `(UserId, RevokedAt)` for cleanup, `(FamilyId)` for revocation queries.
- **dev-skeptic** will hit on auth-migration cliffs (locked-out users, FK breakage, hash migration). Don't pre-harmonize but plan for transparent re-auth on first login.
