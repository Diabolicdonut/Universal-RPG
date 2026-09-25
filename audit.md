# Universal RPG — Audit Log

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
