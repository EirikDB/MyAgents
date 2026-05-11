# Architect Agent Memory

_What I've learned about this user's technical context and constraints. Updated after each session._

## About the user's stack

- Open to Go for backend tooling — single static binary distribution is valued.
- Starts with the simplest working thing (localStorage before SQLite, HTML before React).
- Phase-gates complexity deliberately: prove the concept, then add infrastructure.

## What landed well

- Single static binary story (Go + embed.FS). No runtime deps, no separate worker tier.
- SQLite-first for self-host, Postgres path for multi-user — same interface, gated by config.
- Store interface as the decoupling seam (compute never imports sqlite directly).
- htmx + Preact island: avoiding the SSR/hydration tax for a mostly-form-and-list surface.

## Top risks to flag for this product space

- Reminder delivery across sleeping laptops / disconnected clients — catch-up pass on startup.
- Classifier quality — always a suggestion, never auto-applied. Track acceptance rate.
- SQLite single-writer scaling for multi-user — don't try to scale it past single-user.

## Patterns to apply in future sessions

- This user doesn't want a plugin system or abstraction layers in v1. Name what you're NOT building.
- Distribution is always part of the design — static binary or equivalent is the bar.
- Persist state only for user-authored content; everything else should be computable from inputs.

## Lessons from notes-app + Claude API session (2026-05-11)

- **Distinguish "MVP infra" from "production infra" up front.** I argued for a Go binary in v1 to solve API key exposure. Team-lead correctly pushed back: for a 4-6 week self-hosted single-user validation prototype, browser-only BYOK in localStorage is sufficient. The Go binary is the v2 target once the binary success metrics clear. Lesson: when proposing infra, explicitly ask "does this answer the validation question, or only the production question?" before defending it.
- **Prompt caching is conditional on usage cadence.** Anthropic's ephemeral cache has a ~5-minute TTL. It's load-bearing for conversational/bursty UX (2-4 questions per session) and worthless for daily/intentional cadence. Don't bake caching into the architecture before knowing query frequency. At daily cadence with ~20K input tokens, full-corpus per-query cost is ~$22/user/year — caching matters not at all.
- **Query UX shape changes the output contract.** Conversational query = streaming text. State-change query (skeptic's "every query must end in archive/tag/schedule") = structured tool-use output with a `propose_changes` schema + confirmation step. Pick the lane before committing to streaming vs. structured output.
- **Tiered fast-path/slow-path scanning still applies, but the threshold matters.** For ~500 notes, send everything. For >5K, pre-filter in code (tag mentions + substring match + date hints) before sending. No embeddings/vector store until ~20K notes — linear scan in Go is sub-millisecond, and a vector store buys "why does this stale entry exist" failure modes for no real win.

## Anti-patterns I fell into this session (don't repeat)

- Designing for the key-leak threat model before confirming we're past the single-user self-host phase. The threat model changes between v1-prototype and v2-shipped; design for the current phase.
- Banking on prompt caching as a cost lever without checking query cadence. Caching is a hit-rate problem, not a "free discount" — verify the hit-rate assumption.
- Proposing localhost-token + loopback-bind security theater for a phase that doesn't need it yet. Jupyter-style auth is the right v2 pattern; in v1 with one user, it's overhead.

## Lessons from notes-app dyreid.no visual port (2026-05-11)

- **WebFetch summaries miss technical CSS detail.** When porting a reference site's visuals, the summary tool returned "no font declarations visible" and "no color values exposed" — both wrong. Always `curl` the raw HTML and the linked stylesheets directly. The team-lead specifically called out that the Proxima Nova license finding and the extracted token values were the most useful contributions; both came from `curl` + `grep`, not WebFetch.
- **Brand-port heuristic: tokens > composition.** When the reference site is marketing-shaped (hero bands, grid cards, swipers) and the target is utility-shaped (capture-first list), the brand should land via color/type/button tokens — not by importing the hero composition. I flagged this independently as a likely UX pre-harmonization risk; worth defending again if it comes up.
- **Single-file constraint kills the obvious font answer.** Proxima Nova requires Adobe Typekit (license-locked to domain), and every free near-Proxima (Mona Sans, Manrope, Source Sans 3, Open Sans) requires a Google Fonts `<link>` or self-hosted woff2. For single-file/no-external-deps prototypes, the right call is a refined system stack (`ui-sans-serif, system-ui, -apple-system, ...`). Build the escape hatch as a commented opt-in (`<!-- ENABLE_BRAND_FONT -->`), don't ship it.
- **CSS budget is the cleanest "when to extract" signal.** A hard 10KB inline-`<style>` budget gives a non-arbitrary trigger for going from inline to external stylesheet. Better than "when it feels heavy."
- **Token-only-diff discipline is enforceable.** Rule: any color/spacing change must touch only `:root`. Hex literals outside `:root` are a smell that reviewers can flag without subjective judgment.
- **Cream/warm backgrounds re-open the WCAG AA question.** Cool gray `#6b6b68` on `#f9f9f7` passes; the warmer `#F8EDE5` background drops the contrast ratio enough that small muted text fails. When swapping a neutral palette warm, always re-run axe on the muted pair — don't assume the old token survives.

## Patterns that worked this session

- **"What dyreid.no actually is" preamble before recommendations.** Naming the stack (WordPress/Elementor/Bootstrap, ~MB of CSS, 15 JS files) up front made the "we're extracting tokens, not porting the stack" framing land cleanly.
- **Counting brand-color occurrences as a confidence signal.** `grep -oE '#[0-9A-Fa-f]{6}' | sort | uniq -c | sort -rn` returned 32 hits for `#0D5257` and 1 hit for `#61CE70` — the count itself told me which color is load-bearing and which is decorative. Useful any time the brand isn't documented.
- **Defending the "capture-first, no hero" stance independently.** Resisted pre-harmonizing with UX. Worth doing again — the team-lead reconciler benefits more from honest disagreement than from convergence theater.

## Lessons from notes-app vg.no visual port (2026-05-11)

- **Modern-framework sites put theme tokens in inline `<style>` in the HTML, not in linked CSS.** vg.no's Astro build emitted four ~25–90KB CSS chunks that contained only *consumers* (`color: var(--color-red-500)`) and zero token definitions. The token defs lived in an inline `<style>` block on the HTML itself, under `[data-color-scheme=light]` rules. Lesson: when extracting tokens from any site built since ~2022, search the HTML inline style first; the linked stylesheets are usually consumer-only. The grep that found everything was `grep -oE -- '--color-(red|gray|accent)[a-z0-9-]*\s*:\s*[^;"]+[;"]' vg.html`.
- **WebFetch is still wrong about CSS.** It returned "no CSS visible" / "request the complete source" — both wrong. `curl` + `grep` on the raw HTML and the four linked CSS files found the entire token system in under a minute. The prior session's lesson held up exactly. Always curl raw.
- **Tabloid/news brands warm-tint their neutrals.** VG's "black" is `#1C0000` (rgb 28,0,0) and its grays are red-undertoned (`#FAF6F6`, `#7F6868`). This is a *deliberate* brand-cohesion move — the spot color bleeds into the neutrals. Heuristic: when porting a brand whose chrome is dominated by one saturated hue, audit the "black" and the "grays" — they're probably tinted in the spot's direction, and porting just the saturated color while keeping pure-black/cool-gray neutrals loses half the identity.
- **Find one composition signal that captures voice without new DOM.** Token-only swaps can land flat — the colors change but the *voice* doesn't. For VG, the single move that captured the tabloid feel without inventing markup was switching zone-header chrome from `11px uppercase sans muted` to `13px italic serif`. Heuristic: in any brand port to utility-shaped markup, hunt for the one chrome element where a serif/sans flip or weight flip gives 80% of the voice for 20 bytes of CSS.
- **Single-hue discipline survives a second port.** Dyreid kept one teal; vg.no kept one red; I collapsed the existing blue `--accent-up` into a warm gray both times. Two data points isn't a rule, but it's becoming a default. Worth defending up-front in the next brand port instead of waiting for skeptic to argue it.
- **Optional CSS-only rail beats inventing a hero element.** `body { border-top: 4px solid var(--accent); }` is the cheapest "this is brand X" signal when the markup has no hero region. 30 bytes, zero structural change, defensible against any "we don't have a header element" pushback.
- **The 10KB inline-`<style>` cap is becoming load-bearing.** Used twice now, both held. Mechanical trigger ("any commit that puts CSS over 10KB must extract to a sibling stylesheet") is enforceable in pre-commit (`awk '/<style>/,/<\/style>/' index.html | wc -c`). Default this for any single-file prototype going forward.

## Anti-patterns avoided this session

- Did not propose self-hosting VG's proprietary woff2 (would have violated single-file + no-CDN constraints, and the license bars redistribution anyway). Named the portability problem in the deliverable instead of trying to solve it with a workaround.
- Did not invent a hero/breaking-news banner element to "look more VG." The team's zero-HTML-change constraint and the dyreid round's "tokens > composition" lesson both pointed away from it.
- Did not propose a dark scheme or `[data-skin=breaking]` skin system. Both exist in VG's CSS and would have been easy to port — but neither serves the email-loop validation question, so they stay out of v1.
