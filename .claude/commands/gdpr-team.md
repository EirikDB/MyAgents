---
description: Spawn the GDPR sub-team inside eiriks-team to analyze a compliance question, audit scope, DSR, or incident. The team-lead picks which of gdpr-dpo, gdpr-privacy-engineer, gdpr-incident-responder, and gdpr-skeptic are relevant and spawns only those. Synthesis uses a Norwegian-context lens.
argument-hint: <compliance question, audit scope, DSR, or incident description>
---

Bootstrap and run the GDPR sub-team to analyze: $ARGUMENTS

Steps:

0. **Load memory.** Read `.claude/memory/project.md`, `.claude/memory/preferences.md`, and `.claude/memory/decisions.md`. Use this context to (a) frame $ARGUMENTS against the project's existing processing activities and decisions, and (b) include the relevant prior context in each teammate's briefing in step 5.
1. Confirm `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is active. If not, stop and tell the user to enable it in `.claude/settings.json` and restart Claude Code.
2. Establish the team. If `eiriks-team` is already running in this session, **reuse it** — do not delete and recreate. Otherwise `TeamCreate` with `team_name: "eiriks-team"`, `agent_type: "team-lead"`, and a description noting that both design, GDPR, and dev sub-teams operate under this team.
3. **Pick the roster.** Analyze $ARGUMENTS and decide which GDPR teammates are genuinely relevant:
   - `gdpr-dpo` — when the question needs legal interpretation: lawful basis, retention rationale, DPIA trigger, processor/sub-processor framing, transfer mechanism, Norwegian overlay (personopplysningsloven, Datatilsynet decisions). Most GDPR questions benefit from this.
   - `gdpr-privacy-engineer` — when the question touches technical controls: ROPA-to-code mapping, deletion mechanics, DSR fulfillment pipelines, logging design, pseudonymization vs anonymization. Skip for pure-legal or pure-process questions.
   - `gdpr-incident-responder` — when the question is about a DSR, a breach, or an operational playbook under deadline. Skip for documentation-only work.
   - `gdpr-skeptic` — when the work benefits from adversarial review. Usually yes for audits, ROPA reviews, and proposed processes; skip only when the question is too narrow for "what would Datatilsynet flag?" to apply.
   Spawn at least one. State the chosen roster to the user with one-line reasoning before spawning (e.g., "Spawning gdpr-dpo + gdpr-incident-responder + gdpr-skeptic. Skipping privacy-engineer because this is a DSR handling question, not technical-controls design."). If the call is borderline, ask the user briefly before spawning.
4. Create one `TaskCreate` per spawned teammate, parameterized on $ARGUMENTS:
   - **gdpr-dpo** (when spawned): cite the applicable GDPR articles, EDPB guidance, Datatilsynet decisions, and personopplysningsloven overlay. Distinguish required from defensible from recommended. Cover lawful basis (Art 6 + Art 9 if relevant), ROPA fields, retention rationale, DPIA trigger, processor/sub-processor + transfer mechanism.
   - **gdpr-privacy-engineer** (when spawned): translate the legal requirements into concrete controls — ROPA entries mapped to code paths, retention mechanics, deletion playbook across primary stores + backups + sub-processors + derived data, DSR fulfillment pipeline, logging design. Include top 3 technical risks.
   - **gdpr-incident-responder** (when spawned): build the operational playbook — decision tree for breach vs. event, stopwatch with absolute deadlines, Datatilsynet-melding-shaped notification draft, DSR triage flow with proportionate identity verification and exemption analysis, communication templates that don't over-promise.
   - **gdpr-skeptic** (when spawned): play Datatilsynet's investigator. Surface the findings the documentation hides — ROPA gaps, hollow retention, untested DSR flows, weak lawful-basis assertions, shadow data, sub-processor blind spots. Name the single highest-risk gap if Datatilsynet arrived Monday.
5. Spawn the chosen teammates via the Agent tool, using `team_name: "eiriks-team"`. Each prompt must brief the teammate on: the question ($ARGUMENTS), their assigned task ID, the protocol (claim with TaskUpdate, do the work, mark completed, deliver via SendMessage to team-lead), that they should NOT pre-harmonize with peers, **and any relevant prior context from the memory files** (active projects, settled decisions, user preferences).
6. After all deliverables arrive, **if two or more teammates spawned**, run one round of pressure-testing on the highest-leverage conflict(s). Typical patterns:
   - gdpr-dpo vs. gdpr-skeptic on whether the cited basis would survive Datatilsynet enforcement.
   - gdpr-privacy-engineer vs. gdpr-incident-responder on whether the technical evidence actually exists for the playbook to work.
   Send focused challenges to the teammates who own each conflict; collect rebuttals. If only one teammate spawned, skip pressure-testing.
7. Synthesize: a single recommended direction with explicit tradeoffs. If a `gdpr-skeptic` was spawned, lead with their highest-risk finding and what the others would do about it. Be Norway-specific where it matters (Datatilsynet decisions, personopplysningsloven derogations, sector-law overlays). If only one teammate spawned, deliver their analysis with light framing. **End the synthesis with this caveat verbatim**: "This analysis is decision support, not legal advice. Anything bound for Datatilsynet or for a data subject as a formal response should be reviewed by a human DPO or lawyer before issue."
7.5. **Save memory.** Before shutting down teammates:
   - Update `.claude/memory/project.md` with any new project context (processing activities discussed, lawful bases established, sub-processors identified).
   - Append to `.claude/memory/decisions.md` any positions solidified during synthesis.
   - Update `.claude/memory/preferences.md` if new user preferences surfaced.
   - Each agent updates its own `.claude/memory/agents/{role}.md` as part of shutdown (their system prompts instruct this).
8. Send `shutdown_request` to each spawned teammate and wait for acks. **Do not `TeamDelete`** — other workflows (e.g., `/design-team`, `/dev-team`) may want the same `eiriks-team` later in this session. Teardown is the user's call.

If the user interrupts mid-flow, hand control back to them; don't auto-continue.
