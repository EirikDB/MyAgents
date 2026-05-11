---
description: Spawn the design sub-team inside eiriks-team to explore a product or feature. The team-lead picks which of ux, architect, and skeptic are relevant for the question and spawns only those.
argument-hint: <product or feature description>
---

Bootstrap and run the design sub-team to explore: $ARGUMENTS

Steps:

0. **Load memory.** Read `.claude/memory/project.md`, `.claude/memory/preferences.md`, and `.claude/memory/decisions.md`. Use this context to (a) frame $ARGUMENTS against what's already been decided or built, and (b) include the relevant prior context in each teammate's briefing in step 5.
1. Confirm `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is active. If not, stop and tell the user to enable it in `.claude/settings.json` and restart Claude Code.
2. Establish the team. If `eiriks-team` is already running in this session, **reuse it** — do not delete and recreate. Otherwise `TeamCreate` with `team_name: "eiriks-team"`, `agent_type: "team-lead"`, and a description summarizing $ARGUMENTS.
3. **Pick the roster.** Analyze $ARGUMENTS and decide which design teammates are genuinely relevant:
   - `ux` — when the question touches command surface, defaults, output, UI, or what NOT to build.
   - `architect` — when the question touches system shape: language, persistence, distribution, performance, module boundaries.
   - `skeptic` — when the proposal benefits from dissent at the product level. Usually yes; only skip on narrow, mechanical questions where there's no real product call to push back on.
   Spawn at least one. State the chosen roster to the user with one-line reasoning before spawning (e.g., "Spawning ux + skeptic. Skipping architect because this is a copy/defaults question, not system shape."). If the call is borderline, ask the user briefly before spawning.
4. Create one `TaskCreate` per spawned teammate, parameterized on $ARGUMENTS:
   - **ux** (when spawned): opinionated UX proposal for $ARGUMENTS — command surface, defaults, output format, filters, CI integration, what's NOT being built, top 3 UX decisions.
   - **architect** (when spawned): concrete architecture proposal for $ARGUMENTS — language choice, module boundaries, persistence model, performance target, distribution, test strategy, top 3 technical risks.
   - **skeptic** (when spawned): argue against $ARGUMENTS — real-problem framing, prior art that didn't dominate, what kills it in 18 months, what peers under-weight, strongest "don't build this", single forcing constraint if forced to build, the MVP that proves the concept.
5. Spawn the chosen teammates via the Agent tool, using `team_name: "eiriks-team"`. Each prompt must brief the teammate on: the product context ($ARGUMENTS), their assigned task ID, the protocol (claim with TaskUpdate, do the work, mark completed, deliver via SendMessage to team-lead), that they should NOT pre-harmonize with peers, **and any relevant prior context from the memory files** (active projects, settled decisions, user preferences).
6. After all deliverables arrive, **if two or more teammates spawned**, run one round of pressure-testing: identify the highest-leverage conflict(s), send focused challenges to whichever teammates own them, and collect rebuttals. If only one teammate spawned, skip pressure-testing.
7. Synthesize: a single recommended direction with explicit tradeoffs. Highlight where teammates converged after debate and where they didn't. Be opinionated — don't water it down. If only one teammate spawned, deliver their proposal with light framing rather than reconciliation.
7.5. **Save memory.** Before shutting down teammates:
   - Update `.claude/memory/project.md` with any new or changed project context from this session.
   - Append to `.claude/memory/decisions.md` any decisions solidified during synthesis.
   - Update `.claude/memory/preferences.md` if new user preferences were revealed.
   - Each agent will update their own `.claude/memory/agents/{role}.md` as part of shutdown (their system prompts instruct this).
8. Send `shutdown_request` to each spawned teammate and wait for acks. **Do not `TeamDelete`** — other workflows (e.g., `/gdpr-team`, `/dev-team`) may want the same `eiriks-team` later in this session. Teardown is the user's call.

If the user interrupts mid-flow, hand control back to them; don't auto-continue.
