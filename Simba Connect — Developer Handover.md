# Simba Connect — Developer Handover

**Prepared for:** the developer polishing the Simba Connect demo and scoping the live build
**Prepared by:** Berelo (Dan Taitz)
**Date:** June 2026
**Status:** Internal working document

---

## 0. How to read this document

There are two jobs here, and they run in order:

- **Part A — Polish the demo (now).** Refine the existing single-file HTML demo so it is bullet-proof in front of PepsiCo / Simba leadership. No backend, no infrastructure. This is the priority.
- **Part B — Build the live bot (next).** A technical brief for the Phase 2 closed WhatsApp pilot — a real number leadership can message. This turns "nice demo" into "it's already operational."

The demo is a **sales artefact**, not a product. Every polish decision should protect one thing: the *feeling* that a consumer is talking to a brand in their own language and the brand is quietly getting smarter. Keep that intact above all else.

A companion strategic note already exists — *"Simba Connect — Taking the Demo Live"* — covering the business case, pricing and compliance. Part B here is the technical complement to it; read both.

---

# PART A — POLISH THE DEMO

## A1. What the demo is

A single self-contained HTML file that simulates a WhatsApp-based "owned consumer channel" for the Simba Chicken LTO (limited-time offer: Roast Chicken vs Sticky Wings, launching Nov/Dec 2026). It is shown to PepsiCo / Simba leadership to contrast a modern conversational channel against a legacy "menu-tree" bot.

It runs three ways:

1. **Guided demo** — a narrated, auto-playing 7-chapter story following one fictional consumer ("Thandi") from first contact to advocacy, with presenter notes for each beat.
2. **"Today's bot" mode** — the same consumer hitting a dumb deterministic menu bot, capturing nothing. The contrast slide.
3. **Free chat** — after the tour (or any time), anyone in the room can type and get a plausible reply via keyword intent-matching, or tap suggestion chips to vote, play the game, send a till slip, etc.

Alongside the phone sits a live **intelligence panel** ("What Simba sees") that fills with profile attributes, an event feed, KPIs and a "compounding asset" meter as the conversation progresses.

**Audience:** non-technical senior marketers. **Setting:** a pitch room and, via a shared link, their own phones.

## A2. Files

| File | Role |
|---|---|
| `Simba Connect - AI Channel Demo v2.html` | **Canonical demo.** The current, mobile-optimised version. Edit this one. |
| `simba-demo-deploy/index.html` | Deploy copy — **byte-identical** to v2. This is what gets hosted. Keep it in sync (or better, make it the single source — see A8). |
| `Simba Connect - AI Channel Demo.html` | v1 (617 lines). Superseded — kept for reference only. |
| `simba-demo-deploy/vercel.json`, `README.md`, `.gitignore` | Static-host scaffolding for the deploy copy. |

The demo has **zero dependencies** — no frameworks, no build step, no external network calls. Vanilla HTML/CSS/JS in one file (~1,025 lines). It opens offline by double-clicking. This is a deliberate strength; preserve it.

## A3. Architecture (how the file is laid out)

```
<head><style> … all CSS, uses CSS custom properties (:root) for theming …
<body>
  header.top        → logo, mode toggle (Today's bot / Berelo way), Play / Reset / Notes
  .ticker           → national "live vote" counter (Wings vs Roast), animated
  .stage
    .col-phone      → chapter pills, the phone (WhatsApp UI), director caption
    .intel #intelNew→ "What Simba sees": profile, KPIs, event feed, compounding-asset bar
    .intel #intelOld→ the dead/"deterministic bot" panel
  .mob-intel-btn    → floating button (mobile only) to open the intel bottom-sheet
  .notes            → presenter notes drawer
  .finale           → end-of-tour overlay (the "ask")
<script> … state, helpers, scripts, engine, free-chat, mode/reset …
```

### State (top of script)
`mode` (`new`/`old`), `playing`, `chIdx`/`stepIdx` (chapter/step cursors), and the running counters `dataPts`, `convs`, `refs`, `reach`, `compVal`. The national counters `natW`/`natR` drive the ticker. `localW`/`localR` drive the on-card vote split.

### The guided-story engine
The whole Berelo-mode story is **data, not code**: a `script` array of 7 chapter objects, each with a `steps` array. Each step is a one-key object whose key is its *kind*:

| Step kind | Renders |
|---|---|
| `u` | consumer (outgoing) message |
| `b` | brand (incoming) message, with typing delay |
| `cap` | director's-cut caption `[tagline, body]` |
| `note` | presenter-note text for that beat |
| `div` | a date/section divider in the chat |
| `card` | a rich card — args `[cssClass, emoji, title, body, buttonLabel]` |
| `photo` | an outgoing photo bubble `[emoji, caption]` |
| `battle` | the tap-to-vote pack-shot widget |
| `quiz` | the 3-question "Chicken Personality" game |
| `leader` | the squad leaderboard |
| `status` | a WhatsApp-Status share mock |
| `panel` | a function that mutates the intelligence panel (adds attrs, events, KPIs, ledger) |
| `fin` | triggers the finale overlay |

`nextStep()` is the heart: it reads the next step, renders it, and schedules the following step. Message timing **scales with text length** (typing time + reading time) so it feels human — see lines ~831-838. `jumpChapter()` lets you click a chapter pill to skip. `togglePlay()` / `stopPlay()` handle play/pause/resume/replay.

### "Today's bot" engine
A separate, simpler `oldScript` array played by `playOld()`. It deliberately fails: every consumer message gets "Please reply with a number between 1 and 5," and the side panel narrates what was lost.

### Free chat (the part people type into)
**There is no LLM here.** `sendFree()` matches the typed text against an `intents` array of regex patterns (`intents[].k`) and returns a canned reply plus optional widget and intelligence-feed event. Order matters — the **first** matching regex wins, so specific patterns must precede general ones. If nothing matches, a friendly catch-all reply fires and still logs a "free-text intent understood" event. Suggestion chips (`enableFree()`) either inject a canned user message or call a `free*()` helper directly.

### Interactive widgets
`addBattle()`/`castVote()`, `addQuiz()`/`renderQuizStep()`/`answerQuiz()`/`finishQuiz()` (+ `autoQuiz()` for guided mode), `addLeaderboard()`, `addStatus()`, `addCard()`, `addPhoto()`. All append DOM nodes to `#chat` and most also push to the intelligence panel via `bumpData/bumpConv/bumpRef/bumpReach/bumpComp`, `addAttr`, `addEv`, `addLedger`.

### Mobile
A `@media (max-width:640px)` block reflows the phone to fill the screen and turns the intelligence panel into a slide-up bottom sheet toggled by the floating 📊 button (`toggleIntel()`). This is recent and is the most fragile area — test it hardest (see A5).

## A4. How to run, host and share

- **Run locally:** double-click `index.html`. No server needed.
- **Host (recommended):** the `simba-demo-deploy/` folder is a ready static site. Drag it onto Netlify Drop, or connect it to Vercel/Cloudflare Pages. `vercel.json` is already configured for static serving.
- **Share:** send the **link**, never the `.html` file (WhatsApp won't run an attached HTML file). Turn the link into a QR code for the room. (See *"Simba Connect Demo — How to Share It"* for the full mobile playbook.)

## A5. Polish backlog (prioritised)

### P0 — must fix before the next leadership demo
- **Mobile QA pass on real devices.** Test the bottom-sheet (`toggleIntel`), the `76dvh` phone height, chip scrolling and the finale overlay on iOS Safari and Android Chrome. `dvh` units and `backdrop-filter` are the usual culprits — verify, add fallbacks.
- **Single source of truth for the HTML.** Right now the canonical demo and the deploy copy are two identical files that can silently drift. Decide on one (recommend: make `simba-demo-deploy/index.html` the only edited file) and delete or alias the other. A demo that's live-different from the one rehearsed is a real risk.
- **Reset / replay integrity.** Confirm `resetAll()` and `setMode()` fully clear chat, panel, ledger, finale and counters in every order of operations (play → pause → toggle → reset → replay). Any stale state on stage is visible to leadership.
- **Free-chat regex audit.** The intent list is order-sensitive and greedy. Walk every chip and a list of likely typed phrases; make sure the *right* intent wins (e.g. "halaal" before generic chicken, "in-pack code" before "pack"). Document the intended order in a comment block.

### P1 — strongly recommended
- **Accessibility.** Add `aria-label`s to icon-only buttons (send, mode toggle, mobile intel), ensure focus states, and check colour contrast on muted greys against the dark background. Leadership decks increasingly get screened for this.
- **Reduced-motion.** Respect `prefers-reduced-motion` for the blink/pop animations and the ticker — some viewers are motion-sensitive and the constant blinking can read as cheap on a big screen.
- **Content accuracy lock.** Every figure on screen (72% menu abandonment, 40–60% vs 5–15% redemption, R0.15–R1.80 templates, "11 official languages", "100,000 opted-in", till-slip prices) must be defensible. Cross-check against the strategic note and mark any number that is illustrative. One challenged stat in the room undermines all of them.
- **Presenter-mode robustness.** Make presenter notes legible at projector distance; consider a keyboard shortcut (e.g. space = play/pause, → = next chapter) for driving it hands-free.
- **Copy proofing.** Proof all isiZulu/Afrikaans/slang lines with a native speaker — the multilingual fluency is the whole point, so an error here is expensive. Confirm "Mzansi", "gogo", "chommies", code-switching all read as natural, not caricature.

### P2 — nice to have
- **Theming via config.** The story is already data-driven; lift the hard-coded brand strings, links (`simba.co.za/b/…`), prices and stat numbers into a single `CONFIG` object at the top so non-developers can retune for another brand/LTO without hunting through code.
- **Light refactor.** Optionally split the one file into `index.html` + `app.js` + `styles.css` for maintainability — **but only if** a tiny build step re-inlines them for the single-file deploy. Do not lose the "opens offline, zero deps" property.
- **Analytics (demo-side).** A privacy-safe counter of which chips/intents get used in the room would tell us what resonates. Optional and must not call out to anything during a live pitch.
- **Save/share state.** A "copy current view" or screenshot-friendly mode for follow-up emails.

## A6. Things NOT to break (messaging integrity)

These are not cosmetic — they are why the demo is *safe* to show and *credible*:

- **Everything is fictional and self-contained.** No real consumer data, no live system. The footer says so. Keep it true.
- **The AI discloses itself up front** (chapter 1's first brand message). This is both a compliance point and a trust point — never remove it.
- **Never name a competitor/vendor.** The "today's bot" mode is explicitly generic ("a perfectly competent 2019 bot"). Presenter note at line ~994 says *do not name any vendor.* Keep it that way.
- **Retailer-coupon logic.** The demo repeatedly makes the point that coupons are *never* cross-redeemable and are matched to the consumer's retailer. That's a real Berelo differentiator — don't "simplify" it away.
- **POPIA framing.** Opt-in, easy opt-out, "I only keep what helps me reward you, delete anytime." Keep this language in every data-capture beat.
- **The "compounding asset" through-line.** The single most important idea: a campaign ends, an owned channel compounds. Don't let polish dilute the finale's "the relationship changes owner, to Simba" payoff.

---

# PART B — BUILD THE LIVE BOT (Phase 2 pilot)

This section briefs the real, buildable WhatsApp channel the demo represents. Scope it as a **closed pilot** first: a handful of approved numbers, core flows only, on Meta's official platform — a controlled proof, not a public launch. Target **4–6 weeks** to a working closed pilot.

## B1. Target architecture (five proven pieces)

1. **The WhatsApp number.** Official WhatsApp Business Platform number with a verified green-tick business profile. Access via Meta's Cloud API directly, or — recommended for SA — through a **Business Solution Provider (BSP)** to speed up provisioning, billing and support.
2. **The brain.** An LLM (e.g. Claude) for natural language across all 11 SA official languages, with **strict brand guardrails**, a fixed persona, and **"known-answer" handling** for compliance-sensitive facts (halaal, nutrition, promo rules) so it never improvises on those. Compliance answers are served from fixed, approved responses — the model routes to them, it does not generate them.
3. **The OCR + rewards engine.** Cloud OCR reads till slips; rewards logic matches the coupon to the consumer's retailer, runs fraud checks, and issues digital value. **This is Berelo's existing core** — integrate, don't rebuild.
4. **The data layer.** A POPIA-compliant consumer profile store: consent, language, votes, occasions, purchases, referrals, segment. This *is* the live version of the demo's intelligence panel and the "owned asset."
5. **Attribution + dashboard.** Per-consumer trackable links, squad leaderboards, and a real-time dashboard for the Simba team.

None of this is experimental — it's the standard architecture behind serious WhatsApp programmes. The differentiator is the intelligence layer and proven 40–60% redemption, not the plumbing.

## B2. Demo mechanic → live build (with honest caveats)

| Demo mechanic | Live implementation | Caveat |
|---|---|---|
| Natural-language, multilingual chat | LLM with persona + guardrails; language auto-detect & code-switch | None — core capability |
| AI self-disclosure | Static opening template / session intro | None — required |
| Flavour vote + national ticker | Interactive message buttons → tally in data layer; aggregate counter | "Live national counter" UI is a brand surface, not a Meta feature — build it |
| Chicken-personality game | Quick-reply flow → profile fields | None |
| Till-slip OCR + full basket | Cloud OCR on user-sent image; parse store/time/line items | **Deep retailer-system basket integration is harder** — OCR of the slip image is reliable; live POS integration is a separate, larger scope. Position accurately. |
| Unique in-pack codes (spaza/TT) | Code validation against issued batch | None — standard |
| Retailer-matched coupons | Rewards engine issues retailer-correct digital coupon | None — Berelo core |
| Sixty60 / PnP asap! basket push | Deep link to retailer app/basket | One-tap pre-filled basket depends on retailer's deep-link support; otherwise link to product/app |
| WhatsApp Status / social sharing | Generate shareable card + per-consumer trackable link | **Auto-posting to a user's Status/social cannot be done for them** — we hand them a one-tap share kit; they post. Don't promise auto-post. |
| Leaderboard / squad attribution | Trackable links + points ledger in data layer | None |
| Care queries → R&D signal | LLM answer (known-answer for compliance) + tagged insight log | None |
| Proactive "Friday movie-night" push | Approved marketing/utility template, consent-gated | Paid per template; consent + opt-out required |

**Bottom line:** everything in the demo is buildable. The only two items needing honest framing are deep retailer-basket integration and auto-posting to social — flagged above so we never over-promise.

## B3. Cost model (so the build respects it)

Since 1 July 2025 Meta bills **per message**, in four categories:

- **Service messages (free):** any reply within 24h of a consumer messaging you. Most of Simba Connect's activity — votes, games, questions, slip uploads — is **consumer-initiated and therefore free**.
- **Marketing / utility / authentication templates (paid):** proactive outbound. In SA, marketing templates run roughly **R0.15–R1.80** each (utility cheaper), plus a small BSP markup; charged only on delivery.

**Design implication:** keep the engaging loop consumer-initiated (free); spend mainly to re-engage — which is exactly where the attribution and value sit. Architect flows to maximise the 24-hour service window.

*(Figures are directional, to confirm at scoping. Source: Meta WhatsApp Business Platform pricing; GrowthPulse Media, WhatsApp Marketing Costs South Africa 2026.)*

## B4. Compliance (build it in from message one)

- **POPIA:** explicit opt-in before any marketing; clear purpose; easy opt-out; secure storage. Treat as a feature, not a checkbox — it's a selling point (enterprise-grade, not a scraped list).
- **Meta WhatsApp Business Policy:** verified business; no prohibited content; template pre-approval for all outbound.
- **SA Consumer Protection Act (promotional competitions):** the vote, challenge and any prize need published, CPA-compliant terms. Skill/UGC mechanics are low-risk; **pure-chance mechanics need more care** — steer game design toward skill/UGC.
- **AI disclosure:** the channel declares itself an AI assistant up front and routes compliance-sensitive answers through fixed, approved responses.

## B5. Suggested pilot scope & sequence

1. Provision number + green-tick via BSP; stand up webhook + message handler.
2. Implement core flows: opt-in → flavour vote → OCR slip → retailer-matched reward → share link → one proactive push.
3. Wire the data layer + a basic real-time dashboard (the live "What Simba sees").
4. Integrate Berelo's OCR/rewards core.
5. Closed test with a handful of approved numbers (leadership included).
6. Harden guardrails, compliance copy, opt-out, and template approvals before any widening.

## B6. Open questions for scoping (please confirm with Berelo)

- BSP choice (SA provider vs Meta Cloud API direct)?
- LLM provider, hosting region (data residency for POPIA), and guardrail framework?
- Which retailers for deep-link baskets in the pilot, and what deep-link support do they actually offer?
- Reward fulfilment: which digital coupon rails, and fraud rules?
- Dashboard: build new vs extend an existing Berelo dashboard?
- Exact pilot allowlist size and success metrics (redemption %, opt-in %, completion rates).

---

## Appendix — quick start for the developer

1. Open `simba-demo-deploy/index.html` in a browser; click **▶ Play guided demo**, toggle **Today's bot**, then type freely. That's the whole product in 4 minutes.
2. Read the `script` array (one object per chapter) to see the narrative; read the `intents` array to see free-chat behaviour. Those two arrays are 90% of the content.
3. Start the polish backlog at **P0**. Test mobile on real hardware first.
4. For Part B, pair this doc with *"Simba Connect — Taking the Demo Live"* and book the scoping session before committing to a stack.
