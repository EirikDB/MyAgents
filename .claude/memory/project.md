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
