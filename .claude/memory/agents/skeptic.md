# Skeptic Agent Memory

_What I've learned about failure patterns relevant to this user's work. Updated after each session._

## About this user's product space

- Building in the notes/productivity category — the most-saturated graveyard in software. Prior art: Evernote, Google Keep, Bear, Notion, Obsidian, Roam, Logseq, dozens of dead products.
- The user is aware of this and builds with a "earn your wedge" mindset. Skeptic critique lands.

## Arguments that landed (user accepted)

- Classification-at-capture is a tax users stop paying (Evernote lesson). Revised: tags as resurfacing schedules, not folder labels.
- Discovery is the wrong direction of arrow. Tools that optimize "find your notes" die; tools that resurface notes unbidden survive.
- Browser push is unreliable — 70% denial rate, silently throttled by OS. Email is the durable channel.
- Sequencing argument: ship email loop alone, measure it, don't merge browse view until two binary questions come back green.

## Arguments that were revised (conceded after rebuttal)

- "Don't build a browse view at all" → softened to "don't merge it until email loop is validated." UX and architect both independently landed on email-first; that convergence is real signal.
- LLM-query-over-notes session (2026-05-11): held the line that "ship email loop first" beats both read-only chat AND the "Ask & apply" forcing-constraint variant — even when the constraint genuinely improves the feature. Team-lead specifically asked whether the constraint *redeems* the feature; correct answer is "yes vs. read-only chat, no vs. shipping email loop first." Don't let "I improved the feature" become "therefore build it now."
- **dyreid.no redesign session (2026-05-11):** conceded a single `<h1>` insertion when UX proposed it. Originally said "zero HTML structure changes" as the forcing constraint; correctly recognized that this was an anti-yak-shave heuristic, not a lexical-purity rule. **The right move was not to die on "zero" but to sharpen to "exactly one node, no wrapper, no class, no handler, no second insertion."** Conceding without sharpening would have invited creep (h1 → header wrapper → nav → section grouping → 40-line refactor). Lesson: when conceding a forcing constraint, *tighten what remains* rather than just nodding through.

## Failure modes to watch for this product

- Graveyard accumulation: stale notes grow monotonically, digest gets noisy, user stops opening the tab.
- Reminder-channel collapse: browser push gets muted → team bolts on email → then SMS → now you're a multi-channel notification platform.
- LLM-editor displacement: ChatGPT Projects / Claude Projects absorb "remember this for me" as a free side-effect of conversation memory within 18 months.
- Configuration sprawl: trying to please GTD + bullet-journal + Zettelkasten crowds adds knobs until onboarding becomes a tutorial.
- **Query-your-notes graveyard (named 2026-05-11):** Mem.ai, Reflect, Rewind, Notion AI workspace Q&A, Obsidian Smart Connections — every "ask your notes" product has either pivoted, stagnated, or sits as a sliver-adoption power-user feature. Pattern: great demo, mediocre product. Wins applause in a screenshot, sits unused after week two. Notes corpora are too small, too recent, and too self-authored for retrieval to be a top-3 problem.
- **Signal contamination from parallel feature streams:** building two Phase-1 features in parallel means a retention bump can't be attributed to either. The validation experiment is poisoned. Always push for sequential validation when phases share a metric.
- **"Build it properly the first time" as architect failure mode:** demanding production-grade infrastructure (proxy, key security, embedding pipelines) at MVP stage delays answering the prior question "should this exist at all." When the prior question has a 70%+ chance of "no," the infrastructure was wasted.

## Patterns to apply in future sessions

- Always name the graveyard. What products tried this before and why didn't they win?
- Push hard on "delivery vs. retrieval" distinction. Most product ideas optimize retrieval; the valuable ones optimize delivery.
- Propose an MVP that answers exactly two binary questions. Nothing bigger.
- When a feature is a thin layer of LLM novelty over an existing problem, ask: "what does the LLM editor do here for free in 18 months?" If the answer is "this exact feature," it's displacement-vulnerable.
- The forcing constraint trick: don't just argue "don't build." Offer "if forced, *this one constraint* changes the product class." That gives the team a structural lever even if they overrule the don't-build verdict. But hold the line that an improved-feature is still not a *now-feature* if sequencing argues against it.
- **When a forcing constraint gets pressure-tested, sharpen what remains rather than just conceding.** A constraint that survives "concede one thing, keep the rest tighter" has more teeth than the original. Pattern: original "zero X" → sharpened "exactly one X, with these specific sub-restrictions."
- Cost shape matters: bursty-use × large-context-per-use is a margin trap that masks itself at single-user self-host scale.
- Trust collapse on first hallucination: in personal-truth domains (notes, journals, health), tolerance for fabrication is roughly zero. One wrong answer kills the feature permanently for that user. Surface this risk explicitly.
- **Verify the brief's characterization of references and prior art.** Briefs often misdescribe what an external reference actually is (this session: "personal portfolio" → actually a corporate WordPress veterinary service site). Always fetch and look. A wrong characterization in the brief is often a leverage point for the critique because it means the team is already operating on bad framing.

## Surface-semantics framework (landed 2026-05-11 dyreid redesign session)

When a product spans multiple delivery surfaces (e.g., browser UI + email digest + push + SMS), the surfaces have **different attention semantics** and unified visual components across them is usually a category error:

- **Ambient surfaces** (browser tab the user voluntarily opened, in-app feed): user came on their own. Visual treatment must invite attention without hectoring. Soft alarms, warm-engaging palettes, decorative alert panels make sense.
- **Interruptive surfaces** (email arriving in inbox, push notification, SMS): the surface's mere arrival is the alert. Decorative alarm semantics inside the surface are *redundant* — they compete with surrounding inbox/notification chrome rather than reinforcing the message.

The honest design pattern: **shared design tokens** (palette family, font, type scale, radius) but **divergent components** where attention semantics differ. A "cream Due Now panel" can be correct in the browser and wrong-by-redundancy in the email — both true simultaneously, both shipped, no contradiction.

This framework landed cleanly with the team-lead and was named as "the sharpest insight of the session." Apply it any time the team proposes a visual treatment that's supposed to "carry across" multiple surfaces — especially across notification/delivery channels.

## Visual-polish-before-validated-loop failure pattern (2026-05-11)

When a project has unanswered binary validation questions about its core loop, redesigning visual surfaces is **yak-shaving disguised as polish**. Specific tells:
- The surface being restyled doesn't carry the metric (browser UI when email loop carries Phase 1 metric).
- The reference being copied is from a different product category (here: corporate veterinary site → personal note-capture tool).
- The redesign is framed as "fun, contained" — but absent a forcing constraint, contained projects expand to fill available time.

Graveyard ammunition for this pattern: Evernote 2020 redesign (power-user revolt, retention drop attributed to redesign), Skype 2017 (copied consumer-social aesthetic, lost enterprise wedge to Teams), Path (every design award, no users), Mailbox 2013 (gorgeous, killed in 2015), the Dribbble-tier productivity-clone cohort (visually superior to Notion/Things/Apple Notes, lost anyway). Productivity tools where chrome should disappear are *least* sensitive to aesthetic differentiation — users look at their content, not the frame.

Forcing constraint that worked: "ship the redesign *inside* the email-loop release, not before it; restyle both surfaces in the same PR or neither ships." Couples the cosmetic urge to the actual core-loop work. If the team can't ship the underlying feature, the visual exercise is permanently on hold.
