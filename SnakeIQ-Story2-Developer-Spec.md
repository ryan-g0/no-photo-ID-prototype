# SnakeIQ — Story #2 "Identify a snake without a photo"
## Developer Specification (v1 draft)

**Purpose.** Build the guided, no-photo snake-identification flow as a reusable engine that drives two channels (WhatsApp bot, native app) from one set of rules. This document specifies the system so that **the rules decide the flow while navigation stays a dumb renderer**.

**Companion reference implementation.** `snake-id.html` (the working prototype) is the executable version of everything below: its `<script>` is the rules engine + contract, its dialog functions are one worked navigation renderer, and its `SPECIES`/`QUESTIONS` objects are the content data. Treat this doc as the contract and the prototype as the behaviour reference.

---

## 1. Architecture — three layers + content

The separation you want is achieved by splitting into three specs plus a data file, connected only by a small contract. **No layer reaches across the seam.**

| Layer | Owns | Must NOT contain |
|---|---|---|
| **Rules engine** (headless) | The decision of *what happens next*: scoring, question selection, stopping, branch gates, syndrome inference, safety overrides | Any UI, any channel-specific text/layout |
| **Contract** (the seam) | The vocabulary of *directives* (engine→nav) and *events* (nav→engine) | Logic of any kind — it's just data shapes |
| **Navigation** (per channel) | Rendering a directive to components; capturing input as events | Any snake knowledge; any flow decision |
| **Content data** (a file) | The species × trait matrix, priors, question definitions, copy | Code |

**Resolving "the rules determine the flow":** navigation does not hold a screen graph. The engine is a decision function `next(state, event) → { state, directives[] }`. Navigation is a loop: *render the directive → capture an event → hand it back → render the next directive*. The flow is therefore computed by the rules; navigation never knows what comes next. This is the entire mechanism that keeps the two independent.

```
        answer/escape/restart (Event)
   ┌───────────────────────────────────────┐
   │                                        ▼
[ Navigation ] ── renders ◄── directive ── [ Rules Engine ] ◄── reads ── [ Content data ]
 (WhatsApp / app)                          (pure, headless)              (matrix, priors, copy)
```

---

## 2. The contract (engine ⇄ navigation)

### 2.1 Directives (engine → navigation)

Each turn the engine returns an ordered list of directives (an escalation banner may precede an `ASK`). All payloads are channel-agnostic.

```jsonc
// ASK — present one question
{ "type": "ASK",
  "question": {
    "id": "colour",
    "prompt": "Main colour?",
    "header": null,                 // optional short title
    "footer": "SnakeIQ",            // optional
    "options": [ { "id": "green", "label": "Green", "icon": "🟢" }, ... ],
    "escapeAllowed": true           // whether a "None / not sure" affordance is offered
  } }

// RESULTS — ranked candidate species
{ "type": "RESULTS",
  "band": "High|Medium|Low",        // overall confidence band
  "candidates": [
    { "speciesId": "puff", "common": "Puff Adder", "latin": "Bitis arietans",
      "matchPct": 83, "venom": "danger" }, ...     // 3–5, ordered desc
  ],
  "banner": { "level": "info|warn|emergency",
              "syndrome": "CYTOTOXIC|NEUROTOXIC|HAEMOTOXIC|null",
              "text": "..." },
  "escapeAllowed": true }

// ESCALATE — act-now safety interrupt (can be emitted alongside another directive)
{ "type": "ESCALATE", "level": "critical", "text": "..." }

// SPIT_RESULT — venom-in-eyes branch (distinct management, no antivenom)
{ "type": "SPIT_RESULT",
  "candidates": [ {speciesId,common,latin,matchPct,venom}, ... ],  // spitters only, renormalised
  "banner": { "level":"emergency", "text":"Irrigate eyes 15–20 min; go to hospital; no antivenom for eye exposure." } }

// HANDOFF — leave this flow for an existing flow
{ "type": "HANDOFF", "target": "snake_catcher|first_aid|find_antivenom" }

// SAFE_DEFAULT — escape hatch outcome
{ "type": "SAFE_DEFAULT", "text": "Treat as potentially venomous ..." }
```

### 2.2 Events (navigation → engine)

```jsonc
{ "type": "ANSWER", "questionId": "colour", "optionId": "green" }
{ "type": "ESCAPE" }                        // user tapped "None / not sure"
{ "type": "SELECT_CANDIDATE", "speciesId": "puff" }   // "This is it"
{ "type": "REQUEST_HANDOFF", "target": "first_aid" }
{ "type": "RESTART" }
```

### 2.3 Engine API

```
initSession(context) -> State
step(State, Event)    -> { state: State, directives: Directive[] }
```
`context` carries region (e.g. `"ZA-GP"`), and optionally locale, time/temperature (see §7). `State` is serialisable (answers so far + phase). The engine is **pure and synchronous** — same inputs, same outputs; no I/O.

---

## 3. Rules engine

### 3.1 Data model (content, see §5 for schema)

- **Species** carry a `prior` (regional encounter likelihood 0–1), a `venom` class, a `syndrome`, and a `traits` map.
- **Trait cells** are graded likelihoods `P(answer | species)`, encoded as `ALWAYS/USUALLY/SOMETIMES/RARELY` → `1.0 / 0.7 / 0.4 / 0.15`. Unspecified cells default to a **floor** so nothing is ever impossible.
- `FLOOR = 0.05`.

### 3.2 Scoring (Bayesian)

For each species `s`:
```
score(s) = prior(s) × Π over answered questions q  condProb(s, q, answer_q)
condProb(s,q,a) = L(s,q,a) / Σ over options o of q  L(s,q,o)      // per-question normalisation
L(s,q,a) = max(cell(s,q,a), FLOOR)
posterior(s) = score(s) / Σ score                                 // normalise across species
```
Ranking = posterior descending. Nothing is eliminated → there is always a ranked list (never a dead end).

### 3.3 Adaptive question selection (information gain)

Over the current posterior `P`:
```
infoGain(q) = H(P) − Σ over options a of q  P(a)·H(P | a)
   where P(a) = Σ_s P(s)·condProb(s,q,a),   H = Shannon entropy (bits)
Pick the eligible question with the highest infoGain.
```
Eligibility: exclude already-asked questions; exclude questions **gated** off (see §3.5); exclude the two hand-authored triage questions (`spitbite`, `redflags`) from the picker — they are placed deterministically, not by gain.

### 3.4 Stopping rule (hybrid)

Stop asking and emit `RESULTS` when **any** of:
- `askedCount ≥ MAX_Q` (`MAX_Q = 6`), or
- `topPosterior ≥ CONF_STOP` **and** `top − second > 0.18` (`CONF_STOP = 0.62`), or
- best available `infoGain < IG_MIN` (`IG_MIN = 0.04`).

Confidence band for display: `High ≥ 0.60`, `Medium ≥ 0.38`, else `Low`.

### 3.5 Branch gates (the flow skeleton)

The only fixed part of the flow. Everything between the gates is emergent from §3.3.

1. **Front door — `spitbite`** is asked first, always: *bite / spit-in-eyes / just-saw*.
2. `spit` → emit `SPIT_RESULT` and stop (irrigation branch; see §3.7).
3. `bite` → ask `redflags` once (act-now escalation, §3.6); then run the info-gain picker; the three **symptom questions** (`localsigns`, `neuro`, `bleeding`) are *gated on `spitbite === "bite"`* and are **guaranteed to run** before `RESULTS` (append any unanswered symptom question after the picker stops) — but they are **salience-ordered, not forced first**.
4. `just-saw` → the info-gain picker only; symptom questions stay gated off.

### 3.6 Red-flag escalation

`redflags` = "trouble breathing, fainting/collapse, or heavy bleeding right now?". On `yes`, emit `ESCALATE(critical)` immediately (call an ambulance / go now) and continue. This is triage, not identification — it never changes the species logic.

### 3.7 Syndrome inference (for the results banner on a bite)

```
neuro = yes            -> NEUROTOXIC
else bleeding = yes    -> HAEMOTOXIC
else localsigns ∈ {severe, blister} -> CYTOTOXIC
else                   -> null (unclear/early)
```
When a syndrome is inferred, `RESULTS.banner` **leads with the venom type** and its type-specific first-aid; when null, it shows the generic "treat as venomous, watch 24–48 h" banner.

---

## 4. Safety invariants (MUST — non-negotiable, engine-guaranteed)

These hold **regardless of navigation** and each has a test in §6.

1. **Never a dead end.** Every input state yields a ranked `RESULTS` (or a branch exit). The `FLOOR` guarantees no species reaches zero.
2. **Asymmetric safety.** While any `danger`-class species remains plausible (posterior > 0.05 within the top candidates) OR band is `Low`, the results banner MUST advise "treat as venomous" — regardless of whether the top candidate is harmless.
3. **No de-escalation after a bite.** Once `spitbite === "bite"`, `RESULTS` MUST always carry an emergency banner; a reassuring answer may raise a species' confidence but MUST NOT produce a "you're fine / harmless" outcome.
4. **Absence is not reassurance.** "No symptoms yet" (`neuro=no`, `bleeding=no`) MUST be non-diagnostic — it must not raise harmless species. (Encode symptom "no" as flat across species.)
5. **Spit ≠ bite.** `spit` routes to eye irrigation + hospital + **no antivenom**, and never enters the antivenom/first-aid-for-bite path.
6. **Escape always reachable.** An escape affordance ("None / not sure") is present on every ID question and on results, routing to `SAFE_DEFAULT` + catcher handoff.
7. **ID never gates care.** The user can always reach first-aid / antivenom / catcher without a confirmed species.

---

## 5. Content data schema (edit without touching engine code)

Ship as JSON/YAML, versioned separately, editable by a herpetologist.

```jsonc
{
  "region": "ZA-GP",
  "weights": { "ALWAYS": 1.0, "USUALLY": 0.7, "SOMETIMES": 0.4, "RARELY": 0.15, "FLOOR": 0.05 },
  "questions": [
    { "id": "colour", "kind": "trait", "picker": true,
      "prompt": "Main colour?",
      "options": [ {"id":"green","label":"Green","icon":"🟢"}, ... ] },
    { "id": "localsigns", "kind": "symptom", "picker": true, "gate": "bite", ... },
    { "id": "spitbite", "kind": "triage", "picker": false, ... },
    { "id": "redflags", "kind": "triage", "picker": false, ... }
  ],
  "species": [
    { "id": "puff", "common": "Puff Adder", "latin": "Bitis arietans",
      "venom": "danger", "syndrome": "cytotoxic", "prior": 0.90,
      "traits": {
        "build":    { "thick": "ALWAYS", "average": "RARELY" },
        "colour":   { "brown": "USUALLY", "yellow": "SOMETIMES", "grey": "SOMETIMES" },
        "behaviour":{ "still": "ALWAYS", "fast": "RARELY" },
        "marks":    { "two": "USUALLY" },
        "localsigns": { "severe":"USUALLY", "blister":"USUALLY", "mild":"RARELY" }
        // ... one entry per discriminating question
      } }
    // ... one object per species in the regional set
  ]
}
```

**Discriminators used (Gauteng set):** `build, colour, pattern, behaviour, size, tell` (morphology); `activity, timeofday, marks` (ecology/circumstance); `localsigns, neuro, bleeding` (envenomation syndrome, gated on bite); plus triage `spitbite, redflags`.

**⚠ Data provenance (must be resolved before real use).** The current matrix weights and priors are a **hand-built reconstruction** informed by ASI and African Reptiles & Venom material — *not* a digitisation of a validated dataset, and Gauteng's ~14 species here are a representative subset of 30+. Before production: digitise from authoritative sources (Marais's *A Complete Guide to the Snakes of Southern Africa* key-features, ASI Easy-ID) and have a herpetologist/toxinologist review every cell.

---

## 6. Acceptance tests (given / when / then)

Express rules as behaviour, not prose. These are derived from verified prototype runs.

1. **Fast standout tell.** Given region ZA-GP; when `tell = blackmouth`; then Black Mamba ranks #1 despite its low prior.
2. **Hood collapse.** When `behaviour = hood`; then the top candidates are cobras + rinkhals (all `danger`); danger banner shown.
3. **Mimic safety.** When `build = thick, pattern = chevron`; then Puff Adder is top **and** the harmless Rhombic Egg-eater remains present **and** the "treat as venomous" banner still shows.
4. **Green-in-a-tree asymmetry.** When `colour = green, behaviour = tree`; then Spotted Bush Snake (harmless) is top **and** Boomslang (danger) stays plausible **and** the danger banner stays on.
5. **No-symptom non-reassurance.** When `spitbite = bite, neuro = no, bleeding = no`; then no harmless species is crowned (ranking stays ~flat) **and** the emergency banner still shows.
6. **Syndrome-led banner.** When `spitbite = bite, neuro = yes`; then banner leads with `NEUROTOXIC` + its first-aid; when instead no syndrome is inferred, banner is the generic emergency/watch text.
7. **Spit branch.** When `spitbite = spit`; then only Rinkhals + Mozambique Spitting Cobra are shown, **renormalised between them** (≈63% / 37% by prior), with the eye-irrigation/no-antivenom banner; the bite path is not entered.
8. **Question ordering (bite).** When `spitbite = bite`; then order is: `redflags` → sighting/ecology questions (by info gain) → the three symptom questions (guaranteed) → `RESULTS`.
9. **Cap.** No path asks more than `MAX_Q` picker questions before results (triage + guaranteed symptom questions excepted).

---

## 7. Navigation spec — WhatsApp channel

Navigation maps directive → components and posts back events. **No flow logic here.**

| Directive | WhatsApp rendering | Constraint |
|---|---|---|
| `ASK`, ≤3 options (+escape ≤3 total) | Interactive **reply-button** message | ≤3 buttons; title ≤20 chars |
| `ASK`, 4–10 options | Interactive **list** message (button opens a row sheet) | ≤10 rows; row title ≤24, description ≤72; list button label ≤20 |
| `RESULTS` | Carousel **template** (up to 10 cards) OR a list message with "% · venom" per row (see note) | body ≤1024; footer ≤60 |
| `SPIT_RESULT` | Text banner + 2 cards | as above |
| `ESCALATE` | Plain text message, sent first | — |
| `HANDOFF` / `SAFE_DEFAULT` | Reply buttons (Find catcher / First aid / Restart) | ≤3 buttons |

**Rendering rules to enforce (a "WhatsApp lint"):** reject any message exceeding the caps above; the engine's `ASK` payloads must already be authored to fit. Reply-button messages carry the reply-arrow glyph; list messages use a per-question trigger label ("Pick the colour"). Match platform chrome to the target OS.

**Results note.** A horizontally-scrolling carousel IS available on WhatsApp (carousel template / multi-product), but as a *template* type — confirm your BSP supports conversational carousel; otherwise render `RESULTS` as a list message or sequential image cards. This is a per-channel choice and does not affect the engine.

### 7.1 Native app channel (summary)
Same directives, rendered with the Black Mamba R3 design system: `ASK` → trait-tile screens with a live "X likely matches" counter; `RESULTS` → the R3 species match cards; hands off to the existing Species Info / First Aid / Find Antivenom screens.

---

## 8. Non-functional requirements

- **Offline-first / on-device.** The matrix is small; the engine and content ship in the client and run with no network (consistent with the app's offline principle). No round-trip to decide the next question.
- **Region-parameterised.** `context.region` selects the species set + priors. Adding a region = adding a data file, no code change.
- **Localisation.** All user-facing strings live in the content/copy layer, keyed for translation; the engine references ids only.
- **No PII in the flow.** Do not put location/identity in URLs or logs; keep answers in session state.
- **Determinism & telemetry.** `step()` is pure; log `{state, event, directives}` per turn for analytics and for reproducing any reported result.

---

## 9. Out of scope / open questions (carry from the brief)

- Confidence display: per-card % vs High/Med/Low band — prototype shows both; decide per channel.
- Whether to promote trait-gathering to **WhatsApp Flows** (single-screen multi-field) to reduce list-drawer taps.
- Snake-catcher / expert-ID data availability per region.
- Photo-assisted entry ("pick from local snakes", blurry-photo path) — separate module sharing the same engine.
- **Clinical sign-off of the syndrome layer and the matrix data before any real-world deployment.**
