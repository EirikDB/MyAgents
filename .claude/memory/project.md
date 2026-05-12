# Project Context

_Maintained by team sessions. Read at session start; update after synthesis._

## Active projects

- **notes-app** — single-page HTML/CSS/JS note-taking webapp. Phase 1 MVP built: capture box with inline `#tag` / `@time` syntax, three zones (Due Now / Coming Up / Recent), localStorage persistence, digest preview. Lives at `notes-app/index.html`.
- **agent-team2** — scaffold for running multi-role Claude Code teams under one `eiriks-team` with three sub-teams: design (ux/architect/skeptic), GDPR (dpo/privacy-engineer/incident-responder/skeptic), and development (backend-dotnet/database-postgres/frontend-react/researcher/skeptic).
- **supabase-migration** *(forthcoming, codebase TBD)* — a separate app currently on Supabase (Postgres + gotrue + Storage + Realtime + PostgREST) being migrated to a .NET REST API backed by standalone Postgres. The dev sub-team is the primary tool for this. Codebase location and specifics to be added on first `/dev-team` invocation.

## Tech stack & constraints

- **notes-app**: plain HTML/CSS/JS, no framework, no build step, single file, localStorage. No backend yet. Email delivery deferred to Phase 2.
- **agent-team2**: Claude Code experimental agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). Agent definitions in `.claude/agents/`. Slash command in `.claude/commands/`.

## Current goals

- Evolve the agent team scaffold so teams accumulate memory across sessions.
- Phase 1: ship the email loop for notes-app; validate two binary questions (second capture within 7 days, done-click rate >30%). **The dyreid.no visual redesign is a companion to this PR — it ships with email, not before.**
- Phase 1.5: browser-only "Ask & apply" prototype — **IMPLEMENTED (2026-05-11)**. Feature is in `notes-app/index.html`. All 7 P1 fixes applied. Ready to ship alongside the email loop PR. Validation counters in `localStorage['notes-ask-stats']` (`invocations`, `accepts`, `rejects`, `firstAt`, `lastAt`).
- Phase 2: Go binary proxy (API key server-side), streaming SSE, prompt caching, full "Ask & apply" production design. Gate: Phase 1.5 binary questions must come back green (>2 invocations/user in week 1, >40% accept rate).
- Phase 3: multi-user hosted (deferred until single-user self-host is proven).

## notes-app/index.html — Ask & apply feature (Phase 1.5, added 2026-05-11)

- `?` prefix in capture box triggers query mode (footer hint swaps, button becomes "Ask ↵")
- Settings drawer: API key field (password input), security banner, stored in `localStorage['notes-claude-key']`
- Ask zone renders above Due Now; dismissed on Esc or non-`?` save
- Corpus format: `[YYYY-MM-DD  open|done]  id=<id>  <body>  #tags` (id column added so Claude can reference noteId)
- Model: `claude-haiku-4-5-20251001`, max_tokens: 4096, 30s AbortController timeout
- Proposals: accept (mutates localStorage) / reject per card, plus "apply all"
- Error states: no-key, auth (401), rate-limit (429), http, corpus-too-large, timeout, parse, empty
- Validation: `bumpAskStats()` tracks invocations/accepts/rejects + firstAt/lastAt timestamps


## design workflow: I would like to make a seperate folder where I build a satic — 2026-05-12

- **oslo-weather** *(new, 2026-05-12)* — single-file static page at `oslo-weather/index.html` showing Oslo daily forecast. Data source: `api.met.no/weatherapi/locationforecast/2.0/compact?lat=59.9139&lon=10.7522` (MET Norway, the org behind yr.no — JSON, CORS-open, NLOD/CC-BY 4.0). Requires `User-Agent` header identifying the app. No build step, no dependencies, no backend, no third-party proxy. localStorage cache (30-min TTL). Daily strip only (today + 7 days), Norwegian copy, Unicode emoji from MET `symbol_code`, three-tier color encoding (warm-amber ≥20°C, cold-blue <0°C, blue left-bar on rain ≥5mm). Sits alongside `notes-app/` in the same repo.


## dev workflow: Puck up and implement what the design team spesified — 2026-05-12

- **oslo-weather** status updated 2026-05-12: still pre-implementation. `/dev-team` was invoked for implementation but synthesis recommends exiting and implementing in main session per the just-settled handoff protocol. Pre-implementation artifacts produced by the dev sub-team and reusable in main session: (1) complete `symbol_code` → emoji map (~40 codes) from dev-researcher, (2) daily-aggregation algorithm using `next_6_hours ?? next_12_hours` fallback chain, (3) security pre-commit checklist (no personal email in User-Agent, no innerHTML on API data, CSP meta tag, try/catch on cache parse, `?.` on every response path, `If-Modified-Since` on refresh), (4) dev-qa test plan with three P0 threshold boundary tests at 20°C / 0°C / 5mm via `?test` query-string mode.


## design workflow: I would like the design team to put up a webpage for showing — 2026-05-12

- **oslo-weather** spec is now design-complete (2026-05-12). UX delivered the full layout + Norwegian copy + threshold colors; architect locked the four-seam internal structure (SYMBOL_TO_EMOJI / aggregateDaily / fetchForecast / render + `runTests`) and the persistence model (single localStorage key `oslo-weather-cache`, raw MET JSON cached, 30-min TTL). Skeptic reframed the page as an exercise + glanceable companion to phone weather rather than a product — synthesis adopted that framing. Implementation path: main session implements directly off the Build queue entry + committed `met-api-sample.json` fixture per the just-settled handoff protocol; `/dev-team` is for *review* only.


## design workflow: Is the proposal now written to file? — 2026-05-12

- **oslo-weather** status (2026-05-12, post-synthesis): design complete in memory but handoff artifacts NOT on disk. Two pre-implementation gaps remain: (1) `oslo-weather/met-api-sample.json` fixture must be curled from api.met.no and committed — must include ≥1 day past the 48h horizon so the `next_6_hours ?? next_12_hours` fallback has something to bite on; (2) a literal `## Build queue` section must be appended to `decisions.md` with the four-field shape (binary success / stop rule / forcing constraint verbatim / file path + fixture path). Order matters: fixture first, then Build queue card, then main-session implementation. No `/dev-team` round needed until after `index.html` exists.


## dev workflow: Can you pick up where the design team left off and continue? — 2026-05-12

- **oslo-weather** status (2026-05-12, post third `/dev-team` invocation): still pre-implementation. Filesystem check confirms `oslo-weather/` directory, `met-api-sample.json`, `index.html`, and the `## Build queue` H2 in `decisions.md` all do not exist. Synthesis recommended exit; no implementation code written this round. Locked remediation order: (1) commit `oslo-weather/met-api-sample.json` from a real curl spanning ≥5 days, (2) append literal `## Build queue` H2 to `decisions.md` with four-field card, (3) main session implements `index.html`, (4) re-enter `/dev-team` for review only with `dev-qa` + `dev-skeptic` + `dev-security`.
