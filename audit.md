# Universal RPG — Audit Log

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
