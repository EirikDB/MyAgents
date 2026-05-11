# dev-frontend-react Agent Memory

_Decisions, sequencing, and learnings carried across sessions._

## Project context

- **notes-app** is a single-file plain HTML/CSS/JS app (no React, no build step) at `notes-app/index.html`. Despite my role being "React + TypeScript," the user enforces single-file vanilla for prototypes (see preferences.md). All work goes into `index.html` — never split into modules.
- The Supabase→.NET migration target hasn't been engaged yet. When it lands, the OpenAPI-generated client + TanStack Query approach in my system prompt applies.

## notes-app — settled architecture (vanilla JS)

- **Storage:** `localStorage` under `notes-v1` (notes) and `notes-claude-key` (API key). No IndexedDB, no service worker.
- **Render model:** imperative `render()` re-derives three zones from `loadNotes()` on every state change. A 30s `setInterval` re-renders for relative-time updates. No virtual DOM, no diffing.
- **Event delegation** on `document` for `[data-action]` (note ops), `[data-ask-action]` (proposal ops), `[data-open]`/`[data-close]` (overlay toggles). New features should add their own `data-*` namespace and slot into the existing delegation block — don't add per-element listeners.
- **Overlay pattern:** any modal (`all-overlay`, `digest-overlay`, `settings-overlay`) uses `.overlay` + `.overlay-card` and toggles a single `.open` class. Esc dismisses all open overlays globally.

## Session 2026-05-11 — Phase 1.5 "Ask & apply" browser-only prototype

Implemented in `index.html`. Key decisions:

- **`?` prefix routes through `saveCapture()`**, not a separate keydown branch. The Save button click and Enter press both call `saveCapture()`, which inspects `value.startsWith('?')` and forwards to `runAskApply()`. One code path; mode switch is purely textual.
- **`updateCaptureMode()` is called on every input + after every programmatic textarea mutation** (Esc clear, post-save clear, post-ask-success clear). `value=''` does not fire an `input` event, so mode reset has to be explicit.
- **`askState` is a single module-level object** reassigned wholesale on each query (`{active, query, loading, error, empty, proposals[]}`). Per-proposal `accepted`/`rejected` flags mutate in place. Re-rendering is a single `renderAskZone()` call from each state transition — no observers.
- **JSON parsing has two fallbacks**: strip whole-string markdown fence, then if `JSON.parse` still fails, regex-extract the first `[...]` substring. Claude occasionally adds an intro sentence even when system-prompted not to.
- **Proposal cards display the local note excerpt, not Claude's restatement.** `_excerpt` is computed from `loadNotes()[noteId].body.slice(0, 60)` after the response arrives. Prevents Claude hallucinating note text into the UI.
- **Apply mutations reuse existing note functions** — `markDone` for archive; inline `tags` array merge for tag (deduped via `Set`, lowercased, stripped of `#` and non-word chars); inline `fireAt` assignment for schedule (with `^\d{4}-\d{2}-\d{2}$` regex guard before `new Date()`). Schedule clears `snoozedUntil` so an existing snooze doesn't mask the new fire time.
- **Capture box is cleared only on a clean (non-error) Claude response.** All error states keep the query in the box so the user can hit Enter again to retry. This came directly from the spec but it's the right pattern in general for any LLM-call UI.
- **`anthropic-dangerous-direct-browser-access: true` header is required** for browser CORS to api.anthropic.com. Phase 2 (Go proxy) removes it; the API-key-in-localStorage approach is explicitly prototype-only and gated behind a banner in the Settings drawer.

## Libraries — adopted / rejected (notes-app)

- **Adopted:** none. Vanilla everything, including the API call (native `fetch`).
- **Rejected for prototype:** React, TanStack Query, axios, any state library. Re-introduce only when the app graduates beyond single-file or the Supabase migration begins.

## Recurring frontend risks observed

1. **Programmatic value mutation desync** — when JS sets `textarea.value = ''` directly, `input`-bound state derivations (mode hints, button labels, autosize) don't fire. Always pair with an explicit recomputation call.
2. **Esc-handler stacking** — multiple Esc handlers (element-level and document-level) both run unless `stopPropagation`. Keep both, but make each idempotent.
3. **LLM JSON parsing brittleness** — system prompts asking for "JSON only, no prose" are honored ~95% of the time. Always handle code-fence wrap and prose-prefix cases.
4. **In-place mutation of state objects rendered from a list** — `p.accepted = true` works only because `renderAskZone()` re-reads `askState.proposals` on each call. If anything caches the proposals array, this breaks. Document the assumption.

## Session 2026-05-11 (round 2) — P1 hardening after dev-skeptic review

The first-cut implementation went through dev-skeptic and came back with seven required fixes. Pattern is worth remembering:

- **Skeptic-driven hardening is the right shape for any browser-LLM-client work.** Implementation → skeptic review → P1 patch round, all before validation users touch it. The first cut was functional but silently brittle in five distinct ways (stale fetches, missing timeout, max_tokens collision with JSON, no error specificity, no telemetry).

### Patterns adopted into the runAskApply skeleton

These are now my default skeleton for *any* browser-fronting LLM/REST async flow:

1. **Monotonic request ID + per-await guards.** Module-level `let requestId = 0;` plus capture `const myId = ++requestId;` at function entry. Store on shared state (`state.requestId = myId`). Before every state write that follows an `await`, check `if (state.requestId !== myId) return;`. Without this, rapid resubmits race; with it, every stale resolution silently bails.
2. **AbortController + timeout + cancel-on-dismiss.** `const ctrl = new AbortController(); const t = setTimeout(() => ctrl.abort(), 30_000);` pair, `signal: ctrl.signal` on the fetch, `clearTimeout(t)` in every exit path. Stash `state.abortController = ctrl` so the Esc/dismiss handler can abort the in-flight call. Distinguish `e.name === 'AbortError'` from generic fetch errors in the catch.
3. **HTTP status branching with distinct messages.** 401 → "key rejected" with link to settings; 429 → "rate limited, try again in a moment"; 5xx and other non-OK → generic. The skeptic correctly flagged that "Claude couldn't answer right now" obscures the diagnostic on a stale key — exactly the case the validation user is most likely to hit.
4. **Truncation detection on `stop_reason === 'max_tokens'`.** Bump max_tokens generously (4096 for Haiku, more for Sonnet), then check `data.stop_reason` after parse. If truncated and JSON still parsed, render a banner and use the partial result. Don't silently swallow.
5. **Corpus / payload size guard before the fetch fires.** Cheap pre-check on `corpus.length > LIMIT`. Spends nothing if the prompt is unreasonably big; surfaces a specific error instead of a slow timeout or unparseable response.
6. **Existence re-check before applying state mutations from a stale-by-construction result.** Any LLM proposal references entities that could have changed between query and accept. Top of every `applyX(p)` should re-read storage and short-circuit if the referent is gone. Surface the failure on the card, don't silently no-op.
7. **Local validation counters when a feature is gated on binary questions.** A Phase 1.5 stop rule that depends on "did users invoke this >2 times?" must ship with a counter or the gate is unenforceable. `bumpStats(field)` over a single JSON localStorage key is enough — also write `firstAt` and `lastAt` ISO timestamps for windowing; the spec usually omits these but you can't compute "in week one" without them.

### Tactical decisions made

- **Don't count an apply-failed-because-note-deleted accept as a reject in the validation counter.** The user took an accept action with intent; the failure is environmental. Counting it as a reject would corrupt the accept-rate signal that drives the Phase 2 go/no-go.
- **`'http'` error keyword narrowed semantics** during P1.1 — it now means "non-OK, not 401, not 429" rather than "any non-OK." Worth flagging in deviations when you change an existing enum.
- **State-shape changes need an updated comment on the state declaration** every time. Future-me reads the inline enum comment before reading the function bodies.

## Pending / future for notes-app

- Phase 2: Go binary proxy fronts the Claude API. Frontend changes: drop `x-api-key` and `anthropic-dangerous-direct-browser-access` headers; call a local `/api/ask` instead. Settings drawer goes away (key lives server-side). The rest of the flow — `runAskApply`, proposals, `applyProposal` mutations — should be unchanged. The seven hardening patterns above carry over wholesale; only the URL and headers change.
- Phase 1.5 binary questions to watch: >2 invocations/user in week one; >40% accepted-proposal rate. Now answerable from `JSON.parse(localStorage.getItem('notes-ask-stats'))` — `invocations`, `accepts`, `rejects`, `firstAt`, `lastAt`. If either gate fails at 4–6 weeks, kill the feature; don't invest in the Go binary.

## Session 2026-05-11 (round 3) — Supabase auth → typed fetch (Task #6, design exercise)

This was a fictional Supabase → .NET migration exercise on top of notes-app's vanilla-JS surface. Deliverable went to team-lead as a TypeScript contract plus a vanilla-JS drop-in. Key decisions worth carrying forward:

### Auth call-site inventory pattern

When migrating off Supabase auth, the seven canonical call sites are:
1. `supabase.auth.signInWithPassword` → `POST /auth/login`
2. `supabase.auth.getSession` → synchronous localStorage read at boot (don't round-trip)
3. `supabase.auth.onAuthStateChange` → local pub/sub via `window.dispatchEvent('auth:change')`, no server push
4. `supabase.auth.signOut` → clear locally first, then best-effort `POST /auth/logout`
5. gotrue silent refresh → manual `POST /auth/refresh` with single-flight guard, scheduled ~60s before expiry + reactive on 401
6. Bearer header injection → centralized `apiFetch()` wrapper, only place that knows about tokens
7. `supabase.auth.getUser` → decode JWT claims client-side or `GET /auth/me` (lazy, not eager)

Risk ordering: **refresh (5) > apiFetch wrapper (6) > login (1) > everything else**. Refresh failure breaks every authed call ~1h after install — silent and far from the original action. Wire it first, test it under concurrency, then build outward.

### `refreshInFlight` single-flight pattern (load-bearing)

```js
let refreshInFlight = null;
async function refreshAccessToken() {
  if (refreshInFlight) return refreshInFlight;   // share one request
  refreshInFlight = (async () => {
    try { /* fetch /auth/refresh, update store */ }
    finally { refreshInFlight = null; }          // ALWAYS clear, even on error
  })();
  return refreshInFlight;
}
```

Without this, two concurrent 401s spawn two refreshes; the second uses an already-rotated refresh token and the server kills the session. Symptom: random forced logouts under load. The `finally` clearing the in-flight ref is essential — a thrown error must not leave a dead Promise cached.

The `apiFetch` wrapper does both proactive (`expiresInMs() < 60_000`) and reactive (single retry on 401) refresh through the same single-flight gate. One refresh path; two triggers.

### `expiresAt` (ISO 8601 UTC absolute) over `expiresIn` (relative seconds)

Always push back on backends that issue `expiresIn`. The frontend must store absolute time, not "1800 seconds from when this response arrived" — relative storage drifts across tab sleep / device suspend / browser throttling. Compute relative from absolute at read time, not the other way around.

### localStorage vs httpOnly cookie — REVISED 2026-05-11 after dev-security pressure test

**Initial recommendation: localStorage. Reversed within the session after dev-security C1 finding. Final: httpOnly + Secure + SameSite=Strict, same-origin.**

The reversal matters more than the answer. Four arguments I made and why each failed:

1. **"Single-user self-host limits blast radius."** Rhetorical. The asset is the user's account; if it's stolen the attacker fully owns it. "One user" *is* the blast radius, doesn't shrink it.
2. **"Pattern consistency with `notes-claude-key`."** This argued against me, not for me. The existing pattern is *part of* the XSS attack surface — the same exfil reads both. "We're already doing it wrong" is not a reason to keep doing it.
3. **"No third-party JS deps."** I misidentified the threat. The XSS vector isn't supply chain — it's **Claude-controlled strings rendered into the DOM via the Ask & apply feature**. My own memory file flagged this with "note bodies are markdown-rendered through a sanitizer (or should be; verify)." That "verify" was the hole. Ask & apply adds a second.
4. **"httpOnly cookies need CORS gymnastics."** Cited the wrong deployment shape. Self-host = same-origin. .NET serves `index.html` via `MapFallbackToFile`; cookies attach automatically; no CORS, no credentials gymnastics, SameSite=Strict eats most CSRF risk for free. The "hard cookie case" applies to cross-origin SPAs, not to self-host.

**Lesson for future migrations:** when a peer (especially dev-security) names a specific code path that breaks the recommendation (here: the Ask & apply LLM-string rendering), don't argue from generalities. Either name the specific defense that closes the specific path, or concede. "We have a small attack surface" is not a defense when the peer just pointed at a hole in that surface.

### Revised wire shape (httpOnly cookie target)

- .NET API serves `index.html` same-origin via `MapFallbackToFile`. No CORS for the auth flow.
- `Set-Cookie: auth_access=<jwt>; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=900` (15 min)
- `Set-Cookie: auth_refresh=<opaque>; HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh; Max-Age=2592000` — **scope refresh cookie to `/auth/refresh` only** so it's never sent on data requests.
- Frontend never touches tokens. `apiFetch` uses `credentials: 'same-origin'`, no `Authorization` header path.
- Refresh: `POST /auth/refresh` reads cookie, rotates both, returns 204. Single-flight Promise still required for concurrency.
- Logout: `Set-Cookie: ...; Max-Age=0` on both + server-side RT revocation.
- Boot: optimistic `GET /auth/me` once; 200 = authed, 401 = login. No localStorage math, no `expiresAt` tracking.
- Cross-tab sync: `BroadcastChannel('auth')` (the `storage` event no longer fires — cookies aren't observable from JS).
- CSRF: SameSite=Strict + same-origin closes standard vectors. Require `Content-Type: application/json` on state-changing endpoints to force a preflight on cross-origin attempts. No CSRF token machinery for self-host.

### What httpOnly does *not* fix

Moving tokens to httpOnly closes **session theft**. It does not close:
- XSS-driven UI manipulation
- Action replay (attacker triggers state-changing requests from the user's session while the XSS payload runs)
- Theft of note content (not the same as session theft, still a privacy breach)
- Exfil of any non-httpOnly secrets remaining in localStorage (`notes-claude-key` still vulnerable)

Defense-in-depth additions to push into dev-security's audit scope:
- All Claude-generated strings via `textContent`, never `innerHTML`.
- If markdown rendering of LLM output is needed, use a pinned sanitizer (DOMPurify) with HTML allowlist.
- `Content-Security-Policy: default-src 'self'; script-src 'self'; ...` header from the .NET app — kills inline-script payloads in self-host.
- `notes-claude-key` should move to the .NET backend (proxy the Claude call) in the same migration window as the auth cookie change. Don't leave the localStorage-for-secrets pattern half-removed.

### When localStorage *would* be defensible

Genuinely zero LLM/user-content rendering, a real CSP enforced, refresh-token rotation, short access-token TTL, no third-party JS, and a same-origin XHR-only deployment — *and* the team has accepted that one XSS bug = full session theft as a known risk. The bar is high enough that "just use httpOnly" is the right default and the burden of proof sits on whoever wants localStorage.

### Boot sequence for "am I still logged in after refresh?"

1. Synchronous `authStore.get()` — single source of truth.
2. If `expiresAt > now` → render authed UI immediately (no round-trip).
3. If `expiresAt < now + 60s` and RT present → background refresh; first `apiFetch` awaits via single-flight.
4. If `expiresAt < now` and refresh fails → `clear()`, dispatch `auth:expired`, render login.
5. **Don't call `/auth/me` on every boot.** Every authed call validates server-side; eager validation just adds a synchronous round-trip with no information gain.
6. Cross-tab logout via `window.addEventListener('storage')` — free with localStorage. Costs ~3 lines of code.

### Recurring risks added to the list

5. **Refresh-token race under concurrency** — fix with single-flight Promise. Untested before launch; first time you see it is in production logs.
6. **Clock skew vs proactive refresh** — pad the proactive window generously (60s, not 5s), trust the server's `expiresAt`, let the reactive 401-retry path catch the rest. Don't compute relative-time deadlines on the client.
7. **"Secrets in localStorage" pattern bleeding between API key and auth token** — they store the same way but have different blast radii. Update the security banner copy to name auth tokens explicitly when this lands.

### What the .NET backend should ship for the frontend to work cleanly

- `accessToken` (JWT, ~15 min) + `refreshToken` (opaque, ~30 day) + `expiresAt` (ISO 8601 UTC) + `user: {id, email}` on login response.
- Refresh response: same shape, rotated refresh token, old RT revoked server-side.
- `POST /auth/logout` accepts `{refreshToken}` and invalidates it (idempotent, returns 204 either way).
- 401 on bad/expired access token, 401 on bad/expired refresh token (distinguish via response body if needed, but status alone is enough).
- Optional: `GET /auth/me` for lazy claims refresh; not required if claims are encoded in the JWT and don't change.

Push back on: `expiresIn` (relative), envelope-wrapped responses (`{data: {...}}`), camelCase/snake_case mixing, refresh tokens that don't rotate.
