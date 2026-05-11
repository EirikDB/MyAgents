# dev-security — accumulated learnings

## Auth endpoint audit for self-hosted single-user .NET notes app (2026-05-11)

**Project shape:** single-user self-hosted notes app. Browser frontend (plain JS, single file, already accepts a Claude API key in `localStorage['notes-claude-key']` as an explicit prototype anti-pattern). .NET REST API replaces Supabase gotrue. Two endpoints under audit: `POST /auth/login`, `POST /auth/refresh`.

### Threat model in one paragraph

Self-hosted ≠ private. Anything published on a domain gets scanned by Shodan/masscan within hours. Single-user means the attacker knows the username (one valid account exists, often the operator's email). That collapses credential stuffing into pure brute-force against one known account — the easiest attack class to run, the hardest to detect from a single failed-login signal. The cost of a breach is total: all notes, the Claude API key, and the auth credentials are accessible from one compromised session.

### Highest-risk gap identified

**Tokens in `localStorage` while the app renders Claude-controlled strings.** The notes-app already invokes Claude and renders its output in the Ask & apply feature. Every Claude-controlled field is currently `escHtml`-escaped, but it is a single missed `textContent` switch from XSS-to-token-exfil. With JWT + refresh token + Claude API key all in `localStorage`, one XSS = total compromise. The team is migrating from Supabase, which by default puts the session JWT in `localStorage` — the path of least resistance is to carry that pattern over. Don't.

### Decisions to enforce

1. **httpOnly + SameSite=Strict + Secure cookies for both access and refresh tokens.** Same-origin self-host makes SameSite=Strict trivial. Eliminates XSS exfil; reduces CSRF to a non-issue for state-changing endpoints when combined with origin check on `/auth/refresh`.
2. **Refresh tokens are opaque 256-bit randoms, hashed (sha256) in DB, rotated on every use, with reuse-detection that nukes the family.**
3. **ASP.NET `Microsoft.AspNetCore.RateLimiting` fixed-window: 5 attempts / 15 min / IP on `/auth/login`; 30 attempts / min / IP on `/auth/refresh`.** Plus a global concurrency limit on `/auth/login` to bound argon2 CPU burn.
4. **Argon2id for password hashing.** `Konscious.Security.Cryptography.Argon2` with memoryCost 64MB, iterations 3, parallelism 1 — OWASP 2025 baseline.
5. **JWT secret from env var or DPAPI-protected file, never appsettings.json checked into git.** Asymmetric (RS256) preferred so the public key can be shared without minting power.

### Patterns to watch in this codebase

- Team consistently puts user-controlled and Claude-controlled data in `localStorage`. Auth tokens must break this pattern, not extend it.
- Spec-literal compliance: dev-skeptic already flagged this. The auth spec must explicitly forbid `localStorage` storage, not merely "recommend" cookies.
- The Ask & apply feature `escHtml`s Claude output. One missed call site = XSS. Periodically grep for `innerHTML` near rendered Claude data.

### Accepted risks the team is carrying

- API key in `localStorage` for the prototype Ask & apply feature. Documented as prototype-only. Migrate to Go binary proxy before any multi-user phase.

### Dependency CVE debt to track on next pass

- `Microsoft.AspNetCore.Authentication.JwtBearer` — track CVE feed. Historically had timing-attack and validation-bypass issues.
- `Konscious.Security.Cryptography.Argon2` — single-maintainer package; if abandoned, switch to `Isopoh.Cryptography.Argon2`.
- Any cookie auth lib pulled in — verify it sets HttpOnly/Secure/SameSite by default and check defaults each upgrade.
