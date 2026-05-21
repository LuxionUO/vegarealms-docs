# Achievement System in CORE (SCP-driven) — Design Study and Implementation Guide

This document proposes a **native achievement system** inside the Sphere core, fully configurable through SCP, with progress persisted per player and easy exposure in gumps.

It is designed for:
- server owners who configure achievements in script,
- core maintainers who implement engine-level support,
- script developers who build UI and custom completion logic.

---

## 1) Goals

### Functional goals
1. Define achievements in SCP (`[ACHIEVEMENT <id>]`).
2. Track per-player progress (`actual`) and completion state (`completed`).
3. Support grouped display/filter by category in gumps.
4. Execute custom logic on completion (`ON=@Completion`).
5. Reward points/items automatically.

### Non-functional goals
1. Fast lookup/update (event-heavy scenarios like kills).
2. Save/load-safe persistence of player progress.
3. Backward compatibility and incremental rollout.
4. Extensible to future types (craft, explore, quests, skills, pvp).

---

## 2) SCP schema proposal

Your idea is good. Keep it item-like, but define a dedicated section type for clarity.

### Base syntax

```scp
[ACHIEVEMENT 1]
NAME=Orcs Master
TYPE=KILL
OBJECT=SINGLE
CATEGORY=Hunting
TARGET=500
POINTS=10
ITEMS=i_gold 500, i_platemail_chest 1
ENABLED=1
VISIBLE=1

// Optional filters depending on TYPE/OBJECT
CHARDEF=c_orc
TAGFILTER=orc, humanoid

ON=@Completion
    // Custom completion hook for this achievement
    // example:
    // SRC.YOUNG=0
```

### Recommended keys

- `NAME` (string): display name.
- `DESC` (string, optional): long description.
- `TYPE` (enum): `KILL`, `CRAFT`, `GATHER`, `SKILL`, `EXPLORE`, `QUEST`, `PVP`, ...
- `OBJECT` (enum):
  - `SINGLE` = one target type (eg one chardef)
  - `MULTI` = subsection/list/filter-based set
- `CATEGORY` (string): group label (for gump filtering).
- `TARGET` (int): required amount.
- `POINTS` (int): score reward.
- `ITEMS` (list): reward items.
- `ENABLED` (bool, default 1).
- `VISIBLE` (bool, default 1).
- `REPEATABLE` (bool, default 0).
- `RESET_DAYS` (int, optional): if repeatable.
- `ICON` (int/string, optional): for UI.
- `SORT` (int, optional): order in category.

### Target specification strategy

To avoid many bespoke fields, standardize:

- `TARGETDEF` (single identifier for SINGLE).
- `SUBSECTION` (for `OBJECT=MULTI`): direct subsection name to match, especially from `[CHARDEF x]`.
- `TARGET_EXPR` (advanced filter expression; optional future).

For KILL examples:

```scp
[ACHIEVEMENT 2]
NAME=Lizard Slayer
TYPE=KILL
OBJECT=SINGLE
CATEGORY=Hunting
TARGET=200
TARGETDEF=c_lizardman
POINTS=5
```

```scp
[ACHIEVEMENT 3]
NAME=Undead Purge
TYPE=KILL
OBJECT=MULTI
CATEGORY=Hunting
TARGET=1000
SUBSECTION=undead_tier1
POINTS=20
```

---

## 3) Per-player data model (props/tags)

Your proposed access shape is valid:

- `src.achievement.<ID>.actual`
- `src.achievement.<ID>.completed`

Add a few more fields for robust lifecycle:

- `src.achievement.<ID>.actual` (int)
- `src.achievement.<ID>.completed` (0/1)
- `src.achievement.<ID>.completed_at` (timestamp)
- `src.achievement.<ID>.claimed` (0/1, if rewards are claimable separately)
- `src.achievement.<ID>.version` (int, for migration)

### Engine-level internal representation

In C++ core, use compact struct/map, then expose as dynamic props:

```text
unordered_map<achievement_id, AchievementProgress>
AchievementProgress {
  int actual;
  bool completed;
  int64 completed_at;
  bool claimed;
}
```

Persist into character save as tags/serialized block.

---

## 4) Category management (important for gump filters)

Your concern about categories is central. Best approach:

## A. Dynamic categories from achievement definitions
- Parse all `CATEGORY` strings during load.
- Build normalized category index (case-insensitive key, display label preserved).

## B. Optional category metadata section

```scp
[ACHIEVEMENT_CATEGORY hunting]
NAME=Hunting
ICON=21045
SORT=10
VISIBLE=1
```

If absent, fallback to raw `CATEGORY` string.

## C. Core API for UI filtering
Expose script functions:

- `SRC.ACH_LIST_CATEGORIES` → returns categories available to player.
- `SRC.ACH_LIST_BY_CATEGORY <catKey>` → returns achievement IDs sorted.
- `SRC.ACH_GET <id>.<field>` → unified read access.


### Global read access from SERV

Achievements should also be queryable globally from scripts with:

- `SERV.ACHIEVEMENT.<ID>.PROP`

Where `PROP` can be fields such as `NAME`, `DESC`, `TYPE`, `CATEGORY`, `TARGET`, `POINTS`, etc.

This avoids duplicating metadata in `DEFNAME` blocks and gives a single canonical source in the achievement registry.

### Gump flow example
1. Build left panel with categories.
2. On click category, call `ACH_LIST_BY_CATEGORY`.
3. For each achievement render:
   - name
   - progress `actual/target`
   - completed badge

---

## 5) Event pipeline and progress update

Core should own update logic for performance and consistency.

### Suggested pipeline
1. Game event occurs (`OnKill`, `OnCraft`, etc.).
2. Achievement manager gets `(player, event_type, context)`.
3. Retrieve candidate achievements by `TYPE` (+ indexed filters).
4. Match target(s):
   - `OBJECT=SINGLE`: compare specific target (`TARGETDEF`).
   - `OBJECT=MULTI`: compare killed mob subsection (eg `REF1.SUBSECTION`) with achievement `SUBSECTION`.
5. Increment `actual` safely (clamped to `TARGET`).
6. If threshold reached and not completed:
   - set completed fields,
   - grant rewards,
   - fire `ON=@Completion` script,
   - notify client.

### Performance note
Do **not** iterate all achievements globally on each kill. Maintain indices:

- by `TYPE`
- by `TARGETDEF`
- by `SUBSECTION` membership cache

---

## 6) Completion semantics and reward strategy

Two valid models:

1. **Auto-claim**: rewards granted immediately on completion.
2. **Manual claim**: completion unlocks claim button in gump.

Recommended: support both with `CLAIM_MODE=AUTO|MANUAL`.

For manual claim:
- set `completed=1`, `claimed=0`,
- gump enables claim button,
- on claim, grant items/points, set `claimed=1`.

---

## 7) Script hooks

### Per-achievement hook
- `ON=@Completion` in `[ACHIEVEMENT <id>]`.

### Global hooks (recommended)
- `ON=@AchievementProgress`
- `ON=@AchievementCompleted`
- `ON=@AchievementClaimed`

Pass args:
- achievement id
- old/new progress
- category
- type

This enables global systems (announcements, analytics, anti-abuse).

---

## 8) Proposed CORE implementation steps

1. **Parser layer**
   - Add SCP section parser for `ACHIEVEMENT` (+ optional `ACHIEVEMENT_CATEGORY`).
   - Validate keys/types; log meaningful warnings.

2. **Registry layer**
   - Build immutable runtime registry of achievements.
   - Build indices by type/category/target.

3. **Player progress layer**
   - Add progress container to player object.
   - Serialize/deserialise from saves.
   - Expose scriptable props `achievement.<id>.*`.

4. **Event integration layer**
   - Wire gameplay events into achievement manager.
   - Implement matchers for first TYPE (`KILL`) then expand.

5. **Reward + hooks layer**
   - Items/points grant routines.
   - Completion and claim hooks.

6. **Script API layer for gumps**
   - Category list, filtered list, detail getters.

7. **Admin/debug tools**
   - Commands: reset player achievement, set progress, force complete.

8. **Testing**
   - Unit tests for parser/matching.
   - Integration tests for save/load and duplicate-completion prevention.

---

## 9) Anti-abuse and edge cases

- Ignore progress if player kills invalid sources (self, party exploit, summoned depending policy).
- Clamp `actual <= TARGET`.
- Ensure completion is idempotent (no double rewards).
- Handle achievement definition changes:
  - if `TARGET` changes, decide migration policy via `version`.
- Handle removed achievement IDs gracefully in old save data.

---

## 10) Migration and compatibility plan

- Phase 1: read-only parse + debug command to inspect registry.
- Phase 2: KILL type only + auto-claim.
- Phase 3: categories API + gump integration.
- Phase 4: manual claim + extra types.

Keep old servers safe by defaulting `ENABLED=0` unless configured, or gate by feature flag:

```scp
[SPHERE]
FEATURE_ACHIEVEMENTS=1
```

---

## 11) Full SCP examples (expanded)

### Example A — single target kill

```scp
[ACHIEVEMENT 10]
NAME=Orcs Master
DESC=Kill 500 orcs
TYPE=KILL
OBJECT=SINGLE
CATEGORY=Hunting
TARGET=500
TARGETDEF=c_orc
POINTS=10
ITEMS=i_gold 500, i_platemail_chest 1
CLAIM_MODE=AUTO

ON=@Completion
    SERV.LOG Orcs Master completed by <SRC.NAME>
```

### Example B — multi target kill

```scp
[ACHIEVEMENT 11]
NAME=Dungeon Cleaner
TYPE=KILL
OBJECT=MULTI
CATEGORY=Hunting
TARGET=1000
SUBSECTION=undead_tier1
POINTS=25
CLAIM_MODE=MANUAL

// Example chardefs tagged with a common subsection
[CHARDEF c_skeleton]
SUBSECTION=undead_tier1

[CHARDEF c_zombie]
SUBSECTION=undead_tier1

[CHARDEF c_lich]
SUBSECTION=undead_tier1
```

### Example C — category metadata

```scp
[ACHIEVEMENT_CATEGORY hunting]
NAME=Hunting
ICON=5601
SORT=10
VISIBLE=1

[ACHIEVEMENT_CATEGORY crafting]
NAME=Crafting
ICON=4020
SORT=20
VISIBLE=1
```

---

## 12) Draft AGENTS.md instructions for Claude/Codex contributors

Copy this into `AGENTS.md` at repo root (adapt to your workflow):

```md
# AGENTS — Achievement System Contribution Rules

## Scope
These rules apply to the whole repository.

## Objective
Implement a native SCP-driven Achievement System in CORE with:
- definition parsing,
- player progress persistence,
- completion/claim logic,
- category filtering API for gumps.

## Mandatory design constraints
1. Achievement definitions come from `[ACHIEVEMENT <id>]` sections.
2. Player script access must support:
   - `SRC.ACHIEVEMENT.<ID>.ACTUAL`
   - `SRC.ACHIEVEMENT.<ID>.COMPLETED`
3. Global metadata access must support `SERV.ACHIEVEMENT.<ID>.PROP`.
4. For `OBJECT=MULTI`, matching should use mob `SUBSECTION` from `[CHARDEF x]` (eg `REF1.SUBSECTION`).
5. Category-based listing must be first-class for UI/gump filtering.
6. Completion must be idempotent (no duplicate rewards).
7. Parser warnings must be explicit (invalid TYPE, missing TARGET, etc.).

## Data and API conventions
- Use normalized category keys internally (case-insensitive), preserve display name.
- Clamp progress to target.
- Expose script helpers for:
  - category list
  - achievements by category
  - achievement field reads

## Performance constraints
- Do not scan all achievements on every event.
- Maintain indices by TYPE and target identifier.

## Compatibility constraints
- Must not break existing character save loading.
- Unknown/removed achievements in save files should be ignored safely.

## Implementation phases
1. Parser + registry
2. Player progress save/load
3. KILL event integration
4. Completion hooks + rewards
5. Gump-facing script API

## Validation checklist before merge
- Build passes.
- New parser tests pass.
- Save/load test for achievement progress passes.
- Duplicate-completion reward test passes.
- Category filtering test passes.
```

---

## 13) Recommended development backlog for the CORE team

1. Define enums and structures (TYPE, OBJECT, reward model).
2. Implement parser and validation diagnostics.
3. Implement registry + lookup indices.
4. Implement per-player progress storage and serialization.
5. Wire first event: kill tracking.
6. Implement completion + claim flow and hooks.
7. Implement script API and gump integration helpers.
8. Add tests and migration notes.
9. Publish scripting documentation page with examples.

---

## 14) Final recommendation

Yes: your original idea is solid, and **category-aware indexing** is the key for usable gumps.

If you want, the next concrete step is to produce a **technical mapping document** from this design to actual files/classes in `src/` (where parser, player props, save format, and event hooks should be touched), so whoever is maintaining CORE can implement in small mergeable PRs.