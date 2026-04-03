# ZeroQuest-JSX

**Position-is-Quest Procedural Narrative Mission System**

> *The quest already exists. The coordinate knows it. You only have to ask.*

ZeroQuest is a deterministic procedural quest generation system in which the **objective's world position is the mission seed**. Every coordinate in a game world silently carries a complete, reproducible narrative mission — waiting to be read, never needing to be stored.

No database. No save file. No network call. The geometry of the world **is** the mission ledger.

---

## Overview

ZeroQuest is built on two existing ZeroFamily systems:

- **ZeroBytes** — the position-is-seed hash engine (`xxhash32` / `xxhash64`). Provides the deterministic coordinate-to-seed bridge.
- **ZeroResponse** — the template-and-pool procedural text engine. Drives narrative sentence assembly from JSON profiles.

ZeroQuest extends both into a third layer: **quest narrative generation**, where a quest's geographic location determines its type, context, objectives, tone, and all narrative text — generated on demand, identically, every time.

```
questSeed = xxhash32([obj_x, obj_y, obj_z], worldSeed)
```

A quest objective placed at coordinate `(120, 45, 0)` will **always** generate the same quest type, the same briefing text, the same timer, the same success and failure lines — across any platform, any session, any machine.

---

## How It Works

### The Core Principle

In ZeroBytes, the coordinate **is** the universe at that point. ZeroQuest applies this recursively to the mission layer. Quest type is selected by mapping the lower 2 bits of the position hash across four archetypes, producing a roughly equal but fully deterministic distribution across the world:

```typescript
const typeIndex = questSeed % 4;
const QUEST_TYPES = ["PROTECT_HOME", "PROTECT_POI", "DATA_DUMP", "STRONGHOLD"];
const questType = QUEST_TYPES[typeIndex];
```

### Architecture

ZeroQuest inherits from — and does not replace — ZeroBytes and ZeroResponse. It adds a new layer above them:

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

### Seed Derivation Chain

```typescript
// Step 1: Position to quest seed
questSeed = xxhash32([x, y, z], worldSeed)

// Step 2: Quest type from seed
questType = QUEST_TYPES[questSeed % 4]

// Step 3: Per-layer seeds (salted to prevent cross-layer collision)
briefingSeed        = xxhash32([questSeed, 1000], 0)
objectiveLabelSeed  = xxhash32([questSeed, 2000], 0)
stageSeed_0         = questSeed
stageSeed_1         = xxhash32([questSeed, 1, 0], worldSeed)
stageSeed_2         = xxhash32([questSeed, 2, 0], worldSeed)
successSeed         = xxhash32([questSeed, 4000], 0)
failureSeed         = xxhash32([questSeed, 5000], 0)
inMissionSeed       = xxhash32([questSeed, 6000], 0)

// Step 4: Timer derivation (timed quest types only)
timerBase = hash_to_float(xxhash32([questSeed, 9000], 0))
timer = Math.floor(timerMin + timerBase * (timerMax - timerMin))
```

---

## Quest Types

Four archetypes form the default ZeroQuest taxonomy. Quest type is derived deterministically from the position seed — no authorial placement required.

### PROTECT_HOME
Defend your spawn location. Clear all enemies before the timer expires. The objective coordinate is treated as the home/spawn coordinate. Urgent, defensive, personal in tone. Timer range: 60–180 seconds.

### PROTECT_POI
Reach a remote Point of Interest and hold it against waves. The coordinate fully defines the location identity — same coordinate, same POI narrative, always. Timer range: 90–240 seconds.

### DATA_DUMP
Visit a chain of three sequential locations, defending each for a fixed window before advancing. Stage 2 and 3 coordinates are derived from the origin seed using the ZeroBytes hierarchy pattern:

```typescript
stage1Seed = questSeed
stage2Seed = position_hash(stage1Seed, 1, 0, worldSeed)
stage3Seed = position_hash(stage1Seed, 2, 0, worldSeed)
```

Per-stage timer: 30–60 seconds. Methodical, technical tone.

### STRONGHOLD
Assault an enemy fortress. Eliminate all defenders and the boss-tier commander at the objective coordinate. No timer — difficulty and boss tier are derived from the coordinate seed. Oppressive, confrontational tone.

---

## The Quest Profile System

ZeroQuest uses **Quest Profiles** — JSON files that define templates and vocabulary pools for every narrative layer. The ZeroResponse engine fills the templates; ZeroQuest selects which layer to invoke based on quest type and stage.

### Narrative Layers

| Layer | When Used | Output |
|---|---|---|
| `briefing` | Quest accept screen | 1–3 sentence mission description |
| `objective_label` | HUD / map pin | Short label (5–10 words) |
| `stage_prompt` | DATA_DUMP stage activation | Per-stage instruction string |
| `success` | Mission complete screen | Reward/resolution line |
| `failure` | Mission fail screen | Consequence/tone line |
| `in_mission` | Mid-mission NPC chatter | Radio line during mission |

### Profile Schema

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

Two profiles ship with ZeroQuest out of the box:

- **ZeroQuest-Default.json** — Genre-neutral base profile, suitable for sci-fi, fantasy, and contemporary settings.
- **ZeroQuest-Paradistro.json** — World-specific profile tuned to the Paradistro megastructure lore: a vast Dyson Sphere inhabited by machine descendants of humanity, where missions unfold within nested shell cells and briefings reference the contested covenant of the Supreme Intelligence.

---

## Public API

### `generateQuest(x, y, z, worldSeed?, profileName?): QuestResult`

The single public entry point. Returns the complete quest for any world coordinate.

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
```

```typescript
// Protect POI at grid (120, 45, 0) — default profile
const quest = generateQuest(120, 45, 0);
// quest.questType → "PROTECT_POI"
// quest.timer     → 142 (deterministic from seed, always)

// Same coordinate, Paradistro lore profile
const quest2 = generateQuest(120, 45, 0, 0, "Paradistro");
// quest2.briefing → "Shell cell seven-seven-niner reports hostile incursion.
//                   the rogue units do not acknowledge the covenant.
//                   restore shell integrity. Man built this. We maintain it."

// Data Dump — generates chain of 3 stage coordinates
const quest3 = generateQuest(300, 112, 0);
// quest3.questType          → "DATA_DUMP"
// quest3.stages[0].coordinates → [300, 112, 0]
// quest3.stages[1].coordinates → [derived deterministically from seed]
// quest3.stages[2].coordinates → [derived deterministically from seed]
```

### `getQuestType(x, y, z, worldSeed?): QuestType`

Lightweight query — returns only the quest type for a coordinate. Useful for map rendering (show quest type icons without generating full narrative).

### `getQuestCombinations(profileName?): Record<QuestType, number>`

Returns the total unique narrative combinations per quest type for a given profile. Useful for design validation.

---

## Extended Features

### Regional Tone Bias

Using ZeroBytes coherent noise, geographic regions can be biased toward certain quest types without breaking per-coordinate determinism:

```typescript
const regionalBias = coherent_value(x * 0.001, y * 0.001, worldSeed + 500, 3);
const biasedIndex  = Math.floor(regionalBias * 4);
```

### Difficulty Scaling

Difficulty tier is derived from coordinate distance from the world centre, producing a smooth but non-linear difficulty curve:

```typescript
const dist = Math.sqrt(x*x + y*y + z*z);
const tier = Math.min(5, Math.floor(dist / 1000));  // Tier 0–5
```

### World Seed as Campaign Variable

The `worldSeed` parameter means the same physical map generates entirely different quests per campaign playthrough. Full roguelite campaign support at zero storage cost.

### ZeroResponse NPC Integration

When an NPC at a coordinate gives a quest briefing, ZeroResponse and ZeroQuest share the position — same coordinate, complementary outputs, narrative coherence for free:

```typescript
const npcSpeech = generateSpeech(nx, ny, "commander", 0);
const quest     = generateQuest(nx, ny, 0);
```

---

## ZeroBytes Five Laws Compliance

| Law | ZeroQuest Implementation |
|---|---|
| **O(1) Access** | Any quest at any coordinate computed instantly — no iteration, no lookup |
| **Parallelism** | Each quest depends only on its own coordinate — stages are independent |
| **Coherence** | Quest type distribution is deliberately non-coherent (uniform hash); narrative vocabulary can optionally use coherent noise for regional tone |
| **Hierarchy** | DATA_DUMP stages derive from parent quest seed; profile registry is a static hierarchy |
| **Determinism** | All hash operations use xxhash32 with fixed salts — no platform hash, no time, no random |

### Anti-Patterns (Never Do These)

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

## Module Layout

```
src/
├── quest/
│   ├── index.ts                      ← Public API: generateQuest()
│   ├── engine.ts                     ← Quest seed derivation + type selection
│   ├── narrative.ts                  ← Calls ZeroResponse engine for text
│   ├── types.ts                      ← QuestType, QuestProfile, QuestResult
│   ├── registry.ts                   ← Profile map (default + custom)
│   └── profiles/
│       ├── ZeroQuest-Default.json    ← Base narrative profile
│       └── ZeroQuest-Paradistro.json ← Paradistro world-specific profile
│
└── speech/                           ← ZeroResponse module (existing)
```

---

## ZeroFamily Lineage

ZeroQuest is part of the **ZeroFamily** of deterministic procedural generation systems:

- **ZeroBytes** — Position-is-seed spatial hash engine
- **ZeroResponse** — Template-and-pool procedural text from position seeds
- **ZeroQuest** — Position-is-quest mission narrative generation *(this repo)*
- **Zero-Temporal** — Coordinate+epoch-is-seed world-time system
- **Zero-Quadratic** — Pair-is-seed relational procedural generation
- **Zero-Field** — Continuous influence field generation

---

## 📚 Citation

### Academic Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{zeroquest_jsx,
  title = {ZeroQuest-JSX: Position-is-Quest Procedural Narrative Mission System},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/ZeroQuest-JSX},
  version = {1.0.0}
}
```

### Donate

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
