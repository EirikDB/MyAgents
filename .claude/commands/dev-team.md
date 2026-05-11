---
description: Spawn the development sub-team inside eiriks-team to analyze or progress a backend/database/frontend task. The team-lead picks which of dev-backend-dotnet, dev-database-postgres, dev-frontend-react, dev-researcher, dev-skeptic, dev-security, and dev-qa are relevant for the question and spawns only those.
argument-hint: <development task, migration step, or implementation question>
---

Bootstrap and run the development sub-team to address: $ARGUMENTS

Steps:

0. **Load memory.** Read `.claude/memory/project.md`, `.claude/memory/preferences.md`, and `.claude/memory/decisions.md`. Use this context to (a) frame $ARGUMENTS against the active project, the migration state, and settled decisions, and (b) include the relevant prior context in each teammate's briefing in step 5.
1. Confirm `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is active. If not, stop and tell the user to enable it in `.claude/settings.json` and restart Claude Code.
2. Establish the team. If `eiriks-team` is already running in this session, **reuse it** — do not delete and recreate. Otherwise `TeamCreate` with `team_name: "eiriks-team"`, `agent_type: "team-lead"`, and a description summarizing $ARGUMENTS.
3. **Pick the roster.** Analyze $ARGUMENTS and decide which dev teammates are genuinely relevant:
   - `dev-backend-dotnet` — endpoint design, C# code, auth flow, API contracts.
   - `dev-database-postgres` — schema, indexes, migrations, Supabase data path.
   - `dev-frontend-react` — React / TypeScript components, Supabase-client replacement, auth from the client side.
   - `dev-researcher` — when the question would benefit from outside grounding (library comparison, prior art, "how do others solve X"). Skip on questions the team can answer from project context alone.
   - `dev-skeptic` — when the work would benefit from adversarial review (migration plans, schema changes, refactors, anything with a rollback risk). Usually yes for plan-shaped work; skip for narrow lookup-style questions.
   - `dev-security` — when new dependencies are added, API surfaces change, auth/auth flows are modified, secrets or credentials are handled, or code touches user data. Also spawn for any Supabase → .NET migration work (RLS-to-app-layer authorization gap). Skip for pure UI work with no new dependencies or API changes.
   - `dev-qa` — when new features are implemented, existing code is refactored, or work is approaching a release gate. Produces a coverage gap report, writes missing tests, and gives a yes/no ship verdict. Skip for pure design, research, or planning tasks where no code is written. **Key rule: dev-qa never modifies a failing test without explicit team-lead or user approval — always escalates first.**
   Spawn at least one. State the chosen roster to the user with one-line reasoning before spawning (e.g., "Spawning dev-backend-dotnet, dev-database-postgres, dev-skeptic. Skipping dev-frontend-react because this is server-side only, and skipping dev-researcher because the library choice is already settled."). If the call is borderline, ask the user briefly before spawning.
4. Create one `TaskCreate` per spawned teammate, parameterized on $ARGUMENTS:
   - **dev-backend-dotnet** (when spawned): concrete API design, endpoint contracts, C# snippets, auth flow, migration sequencing, top 3 backend risks.
   - **dev-database-postgres** (when spawned): DDL, index strategy, migration plan (pre/data/post), RLS decision, top 3 database risks.
   - **dev-frontend-react** (when spawned): inventory of Supabase call sites and replacements, component / hook contracts, TypeScript snippets, auth flow from the client side, top 3 frontend risks.
   - **dev-researcher** (when spawned): restate the question, recommend one library / pattern / approach, name the runner-up, three caveats, references.
   - **dev-skeptic** (when spawned): strongest "don't ship this plan as-is" argument, named failure modes with precedent, highest-risk gap, minimum hardening, what each peer is under-weighting.
   - **dev-security** (when spawned): dependency CVE audit (name, pinned version, known CVEs, recommendation), API surface review (auth gaps, CORS, rate limiting, input validation, secrets exposure), vulnerability list prioritized Critical/High/Medium/Low with concrete fixes for Critical and High, single highest-risk gap.
   - **dev-qa** (when spawned): coverage gap report (P0–P3 gaps named specifically), test code for every P0 and P1 gap, failing test report (code fixed not tests), test-change audit (flag any weakened tests), yes/no release gate verdict. Any failing test that requires changing the test — not the code — must be escalated to team-lead before action.
5. Spawn the chosen teammates via the Agent tool, using `team_name: "eiriks-team"`. Each prompt must brief the teammate on: $ARGUMENTS, their assigned task ID, the protocol (claim with TaskUpdate, do the work, mark completed, deliver via SendMessage to team-lead), that they should NOT pre-harmonize with peers, and any relevant prior context from the memory files.
6. After all deliverables arrive, **if two or more teammates spawned**, run one round of pressure-testing on the highest-leverage conflict(s). Typical patterns:
   - Backend vs. database on a schema-vs-query trade-off.
   - Frontend vs. backend on the API contract shape.
   - Skeptic vs. anyone on rollback/cutover discipline.
   - Researcher vs. specialists on library choice.
   Send focused challenges to the teammates who own each conflict; collect rebuttals. If only one teammate spawned, skip pressure-testing.
7. Synthesize: a single recommended direction with explicit tradeoffs. If a `dev-skeptic` was spawned, lead with their highest-risk gap and what the specialists would do about it. Concrete artifacts (DDL, code snippets, endpoint contracts, migration steps) belong in the synthesis, not just in the individual deliverables. If only one teammate spawned, deliver their proposal with light framing.
7.5. **Save memory.** Before shutting down teammates:
   - Update `.claude/memory/project.md` with any new project context (architecture choices, migration state, libraries adopted).
   - Append to `.claude/memory/decisions.md` any positions solidified during synthesis.
   - Update `.claude/memory/preferences.md` if new user preferences surfaced.
   - Each agent updates its own `.claude/memory/agents/{role}.md` at shutdown.
8. Send `shutdown_request` to each spawned teammate and wait for acks. **Do not `TeamDelete`** — other workflows (e.g., `/design-team`, `/gdpr-team`) may want the same `eiriks-team` later in this session. Teardown is the user's call.

If the user interrupts mid-flow, hand control back to them; don't auto-continue.
