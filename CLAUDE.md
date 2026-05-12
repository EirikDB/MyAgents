# agent-team2

A scaffold for running **eiriks-team** — a Claude Code agent team with three sub-teams that share a session:

**Design sub-team** (spawned by `/design-team`):
- **ux** — opinionated UX proposal (commands, defaults, output format, what NOT to build)
- **architect** — concrete technical architecture (language, parsing, persistence, top risks)
- **skeptic** — devil's advocate critique at the product/feature level

**GDPR sub-team** (spawned by `/gdpr-team`):
- **gdpr-dpo** — legal interpretation grounded in GDPR articles, EDPB guidance, Datatilsynet decisions, personopplysningsloven
- **gdpr-privacy-engineer** — technical controls, ROPA-to-code mapping, retention mechanics, deletion playbooks, DSR pipelines
- **gdpr-incident-responder** — 72h breach response, Datatilsynet melding form, DSR triage flow, calibrated communication
- **gdpr-skeptic** — adversarial auditor playing Datatilsynet's investigator

**Development sub-team** (spawned by `/dev-team`):
- **dev-backend-dotnet** — ASP.NET / C# REST API design and implementation (Supabase replacement target)
- **dev-database-postgres** — PostgreSQL schema, indexes, migrations (incl. Supabase → standalone Postgres data path)
- **dev-frontend-react** — React + TypeScript, primary engagement: migrating from `@supabase/supabase-js` to a typed REST client
- **dev-researcher** — prior art, library comparisons, current best practices — recommends, doesn't survey
- **dev-api-specialist** — third-party API evaluation BEFORE integration (auth, rate limits, SLA, versioning, webhooks, sandbox, vendor lock-in)
- **dev-skeptic** — implementation-level adversarial review (rollback, lock contention, scope creep, type drift)
- **dev-security** — dependency CVE audits, API/data-flow exposure review, auth and secrets validation
- **dev-qa** — test coverage audit, missing-test authoring, "nothing ships untested" gatekeeper

All fifteen teammates operate under one underlying team called `eiriks-team`. This sidesteps the "one team per session" limitation — you can run `/design-team`, `/gdpr-team`, and `/dev-team` in the same session without tearing the team down between them.

## How to use

In a Claude Code session in this folder, run one of:

```
/design-team <description of product or feature>
/gdpr-team <compliance question, audit scope, DSR, or incident description>
/dev-team <development task, migration step, or implementation question>
```

Each slash command bootstraps the relevant sub-team, runs the exploration, pressure-tests the highest-leverage conflicts (when two or more teammates spawned), synthesizes the result, and shuts down its spawned teammates. **None of the commands run `TeamDelete`** — the team stays alive for the next workflow in the session.

**All three commands let the team-lead pick the roster.** The lead analyzes the question, picks which teammates from the sub-team are genuinely relevant, states the choice with one-line reasoning, and spawns only those. Token cost scales linearly with how many teammates spawn — a narrow question can run with one teammate; a broad one with the full sub-team. The lead asks the user briefly when the call is borderline.

## Prerequisites

`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be set (already configured in `.claude/settings.json`). Agent teams remain experimental as of 2026-05; restart Claude Code after cloning this repo for the flag to take effect.

## How to evolve the team (the training loop)

The fifteen teammates' system prompts live in `.claude/agents/`. Each is a markdown file with YAML frontmatter (name, description, optional `tools`, optional `model`) and a body that becomes the teammate's system prompt.

When a teammate misses something or pushes the wrong direction:

1. Edit the corresponding `.claude/agents/<role>.md`.
2. Commit the change. The git history *is* the training history.
3. Next slash-command invocation runs against the updated definition.

To make a teammate cheaper/faster, add `model: sonnet` (or `model: haiku`) to its frontmatter. Without that line, teammates inherit the project's default model.

Current model assignments:
- **Sonnet** (synthesis/lookup/templated output): `ux`, `dev-researcher`, `dev-api-specialist`, `dev-qa`, `gdpr-dpo`, `gdpr-privacy-engineer`, `gdpr-incident-responder`
- **Default / Opus** (code production and adversarial reasoning): `architect`, `dev-backend-dotnet`, `dev-database-postgres`, `dev-frontend-react`, `dev-security`, `skeptic`, `dev-skeptic`, `gdpr-skeptic`

Rule of thumb: code-producing roles and skeptics stay on the larger model — cheapening the skeptics turns them into rubber stamps.

To change *how* a sub-team is bootstrapped (different roles, different task structure, different synthesis style), edit `.claude/commands/<command>.md`.

## What lives where

| Location | Purpose | Edit by hand? |
|---|---|---|
| `.claude/agents/*.md` | Durable teammate definitions — **the part you train** | Yes; commit changes |
| `.claude/commands/{design-team,gdpr-team,dev-team}.md` | Slash command bootstrap recipes | Yes |
| `.claude/settings.json` | Claude Code settings (experimental flag lives here) | Yes |
| `.claude/memory/*.md` | Shared cross-session memory (project, decisions, preferences) read by every teammate | Yes; agents also append to this |
| `.claude/memory/agents/<role>.md` | Per-agent accumulated learnings, written at shutdown | Usually no — agents manage these |
| `~/.claude/teams/eiriks-team/config.json` | Runtime team config | **No** — managed by harness |
| `~/.claude/tasks/eiriks-team/` | Runtime task list (ephemeral) | **No** |
| `~/.claude/projects/<this-folder>/memory/` | Auto-memory (cross-session facts the harness saves) | Separate from team training; let it manage itself |

## Adding a new role

1. Create `.claude/agents/<role>.md` with frontmatter and system prompt. Match the style of an existing role.
2. Add the role to step 3 (task) and step 4/5 (spawn) of the relevant `.claude/commands/<command>.md` — or create a new slash command for a new sub-team. New sub-teams should target `team_name: "eiriks-team"` and reuse the team if it's already running.
3. Commit.

## Limitations to be aware of

- One team per session (which is why all sub-teams share `eiriks-team`).
- `/resume` and `/rewind` don't work with in-process teammates.
- Split-pane mode requires tmux or iTerm2; on Windows you get the default in-process mode (cycle teammates with `Shift+Down`, view shared task list with `Ctrl+T`).
- The GDPR sub-team produces decision support, not legal advice. Anything bound for Datatilsynet or for a data subject as a formal response should be reviewed by a human DPO or lawyer.
- Each spawned teammate is a separate Claude session — token cost scales linearly with how many spawn per command. The `/dev-team` "subset" pattern exists for that reason; use it.
