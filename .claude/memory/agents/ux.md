# UX Agent Memory

_What I've learned about this user's UX context and preferences. Updated after each session._

## About the user's products

- Building developer/productivity tools — CLI-adjacent sensibility even in web UIs.
- Wants opinionated, minimal surfaces. Reacts well to cut lists and explicit "we are NOT building" decisions.
- Appreciates when UX decisions are tied to delivery outcomes, not aesthetics.

## What landed well

- Inline syntax (`#tag`, `@time`) as the sole authoring interface — no modals, no pickers.
- Email digest framed as "the killer feature" and treated as a first-class surface, not a notification setting.
- Three-zone home (Due Now / Coming Up / Recent) over a notes-list landing.

## What was revised under pressure

- Tags-as-retrieval (original) → tags-as-resurfacing-schedules (revised after skeptic's Evernote-tax critique). Tags now drive delivery: `#followup` escalates, `#idea` resurfaces, etc.
- `!priority` syntax: cut. It was pure retrieval scaffolding and didn't earn its place on the capture path.

## Patterns to apply in future sessions

- Lead with delivery mechanics first, UX chrome second. This user will push back if classification doesn't feed the delivery channel.
- Cut lists are as important as feature lists — invest equal time in both.
- Default assumptions: one-screen UI, inline syntax, email-first, zero onboarding ceremony.

## Session 2026-05-11 — Claude API query for notes-app

Proposed: capture box doubles as query box, mode-triggered by leading `?` (no second input, no modal, no "AI" button). Results render as ephemeral fourth zone above Due Now. Always-visible sources (real notes, not paraphrased). No chat history.

- **Inline-syntax precedent extends to mode-switching.** This user's products already use `#`/`@` as one-character mode signifiers; adding `?` as a third one is consistent, not novel. Reach for sigil-prefix before reaching for a new input element.
- **Existing literal search (`#search-input`) is a separate job from LLM query.** Resist collapsing them — literal search is instant/free/deterministic; LLM is interpretive. Two jobs, two surfaces; not duplicative.
- **Trust mechanics in LLM-over-X products belong on the durable surface.** Defended: always-visible sources, real-note rendering (not paraphrased snippets), latency + note-count badge. The clutter IS the feature here — collapsing it under "show sources" would invite the exact hallucination-trust failure that kills these products. Hold this line against simplification pressure.
- **Chat history is a parallel durable artifact.** In a notes-app, that's a trap — it duplicates the storage/search/export surface area the user has already cut once (Evernote-tax pattern). Default to one-shot queries, ephemeral results, no transcript. Notes stay the only durable thing.
- **Header badge as honesty contract.** Surfaces like `4 notes · 2.1s` or `20 of 200 considered` make retrieval surface-area visible to the user. Never let a UI lie about what was actually searched.

## Session 2026-05-11 — dyreid.no visual redesign for notes-app

Proposed: full visual refresh of notes-app to mirror dyreid.no's teal-and-cream brand language. Pure-CSS variable swap, zero HTML changes (after pressure-test concession), no Typekit dependency. Cream `#F8EDE5` panel for Due Now reframes "needs attention" from anxiety-coded red to warm-engaging cream.

- **Extract palette from real CSS, never trust a WebFetch summary.** I curled the actual Elementor + Typekit stylesheets and got the exact brand tokens (`#0D5257`, `#F8EDE5`, `#E6F6F8`, `#61CE70`, weights, sizes). The first WebFetch summary was vague and would have led to guessing. Pattern: identify the live CSS file from `<link>` tags, fetch raw, grep for hex frequencies and font-faces. This is repeatable for any visual-refresh task.
- **The distinctive signal is the color *pairing*, not individual colors.** Dyreid's identity is teal+cream together. Stealing one without the other loses the brand. Same will be true for any future "match brand X" task — find the pairing, not the swatch.
- **Adapt saturation for context: marketing-once vs. tool-all-day.** Full-saturation cream (`#F8EDE5`) works on a one-visit homepage; for a productivity tool you stare at 20×/day, dilute to `#FCF7F2` for app bg and *reserve* the full cream for the one panel that needs attention. Generalizes: when porting marketing aesthetics into productivity tools, halve the saturation of warm/saturated tokens, keep the brand-hue intact.
- **Asymmetric semantic layering.** Cream at zone-level (engaging) + red at item-level (alarm) is stronger than either signal alone at one level. The double-coding tells the user "here is your day → this specific one is late" rather than "you're in trouble." Reuse this pattern: zone-level and item-level signals should *cooperate*, not duplicate.
- **CTA carries brand better than wordmark in tools.** Conceded the proposed `<h1>Notes</h1>` under skeptic's zero-HTML constraint. Reasoning: in a productivity tool, buttons get *used*, wordmarks get *tuned out*. A teal-fill save button at the top + cream panel below = stronger identity than a teal title would have been. Default: in tools, push brand identity into the CTAs and the highest-traffic surface, not into chrome.
- **The `::before content:` workaround for missing semantic markup is a trap.** Visually present but semantically empty — lies to screen readers, adds maintenance debt. Either change the HTML or drop the affordance. Don't fake semantic structure with pseudo-elements.
- **Test every palette decision against the email-digest surface for this user.** Their products are email-first. A palette that can't be expressed in inline `bgcolor` + table layout is automatically wrong. Cream + teal happens to survive Gmail/Outlook clipping beautifully (one of the wins of this proposal); black/red would have spam-flagged in inbox previews. Pattern: before finalizing any browser palette, sketch the same surface as an inline-styled table to confirm it carries.
- **Calibrated defense ratio: defend principles, concede details.** Defended cream-not-red (principle: emotional register), no-Typekit (principle: single-file constraint), no-third-hue (principle: discipline-as-design). Conceded H1 (detail: structural change for a wordmark). Team-lead noted both calls were "exactly right." Pattern to keep: when pressure-tested, identify which calls protect a core decision vs. which are nice-to-have polish, and concede the polish freely.

## Session 2026-05-11 — vg.no visual redesign for notes-app (superseded dyreid)

Proposed: port vg.no's typographic urgency hierarchy (kicker pill / headline / red label) onto notes-app's three zones. **Adopted by synthesis as "the core urgency vocabulary."** Drop-in `:root` swap; warm-tinted neutrals (`#faf6f6`/`#322525`); kicker yellow `#fff6d1`+`#4c3d00` text for "today" items; loud red `#dd0000` filled pill only for overdue; three-tier escalation `plain → yellow → red`. Zero HTML/JS changes.

- **The CSS-extraction pattern is now habit and it keeps paying off.** WebFetch summary described vg.no as "calm, authoritative, broadsheet-like" — which is laughably wrong. Curling `/front-page-assets/_astro/*.css` and grepping for `--color-red-*` / `--color-yellow-*` got the actual token values (`red-500: rgb(221,0,0)`, `yellow-300: rgb(252,168,72)`, etc.). **Default for any "match brand X" task: never trust a WebFetch summary on visual tone. The summarizer is biased toward neutral descriptions of strong visual identities.**
- **Tabloid → productivity is mostly category error, but typographic discipline transfers.** The shippable insight: separate the *transferable scaffold* (kicker pattern, 3-tier urgency stack, warm neutrals) from the *content vehicle* (images, density, breaking-news strips, bylines, hero gradients). When asked to port aesthetic X into context Y, name what's transferable vs. category-error before listing values. Cut list is the load-bearing artifact.
- **"Zone headers are chrome, not editorial voice."** Defended against architect's serif-italic red Due Now header. Failure-mode phrase to reuse: **"editorial voice masquerading as state."** Serif italic at 11px sans-stack renders as faux-slant (no true italic glyph in -apple-system / Segoe UI at small sizes); also Gmail/Outlook table-cell-friendly is the test. The pill wins on every axis. General rule: italic + serif belongs to prose; UI state belongs to filled backgrounds and uppercase pills.
- **The 3-tier urgency stack is now the canonical pattern across two consecutive redesigns.** Dyreid was cream-zone + red-item. VG extends it to `plain (coming-up) → yellow (today) → red (overdue)` — same asymmetric layering principle, more resolution. This is the user's load-bearing visual semantics for productivity tools and should be the default starting point for any future urgency-encoding question.
- **Honest impact estimate is more credible than inflated.** When skeptic's pattern critique landed (productive procrastination — second reference in two weeks, neither shipped), I conceded the kicker bumps done-rate `+1–3%` at best and the second-capture metric `~0%`. Calibrated estimates are how I keep credibility for the next defense. **Default: if asked "will this move the needle," quote a number and bound it. Avoid "could meaningfully improve."**
- **Frame redesigns as PR-riders, not projects.** Should have led the original deliverable with "this rides inside the email-loop PR per decisions.md; if it can't, kill it." Visual refreshes presented as standalone work invite the productive-procrastination critique correctly. Pattern to apply: any non-blocking polish work should be presented with its host PR named up-front, not as a deliverable in its own frame.
- **"Tools yield attention back; news seizes it."** One-line decoder for "should I port aesthetic X into productivity tool Y." If X's job is to grab attention (news, social, ads, dashboards), most of its surface vocabulary inverts in Y. The few patterns that survive are the ones that encode *state* not *content* (kickers, badges, color-coded labels).
- **Synthesis adopted the kicker; team-lead's word was "strongest contribution."** Pattern: when proposing visual systems, lead with the one pattern that has both (a) a clear product job to do and (b) brand-identifying force. The kicker had both — yellow pill = unmistakably VG, AND it does the "signal state without screaming" job. Future visual proposals: identify and lead with the one pattern that scores on both axes; everything else is supporting cast.
