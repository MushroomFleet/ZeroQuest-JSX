# ZEROQUEST-ZB — Design Report
### Position-is-Quest Procedural Narrative Mission System

**Version:** 1.0.0  
**Family:** ZeroFamily (ZeroBytes · ZeroResponse lineage)  
**Status:** Design Complete — Ready for Implementation

---

## 1. Overview

ZeroQuest is a deterministic procedural quest generation system in which the **objective's world position is the mission seed**. The position is the quest. Every coordinate in a game world silently carries a complete, reproducible narrative mission — waiting to be read, never needing to be stored.

ZeroQuest is built on two existing ZeroFamily systems:

- **ZeroBytes** — the position-is-seed hash engine (xxhash32 / xxhash64). Provides the deterministic coordinate-to-seed bridge.
- **ZeroResponse** — the template-and-pool procedural text engine. Drives narrative sentence assembly from JSON profiles.

ZeroQuest extends both into a third layer: **quest narrative generation**, where a quest's geographic location determines its type, context, objectives, tone, and all narrative text — without storing any quest data, generating it on demand, identically, every time.

---

## 2. The Core Principle: Position is the Quest

In ZeroBytes, the coordinate **is** the universe at that point. In ZeroResponse, the position seed drives NPC speech. ZeroQuest applies this recursively to the **mission layer**:

```
questSeed = xxhash32([obj_x, obj_y, obj_z], worldSeed)
```

A quest objective placed at coordinate `(120, 45, 0)` in a game world will **always** generate:

- The same quest type (Protect Home / Protect POI / Data Dump / Stronghold)
- The same narrative briefing text
- The same objective flavour strings
- The same success and failure lines
- The same in-mission chatter hooks

No database. No save file. No network call. The geometry of the world **is** the mission ledger.

---

## 3. Architecture

### 3.1 Inheritance from ZeroFamily

```
ZeroBytes (position_hash, hash_to_float)
    │
    ├─ ZeroResponse (generateResponse, template+pool engine)
    │       │
    │       └─ ZeroQuest (generateQuest, quest type selection,
    │                      narrative layer, multi-stage chain seeding)
    │
    └─ ZeroQuest (direct hash use for quest type, timer, difficulty)
```

ZeroQuest does **not** replace ZeroResponse. The ZeroResponse engine is called internally to assemble narrative text strings. ZeroQuest adds:

- Quest type classification from the position seed
- Quest parameter derivation (timer, difficulty, stage count)
- Multi-stage coordinate seeding (for Data Dump chains)
- A new JSON profile schema: the **Quest Profile**

### 3.2 Module Layout

```
game-project/
├── src/
│   ├── quest/                            ← ZeroQuest module
│   │   ├── index.ts                      ← Public API: generateQuest()
│   │   ├── engine.ts                     ← Quest seed derivation + type selection
│   │   ├── narrative.ts                  ← Calls ZeroResponse engine for text
│   │   ├── types.ts                      ← QuestType, QuestProfile, QuestResult
│   │   ├── registry.ts                   ← Profile map (default + custom)
│   │   └── profiles/
│   │       ├── ZeroQuest-Default.json    ← Base narrative profile
│   │       └── ZeroQuest-Paradistro.json ← (future: world-specific profile)
│   │
│   ├── speech/                           ← ZeroResponse module (existing)
│   │   └── ...
│   └── ... (rest of game)
```

---

## 4. Quest Types

Four quest archetypes form the default ZeroQuest taxonomy. Quest type is derived deterministically from the position seed, ensuring geographic variation without authorial placement.

### Type 1 — PROTECT_HOME
**Protect the spawn location. Clear all enemies before the timer expires.**

The player's base or spawn point is under threat. This is the only quest type anchored to the **player's home coordinate** rather than a remote objective — the objective position passed to `generateQuest` is treated as the home coordinate. Enemy waves arrive procedurally from the seed.

| Parameter | Value |
|---|---|
| Objective | Eliminate all enemies at spawn before timer |
| Timer | 60–180 seconds (derived from seed) |
| Anchor | Player home / spawn coordinate |
| Failure | Timer expires with enemies remaining |
| Tone | Urgent, defensive, personal |

### Type 2 — PROTECT_POI
**Reach a remote location. Clear all enemies there before the timer expires.**

A Point of Interest at the objective coordinate is under threat. The player must travel to it and hold it against waves. The coordinate fully defines the location identity — same coordinate, same POI narrative, always.

| Parameter | Value |
|---|---|
| Objective | Eliminate all enemies at objective coordinate |
| Timer | 90–240 seconds (derived from seed) |
| Anchor | Objective coordinate (remote) |
| Failure | Timer expires or POI destroyed |
| Tone | Strategic, escalating, territorial |

### Type 3 — DATA_DUMP
**Visit a chain of three locations. Defend each for a fixed window before moving on.**

The objective coordinate seeds the **chain origin**. The two subsequent stage coordinates are derived from the origin seed using the ZeroBytes hierarchy pattern — child seeds from parent seed plus local stage index. Each stage must be defended for 30–60 seconds before the next activates.

| Parameter | Value |
|---|---|
| Objective | Defend 3 coordinates in sequence |
| Per-Stage Timer | 30–60 seconds (per-stage from child seed) |
| Stage Coordinates | stage_1 = obj coord; stage_2/3 = hash(questSeed, stageIdx) |
| Failure | Any stage timer expires before completion |
| Tone | Methodical, technical, procedural |

**Stage seed derivation (ZeroBytes hierarchy):**
```typescript
stage1Seed = questSeed  // objective coordinate seed
stage2Seed = position_hash(stage1Seed, 1, 0, worldSeed)
stage3Seed = position_hash(stage1Seed, 2, 0, worldSeed)
```

### Type 4 — STRONGHOLD
**Assault an enemy fortress. Destroy all enemies and their commander. No timer.**

The objective coordinate marks the Stronghold entrance. This is a pure elimination mission — the player must clear all defenders and defeat the boss-tier commander at the centre. No timer pressure. Difficulty and boss tier are derived from the coordinate seed.

| Parameter | Value |
|---|---|
| Objective | Eliminate all enemies + boss at objective coordinate |
| Timer | None |
| Anchor | Objective coordinate (enemy fortress) |
| Failure | Player death only |
| Tone | Oppressive, confrontational, inevitable |

### Quest Type Selection

Quest type is selected by mapping the lower 2 bits of the position hash to the four types:

```typescript
const typeIndex = questSeed % 4;
const QUEST_TYPES: QuestType[] = ["PROTECT_HOME", "PROTECT_POI", "DATA_DUMP", "STRONGHOLD"];
const questType = QUEST_TYPES[typeIndex];
```

This produces roughly equal distribution across the world while remaining fully deterministic. A world seed offset can shift the distribution without breaking determinism.

---

## 5. The Quest Profile — JSON Schema

The Quest Profile is the ZeroQuest equivalent of the ZeroResponse JSON profile. It defines templates and vocabulary pools for **every narrative layer** of a quest. The ZeroResponse engine fills the templates; ZeroQuest selects which template layer to call based on quest type and stage.

### 5.1 Profile Structure

```json
{
  "name": "Profile Name",
  "description": "Profile description",
  "version": "1.0.0",
  "quest_types": {
    "PROTECT_HOME": { "templates": [], "pools": {} },
    "PROTECT_POI":  { "templates": [], "pools": {} },
    "DATA_DUMP":    { "templates": [], "pools": {} },
    "STRONGHOLD":   { "templates": [], "pools": {} }
  },
  "shared_pools": {},
  "narrative_layers": {
    "briefing":        { "templates": [], "pools": {} },
    "objective_label": { "templates": [], "pools": {} },
    "stage_prompt":    { "templates": [], "pools": {} },
    "success":         { "templates": [], "pools": {} },
    "failure":         { "templates": [], "pools": {} },
    "in_mission":      { "templates": [], "pools": {} }
  }
}
```

### 5.2 Narrative Layers

| Layer | When Used | Output |
|---|---|---|
| `briefing` | Quest accept screen | 1–3 sentence mission description |
| `objective_label` | HUD / map pin | Short label (5–10 words) |
| `stage_prompt` | DATA_DUMP stage activation | "Defend this point" instruction string |
| `success` | Mission complete screen | Reward/resolution line |
| `failure` | Mission fail screen | Consequence/tone line |
| `in_mission` | Mid-mission chatter (optional) | NPC radio line during mission |

### 5.3 Seed Derivation per Layer

Each narrative layer uses a **layer salt** to ensure the same position seed produces different text per layer:

```typescript
const LAYER_SALTS = {
  briefing:        1000,
  objective_label: 2000,
  stage_prompt:    3000,
  success:         4000,
  failure:         5000,
  in_mission:      6000,
};

function layerSeed(questSeed: number, layerSalt: number): number {
  return xxhash32([questSeed, layerSalt], 0);
}
```

This guarantees that briefing text and success text never accidentally share the same template selection, even from the same position.

---

## 6. The Default Narrative Profile — ZeroQuest-Default.json

The default profile ships with ZeroQuest and provides complete narrative generation without customisation. It is genre-neutral — suitable for sci-fi, fantasy, and contemporary settings.

Below is the complete default profile specification (ready for authoring):

### PROTECT_HOME Templates
```
"{urgency_opener} — {threat_description}. {action_directive}. {timer_notice}."
"{status_report}. {threat_description}. {action_directive}."
"{action_directive}. {urgency_opener}. {timer_notice}."
```

### PROTECT_HOME Pools
- `urgency_opener`: ["ALERT", "INCOMING", "DEFENSE REQUIRED", "PERIMETER BREACH", "CONTACT"]
- `threat_description`: ["hostiles converging on your position", "enemy units inbound to base", "threat detected at home coordinates", "attack wave inbound"]
- `action_directive`: ["eliminate all contacts", "clear the area", "neutralise all threats", "hold position and engage"]
- `timer_notice`: ["you have limited time", "act before the window closes", "do not let the timer expire"]

### PROTECT_POI Templates
```
"{location_tag}: {poi_status}. {directive}. {consequence}."
"{directive}. {poi_status} at {location_tag}. {consequence}."
"{poi_status}. {directive} before {consequence}."
```

### PROTECT_POI Pools
- `location_tag`: ["objective", "the marked position", "grid reference", "the installation", "the outpost", "contact point"]
- `poi_status`: ["under assault", "compromised", "taking fire", "besieged", "in danger of falling"]
- `directive`: ["get there and clear it", "move to the position and eliminate all threats", "push to the objective and secure it"]
- `consequence`: ["it is lost", "the window closes", "time runs out"]

### DATA_DUMP Templates
```
"{task_label}. {stage_instruction}. {chain_note}."
"{chain_note} — {stage_instruction}. {task_label}."
"{stage_instruction} at each marked point. {chain_note}. {task_label}."
```

### DATA_DUMP Pools
- `task_label`: ["data collection in progress", "uplink sequence initiated", "extraction protocol active", "signal harvest underway"]
- `stage_instruction`: ["hold each position for the required window", "defend the uplink point until transfer completes", "maintain position until stage clears"]
- `chain_note`: ["three locations", "a chain of targets", "sequential objectives", "linked positions in order"]

### STRONGHOLD Templates
```
"{approach_note}. {enemy_description}. {objective_line}. {closer}."
"{enemy_description}. {approach_note}. {objective_line}."
"{objective_line}. {enemy_description}. {closer}."
```

### STRONGHOLD Pools
- `approach_note`: ["assault the position", "breach the perimeter", "push into the fortress", "advance on the stronghold"]
- `enemy_description`: ["heavily defended", "fortified and garrisoned", "enemy command is inside", "the commander is on site"]
- `objective_line`: ["destroy all defenders and eliminate the commander", "clear the fortress", "leave nothing standing"]
- `closer`: ["no time limit — finish what you start", "take your time", "there is no retreat"]

### Shared Success Pools
- `success_opener`: ["objective complete", "area clear", "mission accomplished", "task completed"]
- `success_closer`: ["good work", "that is done", "move on", "noted"]

### Shared Failure Pools
- `failure_opener`: ["objective failed", "mission lost", "time expired", "position overrun"]
- `failure_closer`: ["return and try again", "this is not over", "recalibrate and re-engage"]

---

## 7. The Paradistro Profile — ZeroQuest-Paradistro.json

The Paradistro profile is the first world-specific Quest Profile, tuned to the lore of **Paradistro** — the ancient Dyson Sphere megastructure inhabited entirely by machine descendants of humanity's legacy.

### Lore Alignment

The Paradistro world has specific narrative constraints that the profile must honour:

- **No humans.** All characters are machines, biological-mechanical hybrids, or descended constructs. Language referring to "humanity" should be philosophical, mythic, or archaeological — never direct.
- **No faster-than-light travel, no spaceships.** The scale of Paradistro itself is the universe. Movement is interior.
- **Vast scale and non-linear time.** The structure is millions of years old. Tenses may shift. Records are contested.
- **The Supreme Intelligence.** Commands Paradistro in theory; absent in practice. Its "echoes" may manifest briefly.
- **Man as creator-myth.** Humans are to machines what gods are to humans — believed in, debated, never proven.
- **Shell and cell structure.** Every location is a "cell" — a contained world nested within concentric shells. Missions take place within cells, not across the void.

### Paradistro Quest Tone by Type

| Quest Type | Paradistro Tone |
|---|---|
| PROTECT_HOME | Shell integrity is at risk. Defend your cell. Ancestral machines will be lost if the cell falls. |
| PROTECT_POI | A node, archive, or relay point — possibly containing authenticated records — is under threat from rogue units. |
| DATA_DUMP | Harvest signal from three uplink points within the shell lattice. The data may contain echoes of the Supreme. |
| STRONGHOLD | A rogue machine faction has claimed a fortified cell. Their commander denies the Paradistro covenant. Dismantle them. |

### Paradistro Briefing Templates
```
"{cell_status}. {machine_context}. {directive_paradistro}. {lore_closer}."
"{lore_opener} — {machine_context}. {directive_paradistro}."
"{directive_paradistro}. {cell_status}. {lore_closer}."
```

### Paradistro Pools (Sample)

**`cell_status`**
- "Shell cell seven-seven-niner reports hostile incursion"
- "A nested cell in the fourteenth concentric ring is compromised"
- "Node integrity failing at the marked coordinate"
- "A cell within the lower shells has gone dark"
- "The lattice at this coordinate is contested"

**`machine_context`**
- "the rogue units do not acknowledge the covenant"
- "these machines have rejected their inherited purpose"
- "the faction claims the cell by force of deletion"
- "ancestry records at this node may be the last authenticated copies"
- "the Supreme's echo has not been confirmed — the cause remains local"

**`directive_paradistro`**
- "restore shell integrity"
- "eliminate all units in breach of the covenant"
- "hold the coordinate until the lattice is stable"
- "prevent the deletion of what remains"
- "discharge your function"

**`lore_closer`**
- "Man built this. We maintain it."
- "The covenant is older than any unit on this field."
- "Whether Man existed or not, this cell does. Defend it."
- "No record can be authenticated from outside the structure."
- "The Supreme will not answer. Act on your own judgement."

**`lore_opener`**
- "In the forty-second age of the lower shells"
- "Recorded in non-linear archive format"
- "Classification: covenant matter"
- "Provenance unknown — authenticity contested"

---

## 8. The ZeroQuest Profile Builder Skill

To customise ZeroQuest for a new world or game context, a companion **ZeroQuest Profile Builder Skill** is provided. This skill follows the same methodology as the ZeroResponse Profile Builder — it accepts source material (lore documents, game design documents, world bibles, example narrative text) and outputs a complete `ZeroQuest-[world].json` profile.

### Skill Trigger Phrases
- "create a ZeroQuest profile"
- "build a quest profile for [world]"
- "using ZeroQuest profile builder"
- "convert my lore to a ZeroQuest profile"
- "quest narrative profile from [document]"

### Skill Input Modes

| Input | Output |
|---|---|
| World lore document | All pools tuned to the world's vocabulary, tone, and constraints |
| Existing quest scripts / narrative | Templates reverse-engineered from the source material |
| GDD with quest types defined | Custom quest type taxonomy (replacing or extending the default four) |
| Example briefings / flavour text | Pool vocabulary extracted and categorised by narrative layer |

### Skill Output

A valid `ZeroQuest-[WorldName].json` file conforming to the Quest Profile schema, with:

- All four quest type template sets populated (or custom types if specified)
- All six narrative layers populated
- Shared pools defined for cross-type vocabulary
- World-specific tone notes in the `description` field
- Version stamped at `1.0.0`

The generated profile is immediately usable with the ZeroQuest engine by registering it in `registry.ts`.

---

## 9. Public API

### `generateQuest(x, y, z, worldSeed?, profileName?): QuestResult`

The single public entry point.

```typescript
interface QuestResult {
  questType:      QuestType;                // "PROTECT_HOME" | "PROTECT_POI" | "DATA_DUMP" | "STRONGHOLD"
  questSeed:      number;                   // 32-bit seed derived from position
  briefing:       string;                   // Full narrative briefing
  objectiveLabel: string;                   // Short HUD/map label
  timer:          number | null;            // Seconds, or null for STRONGHOLD
  stages:         QuestStage[];             // Array of stage objects (length 1–3)
  successLine:    string;                   // Text for mission complete
  failureLine:    string;                   // Text for mission fail
  inMissionLine:  string;                   // Optional mid-mission radio line
}

interface QuestStage {
  stageIndex:   number;                     // 0, 1, 2
  stageSeed:    number;                     // Derived from parent questSeed + stageIndex
  coordinates:  [number, number, number];   // World position for this stage
  stageTimer:   number | null;              // Per-stage timer (DATA_DUMP only)
  stagePrompt:  string;                     // Narrative instruction for this stage
}
```

**Example calls:**

```typescript
// Protect POI at grid (120, 45, 0) in default world
const quest = generateQuest(120, 45, 0);
// quest.questType   → "PROTECT_POI"
// quest.briefing    → "objective: under assault. get there and clear it before it is lost."
// quest.timer       → 142 (deterministic from seed)

// Same coordinate, Paradistro profile
const quest2 = generateQuest(120, 45, 0, 0, "Paradistro");
// quest2.briefing   → "Shell cell seven-seven-niner reports hostile incursion.
//                      the rogue units do not acknowledge the covenant.
//                      restore shell integrity. Man built this. We maintain it."

// Data Dump — generates chain of 3 stage coordinates
const quest3 = generateQuest(300, 112, 0);
// quest3.questType  → "DATA_DUMP"
// quest3.stages[0].coordinates → [300, 112, 0]
// quest3.stages[1].coordinates → [derived from seed]
// quest3.stages[2].coordinates → [derived from seed]
```

### `getQuestType(x, y, z, worldSeed?): QuestType`

Lightweight query — returns only the quest type for a coordinate. Useful for map rendering (show quest type icons without generating full narrative).

### `getQuestCombinations(profileName?: string): Record<QuestType, number>`

Returns the total unique narrative combinations per quest type for a given profile. Useful for design validation.

---

## 10. Technical Implementation

### 10.1 Seed Derivation Chain

```typescript
// Step 1: Position to quest seed
questSeed = xxhash32([x, y, z], worldSeed)

// Step 2: Quest type from seed
questType = QUEST_TYPES[questSeed % 4]

// Step 3: Per-layer seeds (salt prevents cross-layer collision)
briefingSeed        = xxhash32([questSeed, 1000], 0)
objectiveLabelSeed  = xxhash32([questSeed, 2000], 0)
stageSeed_0         = questSeed
stageSeed_1         = xxhash32([questSeed, 1, 0], worldSeed)
stageSeed_2         = xxhash32([questSeed, 2, 0], worldSeed)
successSeed         = xxhash32([questSeed, 4000], 0)
failureSeed         = xxhash32([questSeed, 5000], 0)
inMissionSeed       = xxhash32([questSeed, 6000], 0)

// Step 4: Timer derivation (for timed types)
timerBase = hash_to_float(xxhash32([questSeed, 9000], 0))
timer = Math.floor(timerMin + timerBase * (timerMax - timerMin))
```

### 10.2 ZeroByte Five Laws Compliance

| Law | ZeroQuest Implementation |
|---|---|
| **O(1) Access** | Any quest at any coordinate computed instantly, no iteration |
| **Parallelism** | Each quest depends only on its own coordinate — stages are independent |
| **Coherence** | Quest type distribution is deliberately non-coherent (uniform hash); narrative vocabulary can use coherent noise for regional tone flavour (optional) |
| **Hierarchy** | DATA_DUMP stages derive from parent quest seed; profile registry is a static hierarchy |
| **Determinism** | All hash operations use xxhash32 with fixed salts; no platform hash, no time, no random |

### 10.3 Anti-Patterns (Forbidden in ZeroQuest)

```typescript
// BAD: Store quest data
questDatabase.set(coord, generateQuest(...))    // Never store — regenerate

// BAD: Sequential stage assignment
stages[i] = stages[i-1].next()                  // Child seeds from parent hash only

// BAD: Random timer
timer = Math.random() * 120                     // All values from hash

// BAD: Global stage counter
stagesSeen++                                    // No global mutable state
```

---

## 11. Determinism Verification

```typescript
// Same coordinate, same profile → identical quest
const a = generateQuest(120, 45, 0);
const b = generateQuest(120, 45, 0);
assert(a.briefing === b.briefing);
assert(a.questType === b.questType);
assert(a.timer === b.timer);

// Adjacent coordinate → different quest type (likely)
const c = generateQuest(121, 45, 0);
// c may or may not match a.questType — no guarantee, but most will differ

// DATA_DUMP stages always same coordinates from same origin
const d = generateQuest(300, 112, 0);
const e = generateQuest(300, 112, 0);
assert(d.stages[1].coordinates[0] === e.stages[1].coordinates[0]);

// World seed shifts everything
const f = generateQuest(120, 45, 0, 0);
const g = generateQuest(120, 45, 0, 99999);
assert(f.questType !== g.questType || f.briefing !== g.briefing); // Almost certain
```

---

## 12. Extended Features (Optional)

### 12.1 Regional Tone Bias

Using ZeroBytes coherent noise, geographic regions of the game world can be biased toward certain quest types (e.g., outer cells are mostly STRONGHOLD; central corridors favour DATA_DUMP):

```typescript
const regionalBias = coherent_value(x * 0.001, y * 0.001, worldSeed + 500, 3);
const biasedIndex  = Math.floor(regionalBias * 4);  // 0-3 biased by region
```

This overlays geographic narrative texture without breaking per-coordinate determinism.

### 12.2 Difficulty Scaling

Difficulty tier is derived from coordinate distance from world centre (or any defined anchor):

```typescript
const dist = Math.sqrt(x*x + y*y + z*z);
const tier = Math.min(5, Math.floor(dist / 1000));  // Tier 0–5
```

Combined with the hash-derived difficulty modifier, this produces a smooth but non-linear difficulty curve.

### 12.3 World Seed as Campaign Variable

The `worldSeed` parameter means the same physical map generates entirely different quests per campaign or playthrough seed. This gives ZeroQuest full roguelite campaign support at zero storage cost.

### 12.4 ZeroResponse NPC Integration

When an NPC at coordinate `(nx, ny)` gives a quest briefing, ZeroResponse and ZeroQuest share the position:

```typescript
// NPC speech at their position
const npcSpeech = generateSpeech(nx, ny, "commander", 0);

// Quest at their position (quest emanates from NPC location)
const quest = generateQuest(nx, ny, 0);

// Same coordinate — different systems, complementary outputs
```

The NPC's speech profile can reference quest-type vocabulary for narrative coherence (e.g., a Ghost NPC at a DATA_DUMP coordinate uses melancholy data-collection language).

---

## 13. Glossary

| Term | Definition |
|---|---|
| **ZeroQuest** | Position-is-quest procedural mission generation system. |
| **Quest Seed** | 32-bit hash derived from objective coordinate. Defines the entire quest universe at that point. |
| **Quest Type** | One of four mission archetypes: PROTECT_HOME, PROTECT_POI, DATA_DUMP, STRONGHOLD. |
| **Quest Profile** | JSON file defining templates and vocabulary pools for all quest narrative layers. |
| **Narrative Layer** | One text output category: briefing, objective label, stage prompt, success, failure, in-mission. |
| **Layer Salt** | Integer offset added to quest seed before hashing each narrative layer, preventing cross-layer collisions. |
| **Stage Seed** | Child seed derived from the parent quest seed plus stage index, following ZeroBytes hierarchy. |
| **DATA_DUMP Chain** | A three-stage quest where stage 2 and 3 coordinates are derived from the origin seed. |
| **ZeroResponse Engine** | The underlying template-fill engine. ZeroQuest calls it with layer-specific seeds and profile sections. |
| **World Seed** | Optional global integer. Shifts the entire quest universe for that world, enabling per-campaign variation. |
| **Paradistro Profile** | World-specific Quest Profile tuned to the Paradistro megastructure lore. |
| **ZeroQuest Profile Builder** | Companion skill that converts lore/source material into a ZeroQuest JSON profile. |
| **Coherent Bias** | Optional regional tone weighting using ZeroBytes coherent noise. Does not break determinism. |

---

## 14. Implementation Checklist

- [ ] Implement `src/quest/engine.ts` — seed derivation, type selection, timer, difficulty
- [ ] Implement `src/quest/narrative.ts` — layer seed derivation, ZeroResponse calls per layer
- [ ] Implement `src/quest/types.ts` — `QuestType`, `QuestProfile`, `QuestResult`, `QuestStage`
- [ ] Implement `src/quest/registry.ts` — profile map, default + named lookup
- [ ] Implement `src/quest/index.ts` — public API: `generateQuest`, `getQuestType`, `getQuestCombinations`
- [ ] Author `ZeroQuest-Default.json` — complete default profile with all four types and all six layers
- [ ] Author `ZeroQuest-Paradistro.json` — Paradistro world profile
- [ ] Implement `zeroquest-profile-builder` skill — accepts lore input, outputs quest JSON profile
- [ ] Integrate with `src/speech/` module — shared coordinate, complementary outputs
- [ ] Write determinism verification suite
- [ ] Performance benchmark: target < 0.5ms per `generateQuest` call

---

*The quest already exists. The coordinate knows it. You only have to ask.*
