## Version 0.4.3
Date: 2026-09-25

### Milestone
Implemented **Context Optimization & Bounded Runtime Prompts** to stop ordinary turn costs from growing with campaign length. No Cloudflare Worker changes are required.

### Why this iteration was necessary
The turn-19 Dinotopia debug export showed routine Sol calls growing to roughly 12k–14k input tokens because adjudication and narration repeatedly received broad campaign state plus long transcript/history slices. The database already stores authoritative memory, so resending most of that state every call was redundant and would make later turns progressively more expensive.

### Core architecture change
Runtime prompting now follows the rule:

> **The database remembers. The AI receives only the state needed for the current job.**

The full save remains authoritative and is still exported for debugging, but ordinary AI calls now use phase-specific compact context builders.

### Phase-specific context builders
Added bounded context packages for:
- adjudication
- outcome narration
- Adventure Director
- hints
- legacy scenario bootstrap

Adjudication receives a compact campaign digest, current scene, relevant player capabilities/inventory/knowledge, relevant facts/NPCs, open threads, a small scenario-anchor slice, and only the most recent dialogue.

Outcome narration receives an even smaller context focused on the current scene, fixed action result, causally relevant NPCs/facts/threads/anchors, and a few recent dialogue messages. It no longer receives the entire adjudication object or the broad runtime state.

The Adventure Director receives compact anchor/NPC/thread/fact state plus a short recent-event digest instead of the entire world and event history.

Hints continue to receive player-known information only, but that package is now bounded as well.

### Fixed context limits
Added `CONTEXT_POLICY` with bounded runtime slices. Current v0.4.3 defaults include:
- adjudication dialogue: last 4 player/referee messages
- outcome dialogue: last 3
- hints dialogue: last 6
- facts: up to 8
- player knowledge: up to 8
- known places: up to 4
- relevant NPCs: up to 4 for ordinary context
- scenario anchors: up to 4 for ordinary context
- threads: up to 6
- recent Director events: up to 6

Selections favor present/named/relevant entities and recent state rather than blindly taking the entire campaign database.

### Relevance selection
Added lightweight local lexical relevance selection. When a collection exceeds its context cap, the client scores entries against the current player action while retaining a mild recency preference. Present NPCs and explicitly named NPCs are always prioritized.

This is local JavaScript; it does not require an extra AI call.

### Reduced repeated prose/state
- Ordinary adjudication uses a lean runtime campaign digest rather than the full campaign object.
- Player concept/background are omitted from routine turn prompts unless a broader context genuinely needs them.
- Operational JSON is serialized compactly rather than pretty-printed.
- Outcome narration receives a compact adjudication projection instead of the entire structured adjudication response.
- Known-place lists, hidden NPC state, facts, and scenario anchors are capped by job.

### Adventure Director cost control
The v0.4.2 Director could be invoked after nearly every committed turn in a source-guided campaign simply because pending anchors existed. v0.4.3 now calls the Director when:
- a major/supporting anchor is active or eligible;
- the campaign has gone two committed turns without meaningful development;
- a pending source anchor exists and the just-completed action meaningfully changed state; or
- source progression pressure has begun to build during movement/observation/conversation/rest.

This preserves world motion without paying for a separate Director call after every trivial action.

### Request-size diagnostics
Every gateway call now records local request metrics:
- instruction characters
- dynamic input characters
- Structured Output schema characters
- approximate pre-API input-token count
- context-policy version

Debug export now includes `request_size_summary` over recent calls and the active `CONTEXT_POLICY`. This makes future cost regressions visible directly in the debug file.

### Validation fixture
Using the existing turn-19 Dinotopia debug state as a fixed test fixture, the serialized authoritative state/context sizes changed approximately as follows:

- previous broad `currentContext()` payload: **34,394 characters (~8,599 rough tokens)**
- v0.4.3 adjudication context: **10,803 characters (~2,701 rough tokens)**
- v0.4.3 outcome context: **8,261 characters (~2,066 rough tokens)**
- v0.4.3 Director context: **8,167 characters (~2,042 rough tokens)**
- v0.4.3 hint context: **7,889 characters (~1,973 rough tokens)**

These figures measure the state/context JSON only; actual API input also includes stable instructions and the Structured Output schema. The important change is that context size is now bounded instead of growing with the complete transcript/event history.

### Additional cleanup
- Removed a duplicated deterministic-success branch in the likelihood helper.
- Structured Output schema names now use the v0.4.3 identifier.
- All visible/runtime version labels and prompts report v0.4.3.

### Validation performed
- Embedded JavaScript syntax checked successfully with Node.js.
- Context builders executed against the turn-19 Dinotopia debug state without errors.
- Confirmed ordinary context packages are materially smaller than the former broad context payload.
- Confirmed context collections have fixed caps so transcript/history growth does not linearly enlarge ordinary prompts.
- Confirmed the full authoritative save remains unchanged and is still available in Save/Debug exports.
- Confirmed no Cloudflare Worker change is required.

### Recommended validation test
1. Upload v0.4.3 and start or continue a campaign.
2. Play 10–20 turns.
3. Export Debug.
4. Inspect `diagnostic_summary.request_size_summary` and the `request_metrics` attached to recent adjudication/narration/Director entries.
5. Verify request sizes remain in roughly the same range instead of climbing every turn.
6. Compare OpenAI usage after a similar-length session to the v0.4.2 turn-19 session.
7. If a model ever appears to forget a relevant established fact, use the debug export to identify which context selector omitted it before increasing any caps globally.

---

## Version 0.4.2
Date: 2026-09-25

### Milestone
Implemented **Scenario Progression & Canonical Fidelity** to prevent campaigns from stalling in repetitive local exploration. No Cloudflare Worker changes are required.

### Why this iteration was necessary
The turn-19 Dinotopia debug export showed that the runtime could resolve movement/search correctly but had little machinery for advancing the world after those actions. The campaign had only Arthur as an NPC, broad survival/exploration threads, no scenario anchors, and no Adventure Director. It could therefore keep producing small local details without ever bringing important people, events, or transitions into play.

The same debug export also showed a source-mode mismatch: the compiled campaign state could report a different Source Mode from the user's creation selection. v0.4.2 makes the player's selected Source Mode authoritative.

### Save Schema v4
Save schema increased from 3 to **4**. New state includes:
- `creation_request` — preserves the exact campaign-creation selections/notes for future scenario repair or recompilation.
- `scenario` — the hidden scenario framework.
- `scenario.anchors[]` — structured events, encounters, discoveries, or transitions.
- `scenario.director` — runtime pacing/progression bookkeeping.

Existing schema-v1/v2/v3 saves migrate automatically. Old pending actions are cleared during migration because their stored adjudication objects predate the new update contract.

### Scenario Anchors
Campaign compilation now creates roughly 3–8 scenario anchors for a normal opening. Each anchor records:
- stable ID and hidden/public label
- status: pending, eligible, active, completed, bypassed, or impossible
- importance: major, supporting, or optional
- source role: canonical, source-inspired, or original
- trigger conditions
- completion conditions
- adaptation guidance
- blocking conditions

Anchors are **not predetermined outcomes**. They define important developments the world should attempt to preserve or reach when causally compatible with player agency.

### Source-mode enforcement
The campaign compiler may no longer silently relabel the player's Source Mode. After compilation, local JavaScript force-locks `campaign.source_mode` to the exact UI selection.

Runtime semantics are now explicit:
- **Canonical** — preserve established source characters, major events, relationships, discoveries, and broad sequence whenever compatible with player agency. Adapt route/timing/location rather than dropping an event because exact staging changed.
- **Adaptive** — preserve major source anchors and important characters while allowing freer relocation, reordering, combination, and branching.
- **Inspirational** — source guides tone/themes; source events are not obligations.
- **Original** — no source-fidelity obligation beyond the explicit premise/notes.

Canonical/Adaptive anchors must never force a player choice or predetermined result.

### Near-term source NPCs
For source-guided campaigns, the compiler is now instructed to precompile important near-term source NPCs even when they are not yet visible. Such NPCs may begin with status `elsewhere`. This prevents an important character from being absent simply because the runtime has no entity record for them.

### Adventure Director
Added a runtime **Adventure Director** using GPT-6 Sol / Low. It runs after a committed player action when:
- a major/supporting anchor is active or eligible;
- a source-guided campaign still has unresolved non-optional anchors that need eligibility evaluation;
- the campaign has gone two committed turns without meaningful development; or
- movement/travel produced no meaningful development.

The Director may:
- advance an eligible scenario anchor;
- allow an autonomous NPC to act;
- create a scene transition;
- reconcile stale threads/facts;
- record a remote development when causally appropriate.

The Director does **not** resolve character task uncertainty and never rolls probability. It operates on the same turn after the player's result is already committed. A Director failure is logged but cannot roll back the player's completed action.

### World-can-come-to-player rule
The runtime now explicitly rejects the idea that the player must discover a magic command or exact route to reach the next meaningful event. If a scenario anchor is eligible and an NPC/event can plausibly reach the protagonist, the world may move toward the player.

Broad intentions such as “follow the coast until something worth investigating happens” should also be summarized through uneventful stretches instead of requiring repeated walking commands.

### State and thread reconciliation
The state-update contract now supports:
- changing an existing world fact
- removing an obsolete world fact
- changing thread status
- changing scenario-anchor status

Outcome narration and the Adventure Director are instructed to remove/update stale facts and close/fail/dormant threads when reality changes, instead of leaving outdated priorities permanently active.

### Non-probability assessment fix
Consequential actions that do not actually have a success probability now use the internal assessment band `not_applicable` rather than allowing null probability to be interpreted as an extremely low chance. The player-facing referee should discuss the known consequence/uncertainty instead of saying the action is nearly impossible.

### Legacy scenario bootstrap
A migrated pre-v0.4.2 campaign receives an empty scenario framework marked `needs_bootstrap=true`. Before the next player action, the client performs a **one-time GPT-6 Astra / High scenario bootstrap** to create anchors from the existing authoritative campaign state without rewriting established history. After that, normal runtime progression uses Sol / Low.

Important limitation: legacy bootstrap respects the Source Mode already stored in that save. If an older campaign was previously compiled with the wrong Source Mode, the migration does not guess what the player originally selected. For source-fidelity testing, starting a new campaign with Canonical or Adaptive selected is the cleanest validation.

### Debugging additions
Debug export now reports:
- scenario mode
- scenario anchor count
- active/eligible anchor IDs
- whether scenario bootstrap is still required
- Director stagnation pressure (`turns_since_development`)
- Adventure Director calls/no-ops/errors
- scenario anchor status changes in update summaries

### AI routing impact
- Campaign compile: GPT-6 Astra / High (unchanged)
- One-time legacy scenario bootstrap: GPT-6 Astra / High
- Runtime Adventure Director: GPT-6 Sol / Low
- Existing adjudication/narration/hint routing remains unchanged

### Validation performed
- Embedded JavaScript syntax checked successfully with Node.js.
- Confirmed all runtime version labels/prompts report v0.4.2.
- Confirmed save schema reports v4 and v1/v2/v3 are accepted for migration.
- Confirmed `campaign.source_mode` is locally locked to the user's creation selection.
- Confirmed scenario framework and Director schemas use strict structured output.
- Confirmed old pending actions are discarded during schema migration to avoid incompatible v0.4.1 adjudication payloads.
- Confirmed Adventure Director errors are isolated and do not undo a committed player action.
- Confirmed no Cloudflare Worker change is required.

### Recommended validation test
1. Start a fresh Dinotopia campaign with **Canonical** (or Adaptive, if desired) explicitly selected.
2. Ask for close adherence to *A Land Apart From Time* in Source / campaign notes.
3. Spend a few actions on the beach without deliberately guessing the book's next scene.
4. Verify the world introduces meaningful forward development rather than requiring repeated beach-walking/search commands.
5. Confirm future scenario anchors are not exposed as spoilers in the player UI/hints.
6. Confirm completed survival facts/threads are updated or closed rather than remaining permanently active.
7. Export Debug after 5–10 turns and inspect scenario anchors/Director records if progression still feels wrong.

---

## Version 0.4.1
Date: 2026-09-25

### Milestone
Added a dedicated troubleshooting/debug export and an optional player-facing **Hints** system. No Cloudflare Worker changes are required.

### Debug Export v2
The existing **Export Debug** control has been expanded into a troubleshooting package intended to be shared when a campaign behaves incorrectly.

The export now includes:
- complete authoritative save state, including hidden referee/NPC data
- current app, save-schema, and Probability Builder versions
- current turn and state revision
- pending-action summary
- hint preferences
- entity/log counts
- current AI routing profiles
- recent debug records and recent event records
- recent system/error transcript messages
- Probability Builder calibration report and self-test
- browser/runtime information useful for reproducing client issues
- the configured Worker URL
- a live `/health` snapshot from the Cloudflare gateway when reachable
- whether a game token is present and whether it is stored persistently

Security rule: the debug file **never exports the GAME_TOKEN itself or the OpenAI API key**. It does contain hidden campaign information and therefore carries an explicit spoiler warning before export.

Debug filenames now include the turn number and an ISO timestamp so multiple reports from the same turn do not overwrite one another accidentally.

### Optional Hints System
Added a **Hints** card to the campaign sidebar. Hints are disabled by default and may be switched on or off at any time without affecting campaign state or advancing time.

When enabled, the player can choose one of three assistance levels:
- **Gentle nudge** — points toward a relevant known detail or unresolved situation without directing the player to a specific answer.
- **Possible options** — offers several plausible actions or questions based on current known information, without ranking one as the correct choice.
- **Direct suggestion** — gives one or two concrete ways to get unstuck and briefly explains why they are reasonable.

Hints use **GPT-6 Sol / Low** and are treated as out-of-character assistance rather than in-world actions.

### Hidden-information protection
The hint model does not receive the full authoritative state. A dedicated player-facing context builder strips hidden information before the hint request is created. Hint context includes only:
- public campaign information
- protagonist public capabilities, inventory, condition, and learned knowledge
- current/known place public descriptions
- public resources and world facts
- NPC public descriptions/status
- visible open situations
- recent player/referee conversation
- the visible assessment for a pending action, if one exists

It explicitly excludes:
- NPC hidden nature, goals, loyalties, fears, secrets, intentions, and private knowledge
- hidden world facts
- hidden place facts
- hidden threads
- objective probabilities and RNG information
- undiscovered clues or future events

Hints cannot mutate the world, advance time, trigger NPC activity, resolve uncertainty, or establish new canon. Hint messages are marked separately in the transcript and are excluded from normal referee context so suggestions do not later become mistaken for established events.

### Persistence
Hint enabled/disabled state and selected hint style are saved with the campaign. Existing schema-v3 saves migrate automatically with hints disabled and the Gentle Nudge style selected.

### Validation performed
- Embedded JavaScript syntax checked with Node.js.
- Confirmed all runtime version labels/prompts report v0.4.1.
- Existing save-schema version remains 3; no destructive migration is required.
- Existing Cloudflare Gateway v1.1 remains compatible; no Worker update is necessary.
- Hint messages are excluded from authoritative AI context.
- Debug export explicitly records token presence without serializing the token value.

### Recommended next test
1. Load an existing campaign and confirm Hints defaults to Off.
2. Turn Hints On and request each of the three hint styles.
3. Verify requesting a hint does not increment Turn or Revision.
4. In a mystery/horror test, confirm hints do not reveal a known hidden NPC secret or hidden fact.
5. Export Debug and confirm the file contains gateway health, routing, recent logs, probability diagnostics, and full save state, but no GAME_TOKEN value.
6. Send a debug export after any reproducible gameplay problem so the failure can be inspected directly.

---

# Universal RPG — Audit Log

## Version 0.4.0
Date: 2026-09-25

### Milestone
Implemented the first calibrated **Universal Probability Builder and Outcome Engine**. The browser now owns the probability math, semantic calibration, circumstance stacking limits, random sampling, and degree-of-outcome calculation. The language model classifies fictional facts into bounded categories but does not invent percentages or roll outcomes.

### Probability Builder v1.0
The local engine now separates:

```text
Capability
+ intrinsic task difficulty
+ bounded circumstances
= objective success probability
```

The calibration anchor is:

```text
Competent character + Standard task + neutral circumstances ≈ 70% success
```

Capability levels remain:

```text
untrained
novice
familiar
competent
skilled
expert
master
legendary
```

Difficulty levels remain:

```text
routine
easy
standard
difficult
formidable
extreme
```

Difficulty now explicitly means the **intrinsic demand of the task under neutral conditions**. Circumstances are no longer supposed to be baked into difficulty and then counted again as modifiers.

### Calibration anchors
With no circumstantial factors, the current v1.0 curve yields approximately:

```text
Untrained  vs Standard    30.9%
Familiar   vs Standard    57.4%
Competent  vs Standard    70.0%
Skilled    vs Standard    80.2%
Expert     vs Standard    87.5%

Competent  vs Easy        83.2%
Competent  vs Difficult   52.4%
Competent  vs Formidable  34.2%
Expert     vs Formidable  61.0%
```

Routine actions should still normally bypass probability entirely when failure is not meaningful. These percentages apply only when the Resolution Gate determines that an uncertain, consequential resolution is actually needed.

### Circumstance categories
Every situational factor must now belong to exactly one bounded category:

```text
opposition
equipment
preparation
environment
condition
assistance
time_pressure
scale
positioning
information
other
```

Factor strength remains semantic rather than numeric at the AI boundary:

```text
minor advantage / hindrance
significant advantage / hindrance
major advantage / hindrance
```

The AI identifies the category and qualitative strength. Local JavaScript translates that classification into the calibrated math.

### Anti-stacking controls
To prevent the referee from inflating probabilities by restating the same circumstance several ways, factor effects are capped both per category and across the full action. The adjudication prompt now explicitly requires one factor per distinct fictional fact and prohibits double-counting the same fact in both intrinsic difficulty and circumstances.

### Objective versus perceived probability
The two-model structure is now explicit:

```text
OBJECTIVE MODEL
Established reality, including hidden facts that physically matter
→ used for actual resolution

PERCEIVED MODEL
Only what the protagonist can reasonably know or infer
→ used for pre-commitment risk assessment
```

The player-facing likelihood is also tempered by assessment confidence. High/moderate confidence preserves the calculated semantic band. Low confidence softens the language one step toward uncertainty. Unknown confidence reports the situation as uncertain rather than pretending to precision.

### Stable likelihood bands
Internal probabilities are translated into semantic bands for referee wording:

```text
98%+      essentially certain
90–97%    very likely
75–89%    good
60–74%    favorable
40–59%    uncertain
25–39%    not good
10–24%    unlikely
2–9%      very unlikely
<2%       nearly impossible
```

These are not displayed as percentages during normal play. The referee receives the semantic band and causal factors and expresses them naturally.

### Impossible is not low probability
Scale and physical possibility are now explicitly distinguished from difficult checks. If the intended effect is impossible under established reality, adjudication must set `action_possible=false` and avoid probabilistic resolution. A handgun cannot acquire a 0.5% chance to penetrate tank armor merely because the engine has a probability floor.

### Outcome magnitude
The same cryptographically generated random sample now determines both binary success/failure and degree of outcome through a calibrated logistic performance margin. Probabilistic outcomes can be:

```text
marginal_success
solid_success
strong_success
exceptional_success

marginal_failure
failure
severe_failure
exceptional_failure
```

This replaces the earlier coarse `ordinary / strong / exceptional` success-only classification. Outcome magnitude is passed to the narrator as authoritative and may not be rerolled or reversed.

### Debugging and validation
Debug exports now include:

- Probability Builder version
- calibration anchors
- factor-to-logit mapping
- category and total factor caps
- probability floor/ceiling
- likelihood bands
- automated probability self-test result
- objective probability calculation records
- applied factor categories and capped contributions
- random sample
- logistic performance margin
- final success/failure and magnitude

The developer snapshot now reports `probability_engine: "1.0"`.

### AI adjudication guidance
The referee prompt now defines capability and difficulty semantics, requires factor categories, prohibits factor duplication, distinguishes intrinsic task demand from temporary circumstances, treats active opposition as a bounded opposition factor, and tells the AI that local JavaScript—not the model—owns probability and RNG.

### Compatibility
- Save schema remains version 3. Existing v0.3.x saves remain compatible.
- Existing pending actions lacking v0.4 factor categories fall back to the `other` category when confirmed after upgrade.
- Gateway v1.1.0 remains compatible. **No Cloudflare Worker update is required.**
- Existing GPT-6 Sol Low → Sol Medium → Astra High routing is unchanged.

### Validation performed
- Confirmed visible and internal application version strings are v0.4.0.
- Confirmed strict structured-output schema requires factor categories for new adjudications.
- Confirmed Competent vs Standard anchor resolves to 70%.
- Confirmed increasing capability raises success probability.
- Confirmed increasing intrinsic difficulty lowers success probability.
- Confirmed advantages increase and hindrances decrease probability.
- Confirmed category and total factor caps are applied locally.
- Confirmed perceived likelihood is confidence-filtered without changing the objective probability.
- Confirmed outcome magnitude is derived from the same RNG sample as success/failure.
- Confirmed debug export includes calibration and self-test data.
- JavaScript syntax validation performed after build.

### Known limitations / next step
Probability calibration is now mechanically stable enough to support domain-specific consequences, but the current engine still treats the resolved action generically. The next major implementation should be **Combat & Injury v0.5**, including the proposed weapon profile:

```text
Damage
Penetration
Range
Handling
Scale
Traits
```

and the pipeline:

```text
hit → coverage/protection → penetration → injury distribution → conditions/consequences
```

Weapon Handling should remain contextual rather than a permanent accuracy bonus.

## Version 0.3.3
Date: 2026-09-25

### Display-version hotfix
Corrected two stale hard-coded `v0.3.2` strings in `index.html`: the visible application header and the campaign-compiler instruction banner. The actual `APP_VERSION` and v0.3.3 model-routing logic were already correct. No rules, save schema, or routing behavior changed in this hotfix.

### Milestone
Refined the cost-routing architecture into a **three-tier referee pipeline** so ordinary prose and simple decisions stay inexpensive while important NPC behavior and meaningful conversations receive more reasoning depth without defaulting to GPT-6 Astra.

### Model routing

The active routing profiles in `index.html` are now:

```text
Campaign compilation             GPT-6 Astra / High
Major world generation           GPT-6 Astra / High

Routine adjudication             GPT-6 Sol / Low
Important/significant adjudication GPT-6 Sol / Medium
Exceptional adjudication         GPT-6 Astra / High

Risk assessment wording          GPT-6 Sol / Low
Routine narration                GPT-6 Sol / Low
Significant narration            GPT-6 Sol / Medium
Connection/simple utility calls  GPT-6 Sol / Low
```

Authoritative simulation remains local JavaScript. The language model does not own probability sampling, state mutation authority, inventory truth, or deterministic mechanics.

### Low → Medium → Astra referee escalation

Ordinary player input first receives a GPT-6 Sol / Low adjudication pass. That response must classify the reasoning burden as `routine`, `complex`, or `exceptional`.

`routine` is intended for:

- clarification
- risk assessment
- ordinary movement and observation
- atmosphere and description
- routine conversation
- simple NPC reactions
- straightforward actions involving only a few obvious facts

`complex` now explicitly includes situations where better reasoning materially matters:

- important NPC decisions
- persuasion and negotiation
- arguments
- deception
- relationship-changing scenes
- multiple important NPCs interacting
- hidden-information reasoning
- conflicting NPC goals or loyalties
- complicated tactical choices
- scenes where several established facts constrain what an NPC should decide or say

When the Low pass classifies a request as `complex` or `exceptional`, the browser automatically re-adjudicates the same authoritative state and player input with **GPT-6 Sol / Medium**. The Low result is not used as the final adjudication.

Only if the Medium pass still classifies the request as `exceptional` or explicitly recommends escalation does the browser send the case to **GPT-6 Astra / High**.

This implements the intended cost ladder:

```text
Sol / Low
   ↓ only when needed
Sol / Medium
   ↓ only when genuinely exceptional
Astra / High
```

High stakes, danger, emotional importance, or a low chance of success are explicitly not sufficient reasons by themselves to use Astra.

### Narration and NPC behavior

Narration now follows the final adjudication complexity instead of always using Low reasoning:

```text
routine final adjudication   → GPT-6 Sol / Low narration
complex final adjudication   → GPT-6 Sol / Medium narration
exceptional final adjudication → GPT-6 Sol / Medium narration after Astra has fixed the difficult adjudication
```

This keeps ordinary descriptive prose inexpensive while allowing relationship-heavy scenes, deception, multi-NPC exchanges, and other consequential social scenes to receive Medium reasoning where it matters.

Astra is used to solve unusually difficult referee problems, not merely to write prettier prose. Once Astra has fixed an exceptional adjudication, Sol / Medium can normally narrate the result faithfully.

### Campaign compilation

Campaign compilation remains GPT-6 Astra / High because it establishes the campaign constitution, initial world state, protagonist, NPCs, hidden information, constraints, and opening situation. No change was made to the campaign compilation contract in this iteration.

### Local engine responsibilities

The following remain local/browser-authoritative and do not consume AI reasoning simply to perform the mechanic:

- probability calculation and random sampling
- pending-action confirmation/cancellation where simple language can be recognized locally
- save-state ownership and validation
- inventory and established world-state truth
- deterministic mechanics
- future Damage / Penetration / Handling resolution
- future travel/resource/time mechanics

### Cloudflare / gateway

**No Worker update is required for v0.3.3.**

The existing stable Universal RPG Gateway v1.1.0 already permits client-side model routing and remains unchanged. Cloudflare continues to hold only the security/infrastructure boundary, principally `OPENAI_API_KEY` and `GAME_TOKEN`.

Normal development remains:

```text
replace index.html
replace audit.md
```

rather than redeploying the Worker.

### UI / diagnostics

The Settings connection test now reports the active role split explicitly:

- routine referee model / reasoning effort
- important NPC/referee model / reasoning effort
- routine narration model / reasoning effort
- significant narration model / reasoning effort
- campaign compiler model / reasoning effort

Debug records continue to capture the actual model and AI profile used for each call. New elevation events distinguish the transition from Sol / Low to Sol / Medium from the rarer escalation to Astra.

### Compatibility

- App version advanced to `0.3.3`.
- Save schema remains version `3`.
- Existing v0.3.x saves remain compatible.
- No campaign-state migration is required.
- Gateway requirement remains v1.1.0 or newer.

### Validation performed

- Embedded JavaScript extracted from `index.html` and passed `node --check`.
- Verified that ordinary adjudication uses the `adjudicate` Sol / Low profile.
- Verified that complex/exceptional first-pass results route through `adjudicate_medium` before any Astra escalation.
- Verified that only a Medium result still marked exceptional/escalation-worthy routes to `adjudicate_complex` (Astra / High).
- Verified that routine outcome narration uses `narrate` and non-routine outcome narration uses `narrate_medium`.
- Verified that campaign compilation remains mapped to Astra / High.
- No Worker source or Cloudflare variables were changed.

### Recommended validation after upload

1. Upload `index.html` v0.3.3 and this `audit.md`; do not modify the Worker.
2. Run **Settings → Test Worker** and confirm the displayed routing tiers are Sol/Low, Sol/Medium, and Astra/High as expected.
3. Test a trivial clarification such as `How far away is the door?`; debug should show Sol / Low only.
4. Test an ordinary NPC exchange; it should normally remain Sol / Low.
5. Test a deliberately meaningful NPC scene involving deception, conflicting motives, persuasion, or relationship consequences; debug should show a Low classification pass followed by a Sol / Medium adjudication.
6. Test several such scenes and verify Astra is not being invoked merely because a scene is dramatic or dangerous.
7. Export debug data after play and compare call counts/model usage before adjusting thresholds further.

### Known limitations / next checkpoint

- The Low pass is currently responsible for recognizing when a scene deserves Medium reasoning. Real play/debug data should be used to tune this threshold if it under- or over-escalates.
- Significant scenes incur an extra inexpensive Low classification/adjudication call before the Medium re-adjudication. This is intentional for cost control; it can later be replaced by a robust local router if sufficient patterns emerge.
- Probability calibration remains provisional.
- Damage / Penetration / Handling are still design-level concepts and remain a strong candidate for the next simulation-focused iteration.

---

## Version 0.3.2
Date: 2026-09-25

### Milestone
Implemented **cost-aware model routing** while keeping authoritative mechanics in local JavaScript. Ordinary play now uses GPT-6 Sol; GPT-6 Astra is reserved for campaign compilation and rare adjudications that genuinely require higher-capability review.

### Model routing

Default AI profiles now live entirely in `index.html`:

```text
campaign_compile      GPT-6 Astra / high
adjudicate            GPT-6 Sol / medium
adjudicate_complex    GPT-6 Astra / high
assessment             GPT-6 Sol / low
narrate                GPT-6 Sol / low
plain/test             GPT-6 Sol / low
world_generate         GPT-6 Astra / high   (reserved for future major generation work)
```

Local JavaScript remains authoritative for:

- probability calculation and random sampling
- campaign/world state storage and validation
- pending-action commitment state
- inventory/state updates once validated
- deterministic resolution gates
- future damage, penetration, handling, travel, and other simulation mechanics

The model does not roll the engine's probability result.

### Exceptional adjudication escalation

The normal adjudicator is now GPT-6 Sol. Its strict structured response contains three new routing fields:

```text
complexity
escalation_recommended
escalation_reason
```

Escalation is intended to be rare. Sol is instructed to recommend Astra only when correctness genuinely depends on unusually difficult multi-system interaction, conflicting hidden-state dependencies, complicated multi-actor sequencing, or difficult source/canon interpretation. High stakes, danger, or a low chance of success are explicitly **not** sufficient reasons to escalate.

When escalation is recommended, the browser sends the same authoritative state and player input to the `adjudicate_complex` Astra profile and uses Astra's result as the final adjudication.

### Reduced unnecessary AI calls

Simple responses to a pending action are now handled locally. Examples such as:

```text
yes
do it
fire
no
cancel
hold off
```

can confirm or cancel a pending action without spending a separate AI request. More complicated responses (for example, `No, I move closer first`) still go through Sol so the new plan can be interpreted correctly.

### Debug/cost visibility

AI debug entries now record the model and logical AI profile used for campaign compilation, adjudication, assessments, escalations, and narration. This makes exported debug files usable for later cost/performance tuning.

### Gateway v1.1.0 — one-time infrastructure update

Gateway v1.0.0 deliberately hard-coded `gpt-6-astra`. Because v0.3.2 routes work between Astra and Sol, one final gateway update is required. Gateway v1.1.0 no longer owns model selection: the authenticated `index.html` request supplies the model while the gateway continues to protect the API key, validate the game token/origin, cap request/output size, force `store:false`, and forward only permitted Responses API fields.

After Gateway v1.1.0 is deployed, **normal Universal RPG development again requires only `index.html` and `audit.md`**. Model routing profiles can be changed in HTML without modifying Cloudflare.

Gateway v1.1.0 requires only these Cloudflare Secrets:

```text
OPENAI_API_KEY
GAME_TOKEN
```

No Runtime Variables are required.

### Validation performed

- Updated embedded JavaScript passed `node --check`.
- Gateway v1.1.0 source passed `node --check`.
- Save schema remains version 3; existing v0.3.x saves remain compatible.
- Structured Output schemas were updated to require the new adjudication complexity/escalation fields.
- Connection test now requires Gateway `>= 1.1.0` and verifies the ordinary request path through GPT-6 Sol.
- Campaign compilation continues to use GPT-6 Astra.
- Probability computation and random sampling remain local and unchanged.

### Recommended validation after deployment

1. Deploy Gateway v1.1.0.
2. Confirm `/health` reports `gateway_version: 1.1.0` and `client_model_routing: true`.
3. Upload `index.html` v0.3.2.
4. Run **Settings → Test Worker**; it should report GPT-6 Sol as the ordinary-turn model and GPT-6 Astra as the campaign compiler.
5. Create a small test campaign and verify the campaign compilation debug entry used Astra.
6. Play ordinary clarification, assessment, and narration turns and verify debug entries use Sol.
7. Test a simple pending confirmation (`yes`) and confirm the debug log records `ai_used:false` for the confirmation itself.
8. Export debug after several turns and inspect model usage before further tuning.

### Known limitations / next checkpoint

- Escalation quality depends on Sol correctly recognizing genuinely exceptional adjudication. Debug logs should be reviewed before changing the thresholds or rules.
- Major runtime world-generation routing is reserved but not yet a separate gameplay path.
- Probability calibration remains provisional.
- Damage / Penetration / Handling are still design-level concepts and should be implemented in the next simulation-focused iteration rather than delegated to the language model.

---

## Version 0.3.1
Date: 2026-09-25

### Milestone
Decoupled Universal RPG game development from Cloudflare infrastructure. The Cloudflare Worker is now designed as a **stable security gateway** only. Universal RPG prompts, Structured Output schemas, referee contracts, and game logic have moved into `index.html`. Ordinary game iterations should now require only two uploaded files: `index.html` and `audit.md`.

### Architecture change

Previous flow:

```text
index.html
→ mode name
→ Worker-owned Universal RPG prompt/schema
→ OpenAI
```

New flow:

```text
index.html
├─ Universal RPG referee instructions
├─ Structured Output schemas
├─ campaign compiler contract
├─ adjudication contract
├─ assessment contract
├─ narration contract
└─ game mechanics/state
        ↓
Stable Cloudflare Gateway
├─ authenticates GAME_TOKEN
├─ protects OPENAI_API_KEY
├─ enforces origin/model/request limits
└─ forwards the permitted Responses API request
        ↓
OpenAI / GPT-6 Astra
```

The Worker no longer knows what `campaign_compile`, `adjudicate`, `assessment`, `narrate`, NPCs, pending actions, probability bands, or campaign state mean.

### Client changes

- Updated app version to `0.3.1`; save schema remains version `3` because authoritative game-state structure did not change.
- Moved the Universal RPG system/referee instructions into `index.html`.
- Moved all current strict JSON Structured Output schemas into `index.html`:
  - campaign compilation
  - intent/adjudication
  - player-facing assessment
  - outcome/narration
- `callWorker()` now builds an OpenAI Responses API request in the browser and sends it to the generic gateway.
- The client parses OpenAI output and Structured Output JSON itself.
- Connection testing now expects `gateway_version >= 1.0.0` rather than a game-specific Worker version.
- Campaign/debug events now record gateway version rather than Worker/game version.
- Updated UI wording to describe Cloudflare as the secure forwarding layer rather than the referee implementation.

### Stable gateway policy

The one-time replacement Worker is **Universal RPG Gateway v1.0.0**. It contains no Universal RPG rules. It performs only infrastructure/security duties:

- holds `OPENAI_API_KEY` server-side
- validates `GAME_TOKEN`
- enforces allowed browser origins
- restricts the allowed OpenAI model
- limits request and output size
- forces `store: false`
- permits strict JSON Schema Structured Outputs supplied by `index.html`
- forwards the request to the OpenAI Responses API
- returns the raw successful OpenAI response to the browser

Normal Universal RPG development should **not** require further Worker edits. A Worker change should be reserved for infrastructure changes such as moving domains, changing authentication, changing the allowed model, changing security limits, or adopting a materially different OpenAI API transport.

### Cloudflare configuration after migration

Keep only these two Cloudflare Secrets:

```text
OPENAI_API_KEY
GAME_TOKEN
```

After the stable Gateway v1.0.0 code is deployed successfully, these old Runtime Variables can be deleted because their equivalents are no longer read from Cloudflare:

```text
ALLOWED_ORIGINS
OPENAI_MODEL
REASONING_EFFORT
MAX_OUTPUT_TOKENS
MAX_INPUT_CHARS
```

The permitted GitHub Pages origin and hard security ceilings now live in the stable gateway source. Game-level model request settings and Structured Output contracts live in `index.html`.

### Deployment order

This is a one-time infrastructure migration:

1. Replace the current Cloudflare Worker code with **Universal RPG Gateway v1.0.0** and deploy it.
2. Open `/health`; it should report `service: universal-rpg-gateway` and `gateway_version: 1.0.0`.
3. Replace GitHub `index.html` with v0.3.1.
4. Use **Settings → Test Worker** in the game.
5. Once the test passes, delete the five obsolete Runtime Variables listed above.
6. Leave the two Secrets in place.

Future normal iterations: upload only `index.html` and `audit.md`.

### Validation performed

- Updated `index.html` embedded JavaScript passed `node --check`.
- Stable Gateway v1.0.0 source passed `node --check`.
- Verified current v0.3 game-state schema remains unchanged.
- Verified the browser now owns the current Structured Output schemas and AI referee instructions.
- Verified connection test no longer depends on a game-specific Worker version.
- Live validation still requires deploying Gateway v1.0.0 and exercising campaign compilation and at least one assessment/confirmation/resolution turn.

### Known limitations

- Gateway v1 currently permits string input plus strict JSON Schema text output, matching the current Universal RPG architecture. Future use of OpenAI tools, images, files, streaming, or another API endpoint would be an infrastructure change and could require a gateway revision.
- `gpt-6-astra` is currently the only model permitted by Gateway v1. Changing the model is intentionally treated as an infrastructure/configuration decision rather than an ordinary RPG rules iteration.
- The probability model remains provisional; this migration does not alter v0.3 probability calibration.

### Next development checkpoint

With infrastructure decoupled, return to game-engine work. The next major iteration should be the Probability Calibration / weapon-resolution layer, including the emerging universal weapon profile:

```text
Damage
Penetration
Range
Handling
Scale
Traits
Requirements / Resources
```

---

## Version 0.3.0
Date: 2026-09-25

### Milestone
Implemented the **informed decision / commitment loop**. Universal RPG now separates asking, assessing, deciding, resolving, and committing instead of treating every player sentence as an action. Consequential intentions normally pause for a character-informed risk assessment before the engine samples an outcome.

### Interaction model

Every player message is now structured as one of these interaction types:

- `clarify` — asks about information already perceivable or already known.
- `assess` — asks whether a contemplated action is likely to work or what its apparent risks are.
- `act` — attempts or proposes a physical/general action.
- `examine` — actively gathers new information in-world.
- `communicate` — speaks to or attempts to influence another character.
- `meta` — addresses the referee/game rather than acting in the fiction.
- `confirm_pending` / `cancel_pending` — responds to an outstanding consequential choice.

Clarification, assessment, and meta interaction do **not** advance world time, alter state, trigger NPC actions, or sample random outcomes.

### Consequence checkpoint

A new consequence gate runs before action resolution:

1. Is the proposed action possible?
2. Does the choice have a meaningful foreseeable consequence?
3. Is the outcome uncertain?

Meaningful consequences include injury, resource loss, exposure, irreversible commitment, substantial time cost, discovery, social consequences, and similar changes that matter to the player's decision.

If a consequential in-world action is proposed, the normal flow is now:

```text
Player intention
→ referee interpretation
→ probability/likelihood preview (no random sample)
→ character-informed assessment
→ player confirmation / reconsideration / clarification
→ resolution only after confirmation
→ state commit
→ narration
```

Actions with no meaningful consequence still resolve normally without an unnecessary checkpoint.

Guaranteed actions may still receive a checkpoint when the **cost** matters. Example: breaking a window may be automatically successful but still warrant warning the player that it will create loud noise and exposure.

### Immediate-action exception

A player can deliberately bypass the checkpoint by unmistakably demanding immediate execution, e.g.:

> "I shoot immediately. Don't wait."

Ordinary action wording such as "I shoot him" remains a committed intention but still receives a checkpoint when the foreseeable consequences are meaningful.

### Pending actions

Added a top-level `pending_action` interaction record. It is not world canon; it represents an uncommitted choice waiting on the player.

A pending action stores:

- original player wording
- summarized intent
- structured adjudication
- character-facing assessment message
- perceived likelihood band
- campaign state revision at which the assessment was made

While an action is pending, the player may:

- confirm it (`yes`, `do it`, `fire`, etc.)
- cancel it
- ask clarification while keeping it pending
- ask additional assessment questions
- replace it with a different plan

A confirmation is rejected if authoritative state changed after the assessment.

### Objective vs. perceived probability

Probability preview now distinguishes two models:

- **Objective resolution model** — uses all established facts that actually affect the action, including hidden facts.
- **Perceived assessment model** — uses only factors the protagonist can reasonably perceive, know, or infer.

This allows mysteries and deception to remain valid. For example, a sabotaged bridge can objectively be much more dangerous than it appears without the referee leaking the sabotage during a pre-action assessment.

The referee supplies separate objective and perceived difficulty/factor inputs. The browser calculates both; only the perceived result is used for player-facing assessment.

### Player-facing likelihood

The browser converts perceived probability into a semantic band before sending it back to Astra for natural phrasing. The current provisional bands are:

```text
essentially_certain
very_likely
good
favorable
uncertain
not_good
unlikely
very_unlikely
nearly_impossible
not_possible
```

The narrator is instructed to turn these into ordinary referee language rather than exposing a probability or repeating mechanical labels.

Example intended interaction:

```text
Player: I want to shoot the cultist.

Referee: You're not very good with a firearm, and he's pretty far away.
Your chances of hitting him aren't good. Do you still want to take the shot?

Player: Yes.

[Only now does the engine sample the outcome and commit consequences.]
```

### Worker changes

- Updated Cloudflare Worker to v0.3.0.
- Added Structured Output mode `assessment`.
- Expanded `adjudicate` schema with:
  - `intent_type`
  - `pending_disposition`
  - `commitment`
  - `action_possible`
  - `meaningful_consequence`
  - `checkpoint_required`
  - separate character-perceived assessment data
- Added `ASSESSMENT_SCHEMA` and reusable `FACTOR_SCHEMA`.
- Updated system instructions to enforce the informed-decision loop and prohibit hidden-state leakage during assessments.
- Structured schema identifiers moved to v3.
- `/health` now reports Worker version `0.3.0` after deployment.

### Client/state changes

- Updated `index.html` to v0.3.0.
- State schema is now version `3`.
- Added automatic migration from v0.1/v0.2 saves.
- Added `revision` for validating pending-action freshness.
- Added `pending_action` interaction state.
- World `turn` now increments only when an in-world action is actually committed; clarification and assessment do not consume turns.
- Added a subtle pending-decision notice above the input box.
- Input placeholder changes while the referee is waiting for confirmation.
- Debug snapshot now displays revision and pending-action status.
- Connection test now rejects Workers older than v0.3.0 before play begins.
- Debug/event log now records:
  - adjudication classification
  - assessment previews
  - checkpoint creation
  - confirmation/cancellation/replacement
  - objective probability resolution only after commitment

### Transaction behavior

The existing transactional safety is retained. If classification, assessment generation, probability resolution, narration, or state validation fails, the interaction restores the pre-message state and no partial world-state mutation remains committed.

### Validation performed

- `universal-rpg-worker.js` passed `node --check`.
- Embedded JavaScript from `index.html` passed `node --check`.
- Verified there is only one active definition of the v0.3 probability and turn-processing functions after migration.
- Confirmed v0.3 client recognizes save schemas 1, 2, and 3.
- Live OpenAI behavior still requires deployment of Worker v0.3 and browser testing.

### Important deployment requirement

Deploy `universal-rpg-worker.js` v0.3 **before** replacing the GitHub `index.html`. The v0.3 client uses the new `assessment` request mode and expanded adjudication schema.

No new Cloudflare variables or secrets are required. Existing configuration remains valid:

```text
OPENAI_API_KEY   (Secret)
GAME_TOKEN       (Secret)

OPENAI_MODEL = gpt-6-astra
REASONING_EFFORT = high
MAX_OUTPUT_TOKENS = 12000
MAX_INPUT_CHARS = 240000
ALLOWED_ORIGINS = https://diabolicdonut.github.io
```

### Known limitations

- Probability anchors, factor shifts, and the logistic curve remain provisional. v0.3 implements the decision workflow, not final probability calibration.
- Astra still chooses bounded objective/perceived difficulty inputs; these require calibration and consistency testing in v0.4.
- Risk-language thresholds are provisional and need empirical tuning.
- The current pending-action record is stored in the browser save envelope for reload safety even though it is not world canon.
- Hidden campaign state remains inspectable through browser storage/debug exports.
- NPC autonomy is structured but independent off-screen simulation remains a later milestone.
- Clarification depends on Astra correctly distinguishing "already observable/known" information from active investigation; debug review is needed for edge cases.

### Recommended next checkpoint

Run focused interaction tests before probability calibration. At minimum test:

- low-risk deterministic action: no checkpoint
- uncertain consequential action: assessment → confirmation → resolution
- guaranteed action with meaningful cost: assessment → confirmation → deterministic consequence
- explicit immediate action: checkpoint bypassed
- clarification: no turn/time/state mutation
- assessment-only question: no random sample or state mutation
- clarification while an action is pending: answer while preserving pending action
- cancel pending action
- replace pending action with a new plan
- hidden-factor case where objective risk differs from perceived risk

After those tests, v0.4 should calibrate the Probability Builder rather than adding new genre subsystems.

---

## Version 0.2.0
Date: 2026-09-25

### Milestone
Converted the first prototype from prompt-requested JSON to **API-enforced Structured Outputs** and formalized the first authoritative Universal RPG state schema. This checkpoint establishes the transport/state contract that later probability calibration, autonomous NPC simulation, world events, and source/module compilation will build on.

### Worker changes

- Updated Cloudflare Worker to v0.2.0.
- Added request modes:
  - `plain`
  - `campaign_compile`
  - `adjudicate`
  - `narrate`
- Added OpenAI Responses API Structured Outputs for the three game/referee modes using `text.format.type = json_schema` with `strict: true`.
- Added server-side schemas for:
  - campaign compilation
  - consequence-gate adjudication
  - post-resolution narration/state updates
- Worker now parses structured model output itself and returns it in `data`; the browser no longer depends on extracting JSON from prose/code fences.
- Added refusal and incomplete-response handling.
- `/health` now reports:
  - Worker version
  - configured model
  - whether Structured Outputs are enabled
- Preserved `store: false`; OpenAI remains stateless and never becomes a second hidden campaign database.
- Preserved GAME_TOKEN authentication and CORS origin controls.

### Authoritative state schema v2

The browser state now has explicit schema version `2` and includes:

- campaign constitution
- one player persona
- world state
- autonomous NPC records
- open threads/situations
- reserved clocks collection
- transcript
- event log
- debug log

Campaign compilation now requires stable IDs for generated entities and formalizes:

- immutable campaign foundation facts
- allowed/prohibited generation constraints
- source fidelity/deviation policy
- protagonist public and hidden capabilities
- protagonist knowledge records with confidence (`known`, `probable`, `suspected`)
- places with public description, hidden facts, and local generation constraints
- world facts with public/hidden visibility
- resources as typed records rather than arbitrary object keys
- NPC public profile
- NPC capabilities
- NPC hidden nature, goals, knowledge, loyalties, fears, secrets, temperament, current intention, and relationship state
- public/hidden adventure threads

### State-update contract

Removed free-form path patches from model output. Astra can now propose only domain-specific update collections:

- time label
- current place
- player condition
- inventory additions
- player knowledge additions
- world fact additions
- known-place additions
- NPC additions
- NPC status changes
- NPC relationship changes
- NPC intention changes
- thread additions
- thread status changes

The browser validates references and duplicate IDs before committing the update set. Campaign foundation cannot be modified through ordinary turn output.

### Turn transaction safety

- A full pre-turn snapshot is now taken before adjudication.
- A failed Worker request, invalid structured response, invalid probability request, or invalid state update rolls the turn back to the pre-turn state.
- The failure is recorded as a system message only after rollback.
- This prevents partially committed turns where prose and state disagree.

### Referee pipeline

The live turn pipeline is now:

1. Player supplies natural-language intent.
2. Astra interprets intent and applies the resolution gate.
3. If deterministic, Astra returns narration plus validated domain updates.
4. If consequential uncertainty exists, Astra supplies only bounded capability/difficulty/factor inputs.
5. Browser probability engine resolves the random outcome.
6. Astra receives the fixed result and narrates it.
7. Domain-specific updates are validated and committed transactionally.
8. Event/debug records are appended.

NPC actions are also returned as structured records containing the NPC, action, whether the action is player-visible, and a hidden reason. This is groundwork for a later independent NPC simulation pass.

### Client improvements

- Updated `index.html` to v0.2.
- Added clearer network/CORS diagnostics when `fetch` fails before a Worker response is received.
- Connection test now reports Worker version, model, and Structured Outputs status.
- Added a small player-facing `Known` panel based on explicit protagonist knowledge records.
- Added v0.1 -> v0.2 save migration.
- Imported saves are checked for a recognized Universal RPG schema before use.
- Campaign compile results are validated for required sections and duplicate IDs.
- Current place is guaranteed to appear in known places after compilation.

### Validation performed

- `universal-rpg-worker.js` passed `node --check`.
- Embedded `index.html` JavaScript passed `node --check`.
- Verified current OpenAI documentation before implementation:
  - GPT-6 Astra supports the Responses API and Structured Outputs.
  - Responses Structured Outputs use `text.format` with `type: json_schema` and `strict: true`.
  - Structured Output object fields must be required and objects must specify `additionalProperties: false`.
  - nullable fields may be represented with a union including `null`.
- Live v0.2 Worker/API behavior cannot be validated until the updated Worker is deployed by the user.

### Important deployment requirement

`index.html` v0.2 expects Worker v0.2 or later. Deploy the updated Worker **before** testing campaign compilation. If the old Worker remains deployed, the client will report that structured data was not returned.

Recommended Cloudflare configuration remains:

```text
OPENAI_API_KEY   (Secret)
GAME_TOKEN       (Secret)

OPENAI_MODEL = gpt-6-astra
REASONING_EFFORT = high
MAX_OUTPUT_TOKENS = 12000
MAX_INPUT_CHARS = 240000
ALLOWED_ORIGINS = https://diabolicdonut.github.io
```

### Known limitations

- Probability anchors and logistic curve remain prototype values and have not been calibrated.
- Astra still selects the bounded capability/difficulty/factor inputs; consistency across many equivalent situations still requires an evaluation suite.
- Hidden state remains stored in the browser and can be inspected by a technically inclined player.
- NPC action records are structured but NPCs are not yet simulated off-screen on an independent clock.
- Place/Being/Item/Weapon/Vehicle/Hazard/Capability schemas are not yet generalized into the full universal object family.
- Clocks are reserved in state but have no live advancement procedure yet.
- Long-campaign context compaction/summarization is not implemented.
- Uploaded sourcebooks/modules are not yet compiled or provenance-tracked.

### Recommended next checkpoint

1. Deploy Worker v0.2 and GitHub `index.html` v0.2.
2. Run a fresh Dinotopia campaign compile and export the debug JSON.
3. Exercise 8-12 representative turns containing:
   - deterministic actions
   - consequential uncertain actions
   - requests made to autonomous companions
   - information gathering
   - movement between places
4. Audit all structured state changes for continuity.
5. Then build the **Probability Calibration Suite** before adding combat or other large subsystems.

---

## Version 0.1.0
Date: 2026-09-25

### Purpose
Initial playable architectural prototype for **Universal RPG**, a browser-hosted single-protagonist RPG using a GitHub-hosted `index.html`, a Cloudflare Worker, and GPT-6 Astra through the OpenAI Responses API.

The design goal is not to expose a tabletop rules interface. The player controls one persona in natural language while hidden mechanics, persistent state, and autonomous NPCs operate underneath.

## Added

- Single-file `index.html` suitable for static hosting such as GitHub Pages.
- Preconfigured Worker base URL:
  - `https://universal-rpg.diabolicdonut.workers.dev`
- Connection settings dialog with:
  - Worker URL
  - GAME_TOKEN entry
  - session-only token storage by default
  - optional localStorage token persistence
  - Worker/OpenAI connection test
- Campaign compiler form with:
  - campaign title
  - genre / setting
  - premise
  - tone
  - era / technology
  - source mode: Original / Canonical / Adaptive / Inspirational
  - player character name and concept
  - campaign/source notes
- One-persona architecture:
  - the player controls only the protagonist
  - NPCs remain autonomous entities
  - companions have public and hidden state
  - hidden NPC facts are not rendered in the normal UI
- Initial persistent state model containing:
  - campaign constitution
  - player dossier
  - world state
  - NPCs
  - threads / open situations
  - transcript
  - event log
  - debug log
- Local browser persistence via `localStorage`.
- Save export and import as JSON.
- Full debug export with an explicit warning that hidden/referee information is included.
- Consequence-driven resolution gate:
  - probability is used only when the outcome is genuinely uncertain **and** materially consequential
  - deterministic actions do not receive unnecessary checks
  - time pressure can qualify as a meaningful consequence
  - impossible actions resolve deterministically rather than being given a token chance
- First hidden probability builder:
  - capability levels: untrained, novice, familiar, competent, skilled, expert, master, legendary
  - challenge bands: routine, easy, standard, difficult, formidable, extreme
  - bounded circumstance factors: minor / significant / major advantage or hindrance
  - capability and challenge are converted into a logistic success probability
  - random sampling uses `crypto.getRandomValues()`
  - results produce ordinary / strong / exceptional success or failure magnitude
- Two-phase uncertain-action pipeline:
  1. Astra determines whether resolution is required and identifies bounded inputs.
  2. Browser probability engine resolves the uncertainty.
  3. Astra narrates the already-fixed result and proposes state changes.
- Basic state-change validation:
  - only whitelisted state roots are writable
  - only `set` and `append` are currently supported
  - prototype-pollution paths are rejected
  - campaign foundation is not directly writable by turn patches
- Basic desktop/mobile responsive interface.

## Architectural Rules Established

1. **One player, one persona.**
   Everyone else is part of the simulated world.

2. **NPCs have agency.**
   Companions may cooperate, disagree, leave, pursue personal interests, or possess hidden information according to established state.

3. **Established state is canon.**
   Existing facts are not rerolled merely because a later prompt creates uncertainty.

4. **Only consequential uncertainty is resolved probabilistically.**
   The game does not roll merely because an action resembles a traditional skill check.

5. **Mechanics are hidden; causes are not.**
   The player receives fictional information sufficient to make decisions rather than raw modifiers and probabilities.

6. **The engine resolves; Astra narrates.**
   When an uncertain action is resolved, Astra receives the fixed result and may not reroll or reverse it.

7. **The browser is currently the authoritative state store.**
   The Worker remains stateless (`store: false` on the OpenAI request in the current Worker design).

## Probability Builder — Current Prototype

Internal anchor values are intentionally simple and are expected to be recalibrated.

### Capability anchors
- Untrained: 25
- Novice: 35
- Familiar: 45
- Competent: 55
- Skilled: 65
- Expert: 75
- Master: 85
- Legendary: 95

### Challenge anchors
- Routine: 20
- Easy: 35
- Standard: 50
- Difficult: 65
- Formidable: 80
- Extreme: 95

### Circumstance shifts to challenge
- Major advantage: -20
- Significant advantage: -10
- Minor advantage: -5
- Minor hindrance: +5
- Significant hindrance: +10
- Major hindrance: +20

Probability is derived from capability minus final challenge using a logistic curve. It is presently clamped to 0.5%–99.5% for uncertain actions. Impossible or effectively certain actions should instead be caught by the resolution gate and resolved deterministically.

### Degree of outcome
For successful rolls, magnitude is normalized within the successful probability range. For failed rolls, magnitude is normalized within the failure range.

Current bands:
- ordinary: < 50% normalized magnitude
- strong: 50%–79.99%
- exceptional: >= 80%

These bands are internal and are not shown to the player.

## Validation Performed

- Cloudflare `/health` endpoint was visibly verified by the user and returned:
  `{"ok":true,"service":"universal-rpg","status":"ready"}`
- `index.html` is self-contained and does not hard-code the GAME_TOKEN or OpenAI API key.
- Worker endpoint is preconfigured but editable from Settings.
- State patch paths reject `__proto__`, `prototype`, and `constructor`.
- New campaign state and imported save state use an explicit schema version.
- Browser random sampling uses a cryptographic random source rather than `Math.random()`.

## Known Issues / Limitations

- This is an architectural prototype, not a balanced final rules engine.
- The current Probability Builder has not yet been statistically calibrated against reference tasks, real-world rates, or source-system probability distributions.
- Astra currently supplies the qualitative capability selection, difficulty band, and relevant circumstance categories. These are bounded, but adjudication consistency still needs testing.
- The Worker currently returns plain text. The HTML asks Astra to return JSON and parses it defensively, but true API-level Structured Outputs are not yet enforced.
- Hidden world/NPC data is stored client-side. It is not displayed normally, but a technically inclined player could inspect browser storage. Server-side referee state may be desirable for public/mystery-focused releases.
- Debug export intentionally contains hidden information and can spoil mysteries.
- State patch support is intentionally narrow. Complex updates may be rejected until more operations are implemented.
- No formal item, weapon, vehicle, place, hazard, culture, faction, or capability schemas are implemented yet beyond loose campaign-generation data.
- No imported PDF/module compiler exists yet.
- No source provenance tracking exists yet.
- No NPC behavioral simulation loop exists independently of model adjudication yet.
- No independent world-time scheduler or off-screen NPC simulation exists yet.
- No calibrated combat, injury, armor, travel, survival, vehicle, or scale subsystem exists yet.
- No automatic transcript summarization or state compaction exists yet; long-running campaigns will eventually require context management.
- GitHub Pages origin must be added to the Worker's `ALLOWED_ORIGINS` variable before hosted play will work from that origin.

## Recommended Next Steps

1. **Deploy and connection-test v0.1.**
   Verify campaign compilation and at least several deterministic and uncertain turns through the live Worker.

2. **Switch Worker responses to Structured Outputs.**
   Enforce JSON schemas for campaign compilation, adjudication, and outcome narration at the API level rather than relying on prompt compliance.

3. **Build a Probability Calibration Suite.**
   Define reference cases such as routine professional work, expert marksmanship, difficult climbing, opposed stealth, medical treatment, and extreme tasks. Tune the mapping until results behave consistently.

4. **Formalize universal schemas.**
   Add typed schemas for Place, Being, Item, Weapon, Vehicle, Hazard, Capability, Skill/Competency, Culture, and Faction.

5. **Separate objective truth from knowledge.**
   Formalize True Profile / Observed Profile / Believed Profile and per-entity knowledge visibility.

6. **Add autonomous NPC behavior.**
   NPC actions should emerge from goals, knowledge, temperament, loyalties, relationships, obligations, and circumstances rather than generic narrative invention.

7. **Add world-event and trigger systems.**
   Implement keyed triggers, clocks/progress, relevant-entity weighting, and bounded surprise generation inspired by useful Mythic concepts without making narrative pressure alter physical rules.

8. **Add source/module compilation.**
   Support Canonical / Adaptive / Inspirational source handling, provenance, immutable facts, adventure features, and module-to-universal-mechanics translation.

9. **Stress-test with contrasting campaigns.**
   Recommended order:
   - a small Black Meridian-style Lovecraft scenario
   - a Smoking Pillar-style fantasy wilderness/site adventure
   - a Dinotopia-style cooperative exploration scenario
   - a small Traveller-like science-fiction scenario

## Files

- `index.html` — Universal RPG v0.1 playable client
- `audit.md` — this running project history
