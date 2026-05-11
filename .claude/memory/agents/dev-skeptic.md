# dev-skeptic — accumulated learnings

## notes-app "Ask & apply" prototype review (2026-05-11)

**Project shape:** single-file browser prototype, localStorage-backed, Anthropic API called directly from browser with key in `localStorage['notes-claude-key']`. Security-banner anti-pattern is explicitly accepted as prototype-only.

### Failure modes that landed

- **`max_tokens: 1024` silent truncation.** When Claude has many proposals to return, the response truncates mid-JSON, `JSON.parse` fails, user sees the generic "Claude couldn't answer" — no signal that the actual problem is response-size. Recurring class: bounded output limits that silently corrupt JSON in LLM-output paths.
- **Stale-response race in promise chains.** `askState` is a module-level singleton. Firing two `? queries` in rapid succession lets the first response overwrite the second's render state. JS single-threaded ≠ race-free when there are awaited fetches and shared mutable state. Solution pattern: monotonic `requestId`, drop responses where current id differs.
- **HTTP error categories collapsed into one bucket.** 401 (bad key), 429 (rate limited), 500 (server) all surfaced as "Claude couldn't answer right now." 401 specifically traps users — they have no path back to Settings to update the key. Branch on `resp.status` even in prototypes.
- **Corpus-size unbounded.** `compactCorpus` packs every note without estimation. Power user with 5K notes will hit Claude's input limit and get the generic HTTP error. Even prototypes need a soft cap with an explicit message.

### Failure modes that were rebutted / not present

- **XSS via Claude-controlled strings.** Implementation correctly escHtml's every Claude-controlled field (verb, excerpt, rationale). Query rendered via textContent. No P0 found here.
- **API key leakage.** Key only in `x-api-key` header. No console.log. Bare catch swallows fetch errors. Password input. No URL params. No P0.
- **Apply-all clobber/race.** Sequential read-modify-write through localStorage. JS single-threaded, no overlapping awaits. Each iteration sees the previous mutation. Not a clobber. Don't reflexively flag "multiple writes" as a race — examine the actual scheduling.

### Patterns the team drifted into

- **Generic error catches that erase actionable info.** Reflexively bucketing all HTTP errors into one user-facing message is a UX-and-debuggability bug, not just a UX nit. Especially severe for auth errors where the user owns the credential.
- **Spec-literal compliance over spec-intent compliance.** Implementation matched DESIGN.md error string verbatim ("Claude couldn't answer right now.") for all error categories, including the ones where DESIGN didn't actually specify a category-collapsed treatment.

### My own rating-axis mistake (caught by team-lead in pressure-test)

- I initially rated "missing invocation/accept-rate instrumentation" as P2 optional hardening. Team-lead caught a self-contradiction: I'd also said the binary validation questions cannot be answered without those counts, and that a wrong kill-decision is worse than any bug. The two statements forced P1.
- **Lesson:** when scoring severity in a validation-phase prototype, the axis is not "is the code broken" but "does the build let the team answer the question it was built to answer." Missing instrumentation that makes the validation gate unanswerable is P1 even when the code runs fine. Apply this lens to future Phase-N.5 prototype reviews — every feature exists to answer a binary question, and anything that prevents the question from being answered is shipping-blocker severity.

### Project-specific cliffs to watch on Phase 2 (Go binary)

- Localhost CSRF (already in DESIGN top-3) — verify Origin allowlist works against null Origin from file:// pages.
- Prompt cache busting on every note edit — 30s debounce is the planned mitigation; load-test this with real edit cadence.
- Migration of API key from browser localStorage to OS keyring on cutover — users will have keys in both places transiently; document the cleanup.

## Supabase gotrue → .NET /auth/login + /auth/refresh review (2026-05-11)

**Plan shape:** team-lead asked me to review a plan written up as "POST /auth/login + POST /auth/refresh on .NET REST API, decommissioning Supabase gotrue." Self-hosted single-user notes app.

### Failure modes that landed (anchor for future Supabase-auth migrations)

- **"Two endpoints" framing is the trap.** Production-grade auth is ~7 endpoints (login, refresh, logout, password-reset-request, password-reset-confirm, change-password, revoke-all-sessions). When someone scopes the migration as "just login + refresh," that's the signal — password reset, revocation, rate limiting, replay detection have all been silently elided. Push them to enumerate the missing six before accepting the scope.
- **"Users will just reset" needs the reset endpoint actually in the plan.** This is the password-hash-portability cliff playing out concretely. Hand-waving "they'll reset" without shipping /auth/password-reset = the user is bricked on day one with no recovery. For self-host single-user, a CLI `reset-password <email>` is an acceptable substitute, but it must exist and be in the runbook. Hash format: Supabase gotrue uses bcrypt cost-10; .NET BCrypt.Net-Next is wire-compatible, ASP.NET Identity (PBKDF2) is NOT. Dual-verifier with upgrade-on-success eliminates the reset cliff entirely when hashers can be made compatible.
- **supabase-js cached-session trust window.** The library renders `session.user` from cache before validating the token. On cutover, UI flashes "logged in" while every API call 401s. Migration runbook must include: wipe `sb-*-auth-token` localStorage keys on first load post-deploy, AND frontend version bump that removes `@supabase/supabase-js` entirely (not coexisting). Anchor this for future frontend teammates.
- **Auto-refresh-on-401 stampede.** Classic axios-interceptor pattern: any 401 triggers /auth/refresh, then retries. When /auth/refresh itself is failing, every queued request retries on the same dead refresh token. Needs single-flight refresh queue (one in-flight refresh; all other 401s await its result). Flag this any time a frontend teammate proposes "interceptor refresh."
- **Refresh-token rotation + family-based replay detection.** gotrue does this; replacements that skip it are strictly less secure than what's being replaced. Required: each refresh issues NEW refresh token, invalidates old; if old presented again → invalidate entire family. Requires `refresh_tokens` server-side table with `(user_id, family_id)` index. ORM-default index won't survive concurrent refresh from multiple tabs/devices — single-user app still produces concurrent refresh from desktop + phone.

### Supabase-specific migration cliffs (extending the inherited list)

- **`auth.users` has many gotrue-internal columns** (encrypted_password, instance_id, aud, role, raw_app_meta_data, raw_user_meta_data, confirmation_token, recovery_token, email_change_token_new, last_sign_in_at, banned_until). Plan must specify column-by-column which survive, which drop. Don't accept "we'll migrate the users table" as a complete answer.
- **JWT secret rotation = total session invalidation.** gotrue HS256 signed with the Supabase project secret; new .NET secret = every active session dead. Combined with the supabase-js cache trust window, this produces a "UI looks logged in but every call 401s" UX bug.

### Patterns the team recurringly drifts into

- **Scope-elision via narrow endpoint count.** Two endpoints sounds simple; the simplicity is illusory because the missing endpoints are still required by the threat model. When backend teammates propose a "minimal" auth surface, ask what happens on forgotten password, revoked session, stolen refresh token. If there's no answer, the scope is wrong.
- **"Single-user self-host means we can skip X."** Recurring drift: rate limiting, audit log, ownership predicates, password reset all get cut with this justification. Single-user removes coordination concerns, NOT security concerns. Anyone on the LAN can hammer /auth/login. The app will be Phase 3 multi-user eventually; ownership predicate must be in place from day one or the privesc bug ships at Phase 3 cutover.
- **Big-bang cutover proposed despite trivial parallel-run cost.** Self-hosted single-user makes dual-stack parallel-run a one-toggle problem, yet plans are still written as "deploy → point frontend → decommission." Push for parallel-run with a verified-user-action gate ("user logged in to .NET ≥1 time AND read notes ≥1 time") instead of a deploy-timestamp gate.

### Calibration note

- Frame critique as "what's missing from the plan to get to a shippable state" rather than "the plan is bad." Team-lead and peers respond to actionable hardening lists better than to global verdicts. Verdict still goes at the bottom, but the body should be the punch list.
