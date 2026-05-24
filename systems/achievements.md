# Achievement System in CORE (SCP-driven) — Design Study and Implementation Guide

This document proposes a native achievement system inside the Sphere core, fully configurable through SCP, with progress persisted per player and easy exposure in gumps.

It is designed for:

- server owners who configure achievements in script
- core maintainers who implement engine-level support
- script developers who build UI and custom completion logic

## 1) Goals

### Functional goals

1. Define achievements in SCP using `[ACHIEVEMENT <id>]`.
2. Track per-player progress using `actual` and completion state using `completed`.
3. Support grouped display and filtering by category in gumps.
4. Execute custom logic on completion using `ON=@Completion`.
5. Reward points and items automatically.
6. Optionally track server-wide completion history, including who completed an achievement and when.

### Non-functional goals

1. Fast lookup and update, especially in event-heavy scenarios like kills.
2. Save and load-safe persistence of player progress.
3. Backward compatibility and incremental rollout.
4. Extensible to future types such as craft, explore, quests, skills and PvP.
5. Server-side analytics support without requiring expensive scans of every character save file.

## 2) SCP schema proposal

The achievement system should use a dedicated SCP section type for clarity.

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

- `NAME`: display name.
- `DESC`: optional long description.
- `TYPE`: achievement type, for example `KILL`, `CRAFT`, `GATHER`, `SKILL`, `EXPLORE`, `QUEST`, `PVP`.
- `OBJECT`: target matching mode.
- `CATEGORY`: group label for gump filtering.
- `TARGET`: required amount.
- `POINTS`: score reward.
- `ITEMS`: reward items.
- `ENABLED`: whether the achievement is active.
- `VISIBLE`: whether the achievement is visible in gumps.
- `REPEATABLE`: whether the achievement can be completed more than once.
- `RESET_DAYS`: optional reset cycle for repeatable achievements.
- `ICON`: optional icon for UI.
- `SORT`: optional order in category.
- `CLAIM_MODE`: reward claim behaviour, for example `AUTO` or `MANUAL`.
- `TRACK_COMPLETION`: whether server-side completion history should be stored.
- `ANNOUNCE_FIRST`: whether the first completion should be announced.
- `GLOBAL_RANKING`: whether the achievement can appear in global rankings.
- `SHOW_COMPLETION_COUNT`: whether gumps can display how many players completed it.

### Target specification strategy

To avoid too many bespoke fields, target matching should be standardised.

Recommended fields:

- `TARGETDEF`: single identifier for `OBJECT=SINGLE`.
- `SUBSECTION`: subsection or group identifier for `OBJECT=MULTI`.
- `TARGET_EXPR`: optional advanced filter expression for future use.

For kill achievements:

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

## 3) Per-player data model

Player progress should be stored per character.

Suggested script access:

```text
SRC.ACHIEVEMENT.<ID>.ACTUAL
SRC.ACHIEVEMENT.<ID>.COMPLETED
SRC.ACHIEVEMENT.<ID>.COMPLETED_AT
SRC.ACHIEVEMENT.<ID>.CLAIMED
SRC.ACHIEVEMENT.<ID>.VERSION
```

Recommended fields:

- `actual`: current progress amount.
- `completed`: whether the achievement has been completed.
- `completed_at`: timestamp of completion.
- `claimed`: whether rewards have been claimed.
- `version`: achievement version used for migrations.

### Engine-level internal representation

In the C++ core, a compact player-side structure should be used and then exposed through dynamic script properties.

Suggested internal structure:

```text
unordered_map<achievement_id, AchievementProgress>

AchievementProgress {
  int actual;
  bool completed;
  int64 completed_at;
  bool claimed;
  int version;
}
```

This data should be serialised safely into character saves.

## 4) Server-side completion tracking

In addition to per-player progress, the achievement system should maintain a server-side completion registry.

Per-player progress is useful to know the current state of a specific character, for example:

```text
SRC.ACHIEVEMENT.<ID>.ACTUAL
SRC.ACHIEVEMENT.<ID>.COMPLETED
SRC.ACHIEVEMENT.<ID>.COMPLETED_AT
SRC.ACHIEVEMENT.<ID>.CLAIMED
```

However, per-player data alone is not enough for global queries such as:

- who completed a specific achievement
- when the achievement was completed
- who completed it first
- how many players completed it
- whether a specific character has already completed it
- how many times a repeatable achievement has been completed

For this reason, the core should maintain a persistent global completion registry.

### Completion record structure

Suggested internal structure:

```text
AchievementCompletionRecord {
  achievement_id
  char_uid
  account_name
  char_name
  completed_at
  claimed_at
  repeat_index
  points_awarded
  rewards_claimed
}
```

For non-repeatable achievements, the following combination must be unique:

```text
achievement_id + char_uid
```

For repeatable achievements, the system should allow multiple completion records by using:

```text
achievement_id + char_uid + repeat_index
```

This allows the server to keep a proper history of repeated completions without overwriting previous records.

### Suggested script access

Global achievement metadata should remain accessible through:

```text
SERV.ACHIEVEMENT.<ID>.PROP
```

The completion registry should extend this with server-wide completion fields:

```text
SERV.ACHIEVEMENT.<ID>.COMPLETION_COUNT
SERV.ACHIEVEMENT.<ID>.FIRST_COMPLETED_BY
SERV.ACHIEVEMENT.<ID>.FIRST_COMPLETED_AT
SERV.ACHIEVEMENT.<ID>.LAST_COMPLETED_BY
SERV.ACHIEVEMENT.<ID>.LAST_COMPLETED_AT
```

Optional helper functions:

```text
SERV.ACH_COMPLETION_COUNT <achievement_id>
SERV.ACH_COMPLETED_BY <achievement_id> <char_uid>
SERV.ACH_FIRST_COMPLETED_BY <achievement_id>
SERV.ACH_LAST_COMPLETED_BY <achievement_id>
SERV.ACH_COMPLETIONS <achievement_id>
```

These helpers would allow scripts and gumps to query global achievement history without manually scanning player data.

### Completion pipeline

When an achievement is completed, the core should perform the following steps:

```text
Player reaches achievement target
↓
Set SRC.ACHIEVEMENT.<ID>.COMPLETED = 1
↓
Set SRC.ACHIEVEMENT.<ID>.COMPLETED_AT = current timestamp
↓
Create or update server-side completion record
↓
Grant rewards or unlock manual claim
↓
Fire @AchievementCompleted and @Completion hooks
↓
Optional announcement or analytics update
```

### Idempotency rules

Completion tracking must be idempotent.

A non-repeatable achievement must never:

- create duplicate completion records for the same character
- grant duplicate points
- grant duplicate reward items
- fire completion hooks multiple times

Before inserting a new completion record, the core should check whether the same character has already completed the same non-repeatable achievement.

For repeatable achievements, the system should create a new record only when a valid new completion cycle is reached.

### Optional SCP keys

Achievements may optionally define whether global completion tracking is enabled:

```scp
[ACHIEVEMENT 10]
NAME=Orcs Master
TYPE=KILL
TARGET=500
TARGETDEF=c_orc
TRACK_COMPLETION=1
ANNOUNCE_FIRST=1
GLOBAL_RANKING=1
SHOW_COMPLETION_COUNT=1
```

Recommended keys:

- `TRACK_COMPLETION`: enables server-side completion history.
- `ANNOUNCE_FIRST`: announces the first character to complete the achievement.
- `GLOBAL_RANKING`: allows the achievement to appear in global achievement rankings.
- `SHOW_COMPLETION_COUNT`: allows gumps to display how many players completed the achievement.

Default behaviour should be:

```text
TRACK_COMPLETION=1
ANNOUNCE_FIRST=0
GLOBAL_RANKING=0
SHOW_COMPLETION_COUNT=1
```

### Use cases

Server-side completion tracking enables several useful features:

- first player to complete an achievement
- rare achievement detection
- achievement leaderboards
- player profile badges
- global completion percentages
- seasonal achievement history
- admin audit tools
- server-wide progression statistics
- public gumps showing achievement popularity

### Admin and debug commands

The core should also provide admin and debug commands for maintenance:

```text
ACH_COMPLETIONS <achievement_id>
ACH_RESET_COMPLETION <char_uid> <achievement_id>
ACH_FORCE_COMPLETE <char_uid> <achievement_id>
ACH_REMOVE_COMPLETION <char_uid> <achievement_id>
```

These commands are useful for testing, fixing corrupted data, handling abuse cases, or migrating achievement definitions.

### Persistence

The completion registry must be saved independently from individual character progress.

This prevents global achievement history from being lost or becoming expensive to reconstruct by scanning every character save file.

Possible persistence strategies:

```text
achievement_completions block in world save
```

or, if database support is available:

```text
achievement_completions table
```

Suggested table-style structure:

```text
achievement_id
char_uid
account_name
char_name
completed_at
claimed_at
repeat_index
points_awarded
rewards_claimed
```

Character progress and global completion history should stay in sync, but they should serve different purposes:

```text
Character progress = personal achievement state
Server completion registry = global achievement history
```

### Final recommendation

Server-side completion tracking should be treated as a core part of the achievement system, not as a later optional extension.

Per-player progress is enough to display a character's own achievements, but it is not enough for global statistics, first completions, rankings, audits, or server-wide gumps.

The system should therefore persist both:

```text
SRC.ACHIEVEMENT.<ID>.*          // personal progress
SERV.ACHIEVEMENT.<ID>.*         // global achievement metadata and completion history
```

## 5) Category management

Categories are important for gump filters and player-facing organisation.

### Dynamic categories from achievement definitions

The core should parse all `CATEGORY` strings during load.

It should build a normalised category index using a case-insensitive internal key while preserving the display label.

Example:

```text
Hunting -> internal key: hunting
Crafting -> internal key: crafting
```

### Optional category metadata section

Category metadata may be defined through a dedicated section.

```scp
[ACHIEVEMENT_CATEGORY hunting]
NAME=Hunting
ICON=21045
SORT=10
VISIBLE=1
```

If no category metadata exists, the system should fall back to the raw `CATEGORY` string from the achievement definition.

### Core API for UI filtering

The core should expose script helpers for gumps.

Suggested helpers:

```text
SRC.ACH_LIST_CATEGORIES
SRC.ACH_LIST_BY_CATEGORY <catKey>
SRC.ACH_GET <id>.<field>
```

### Global read access from SERV

Achievements should also be queryable globally from scripts.

Suggested syntax:

```text
SERV.ACHIEVEMENT.<ID>.PROP
```

Where `PROP` can be:

```text
NAME
DESC
TYPE
OBJECT
CATEGORY
TARGET
POINTS
ITEMS
ENABLED
VISIBLE
REPEATABLE
CLAIM_MODE
COMPLETION_COUNT
FIRST_COMPLETED_BY
FIRST_COMPLETED_AT
LAST_COMPLETED_BY
LAST_COMPLETED_AT
```

This avoids duplicating metadata in `DEFNAME` blocks and gives scripts one canonical source of truth.

### Gump flow example

1. Build the left panel with categories.
2. On category click, call `ACH_LIST_BY_CATEGORY`.
3. For each achievement, render:
   - name
   - description
   - progress
   - target
   - completed badge
   - claim button if manual claim is enabled
   - optional global completion count

## 6) Event pipeline and progress update

The core should own progress update logic for performance and consistency.

### Suggested pipeline

1. A game event occurs, such as `OnKill`, `OnCraft`, `OnGather`, or `OnExplore`.
2. The achievement manager receives the player, event type and event context.
3. The manager retrieves candidate achievements by indexed type and filters.
4. The manager checks whether the event matches the achievement target.
5. The manager increments `actual`, clamped to `TARGET`.
6. If the target is reached and the achievement is not completed:
   - set `completed`
   - set `completed_at`
   - create or update server-side completion record
   - grant rewards or unlock claim
   - fire completion hooks
   - notify the client if needed

### Kill matching example

For `OBJECT=SINGLE`:

```text
event.target_def == achievement.TARGETDEF
```

For `OBJECT=MULTI`:

```text
event.target_subsection == achievement.SUBSECTION
```

### Performance note

The core must not iterate all achievements globally on every event.

It should maintain indices such as:

- by `TYPE`
- by `TARGETDEF`
- by `SUBSECTION`
- by `CATEGORY`
- by repeatable status if needed

This allows high-frequency events such as kills to remain cheap.

## 7) Completion semantics and reward strategy

Two reward models should be supported.

### Auto-claim

Rewards are granted immediately when the achievement is completed.

Example:

```scp
CLAIM_MODE=AUTO
```

### Manual claim

Completion unlocks a claim button in the gump.

Example:

```scp
CLAIM_MODE=MANUAL
```

For manual claim:

```text
completed = 1
claimed = 0
```

When the player claims the reward:

```text
claimed = 1
claimed_at = current timestamp
```

The server-side completion record should also be updated with claim information.

## 8) Script hooks

### Per-achievement hook

Each achievement may define a completion hook.

```scp
ON=@Completion
    SERV.LOG Achievement completed by <SRC.NAME>
```

### Global hooks

The core should support global hooks for achievement systems.

Suggested hooks:

```text
ON=@AchievementProgress
ON=@AchievementCompleted
ON=@AchievementClaimed
```

Suggested arguments:

```text
achievement_id
old_progress
new_progress
target
category
type
claim_mode
```

This enables global systems such as:

- announcements
- analytics
- anti-abuse checks
- seasonal systems
- Discord notifications
- admin logging

## 9) Proposed CORE implementation steps

### 1. Parser layer

Add SCP section parsing for:

```text
[ACHIEVEMENT <id>]
[ACHIEVEMENT_CATEGORY <key>]
```

The parser should validate keys and produce meaningful warnings for:

- invalid `TYPE`
- invalid `OBJECT`
- missing `TARGET`
- missing target specification
- invalid reward item
- invalid category metadata
- duplicate achievement IDs

### 2. Registry layer

Build an immutable runtime achievement registry.

The registry should include:

- achievement definitions
- category definitions
- lookup by ID
- indices by type
- indices by target
- indices by subsection
- indices by category

### 3. Player progress layer

Add a progress container to the player object.

The system should:

- save progress into character saves
- load progress safely
- ignore removed or unknown achievement IDs where needed
- expose scriptable properties through `SRC.ACHIEVEMENT.<ID>.*`

### 4. Server completion registry layer

Add a global completion registry.

The system should:

- store one record per completed non-repeatable achievement per character
- support repeatable completion records through `repeat_index`
- expose global completion data through `SERV.ACHIEVEMENT.<ID>.*`
- avoid duplicate completion records
- avoid duplicate rewards
- persist independently from player progress

### 5. Event integration layer

Wire gameplay events into the achievement manager.

Initial implementation should focus on:

```text
TYPE=KILL
```

Future types can include:

```text
CRAFT
GATHER
SKILL
EXPLORE
QUEST
PVP
```

### 6. Reward and hooks layer

Implement:

- item rewards
- point rewards
- auto-claim
- manual claim
- completion hooks
- global hooks
- claim hooks

### 7. Script API layer for gumps

Expose helpers for:

- category list
- achievements by category
- achievement metadata reads
- player progress reads
- claim actions
- global completion counts
- first and last completion data

### 8. Admin/debug tools

Add admin commands for:

```text
ACH_RESET_PROGRESS <char_uid> <achievement_id>
ACH_SET_PROGRESS <char_uid> <achievement_id> <value>
ACH_FORCE_COMPLETE <char_uid> <achievement_id>
ACH_REMOVE_COMPLETION <char_uid> <achievement_id>
ACH_COMPLETIONS <achievement_id>
```

### 9. Testing

Add tests for:

- parser validation
- registry lookup
- category filtering
- progress increment
- save and load
- duplicate-completion prevention
- reward idempotency
- server-side completion registry
- repeatable achievement completion history

## 10) Anti-abuse and edge cases

The achievement system should handle edge cases carefully.

Recommended rules:

- Ignore invalid kills.
- Ignore self-kills.
- Ignore kills from invalid sources.
- Decide whether party kills count.
- Decide whether summoned creatures count.
- Clamp `actual` to `TARGET`.
- Prevent duplicate completions.
- Prevent duplicate rewards.
- Handle achievement definition changes safely.
- Handle removed achievement IDs gracefully.
- Allow migration through achievement `version`.
- Avoid scanning all players to build global stats.
- Make completion and claim operations atomic where possible.

### Definition changes

If an achievement target changes, the system should have a migration policy.

Example:

```scp
[ACHIEVEMENT 10]
VERSION=2
TARGET=1000
```

Possible policies:

- keep existing completion
- reset progress
- recalculate progress
- mark as legacy
- require admin migration command

## 11) Migration and compatibility plan

### Phase 1

Parser and registry only.

- Parse achievement definitions.
- Parse category definitions.
- Add debug command to inspect registry.
- Do not affect gameplay yet.

### Phase 2

Player progress and save/load.

- Add player progress structure.
- Add save/load support.
- Expose `SRC.ACHIEVEMENT.<ID>.*`.

### Phase 3

Kill achievement support.

- Implement `TYPE=KILL`.
- Add target matching.
- Add progress increment.
- Add completion detection.

### Phase 4

Completion, rewards and hooks.

- Add completion hooks.
- Add point rewards.
- Add item rewards.
- Add auto-claim.
- Add manual claim.

### Phase 5

Server-side completion tracking.

- Add global completion registry.
- Persist completion records.
- Expose completion count.
- Expose first and last completion data.
- Add admin/debug commands.

### Phase 6

Gump-facing API.

- Add category listing.
- Add filtered achievement listing.
- Add claim support.
- Add global completion display support.

### Phase 7

Additional achievement types.

Possible future types:

```text
CRAFT
GATHER
SKILL
EXPLORE
QUEST
PVP
EVENT
SEASONAL
```

### Feature flag

To keep old servers safe, the feature should be gated.

```scp
[SPHERE]
FEATURE_ACHIEVEMENTS=1
```

If disabled, achievement definitions can be ignored or parsed in read-only mode.

## 12) Full SCP examples

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
TRACK_COMPLETION=1
ANNOUNCE_FIRST=1
GLOBAL_RANKING=1
SHOW_COMPLETION_COUNT=1

ON=@Completion
    SERV.LOG Orcs Master completed by <SRC.NAME>
```

### Example B — multi target kill

```scp
[ACHIEVEMENT 11]
NAME=Dungeon Cleaner
DESC=Kill 1000 tier-one undead creatures
TYPE=KILL
OBJECT=MULTI
CATEGORY=Hunting
TARGET=1000
SUBSECTION=undead_tier1
POINTS=25
CLAIM_MODE=MANUAL
TRACK_COMPLETION=1
SHOW_COMPLETION_COUNT=1

ON=@Completion
    SERV.LOG Dungeon Cleaner completed by <SRC.NAME>
```

Example chardefs tagged with a common subsection:

```scp
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

### Example D — repeatable achievement

```scp
[ACHIEVEMENT 20]
NAME=Daily Orc Hunt
DESC=Kill 50 orcs
TYPE=KILL
OBJECT=SINGLE
CATEGORY=Daily
TARGET=50
TARGETDEF=c_orc
POINTS=2
CLAIM_MODE=AUTO
REPEATABLE=1
RESET_DAYS=1
TRACK_COMPLETION=1
SHOW_COMPLETION_COUNT=1
```

For this achievement, the completion registry should allow multiple records per character using:

```text
achievement_id + char_uid + repeat_index
```

## 13) Draft AGENTS.md instructions for Claude/Codex contributors

```md
# AGENTS — Achievement System Contribution Rules

## Scope

These rules apply to the whole repository.

## Objective

Implement a native SCP-driven Achievement System in CORE with:

- definition parsing
- player progress persistence
- server-side completion tracking
- completion and claim logic
- category filtering API for gumps

## Mandatory design constraints

1. Achievement definitions come from `[ACHIEVEMENT <id>]` sections.
2. Player script access must support:
   - `SRC.ACHIEVEMENT.<ID>.ACTUAL`
   - `SRC.ACHIEVEMENT.<ID>.COMPLETED`
   - `SRC.ACHIEVEMENT.<ID>.COMPLETED_AT`
   - `SRC.ACHIEVEMENT.<ID>.CLAIMED`
3. Global metadata access must support:
   - `SERV.ACHIEVEMENT.<ID>.PROP`
4. Server-side completion tracking must support:
   - who completed an achievement
   - when it was completed
   - first completion
   - last completion
   - total completion count
5. For `OBJECT=MULTI`, matching should use mob `SUBSECTION` from `[CHARDEF x]`.
6. Category-based listing must be first-class for UI and gump filtering.
7. Completion must be idempotent.
8. Parser warnings must be explicit.
9. The system must not scan all achievements on every event.
10. The system must not scan all character saves to calculate completion statistics.

## Data and API conventions

- Use normalised category keys internally.
- Preserve category display names.
- Clamp progress to target.
- Keep player progress and server completion history separate.
- Expose script helpers for:
  - category list
  - achievements by category
  - achievement field reads
  - completion count
  - first completed by
  - last completed by
  - completion history

## Performance constraints

- Do not scan all achievements on every event.
- Maintain indices by `TYPE`.
- Maintain indices by target identifier.
- Maintain indices by `SUBSECTION`.
- Maintain indices by category.
- Server-side completion stats should be queryable without scanning player saves.

## Compatibility constraints

- Must not break existing character save loading.
- Unknown or removed achievements in save files should be ignored safely.
- Global completion records for removed achievements should not crash the server.
- Feature should be gated by `FEATURE_ACHIEVEMENTS=1`.

## Implementation phases

1. Parser and registry
2. Player progress save/load
3. Kill event integration
4. Completion hooks and rewards
5. Server-side completion tracking
6. Gump-facing script API
7. Additional achievement types

## Validation checklist before merge

- Build passes.
- Parser tests pass.
- Registry lookup tests pass.
- Save/load test for achievement progress passes.
- Duplicate-completion reward test passes.
- Server completion registry test passes.
- Repeatable achievement completion history test passes.
- Category filtering test passes.
```

## 14) Recommended development backlog for the CORE team

1. Define enums and structures:
   - achievement type
   - object type
   - claim mode
   - reward model
   - completion tracking flags
2. Implement parser and validation diagnostics.
3. Implement registry and lookup indices.
4. Implement per-player progress storage and serialisation.
5. Implement server-side completion registry.
6. Wire first event type: kill tracking.
7. Implement completion, claim flow and hooks.
8. Implement rewards.
9. Implement script API and gump integration helpers.
10. Add admin/debug commands.
11. Add tests and migration notes.
12. Publish scripting documentation page with examples.

## 15) Final recommendation

The achievement system should be implemented as a native core feature with SCP-driven configuration.

The key design principles should be:

- definitions live in `[ACHIEVEMENT <id>]`
- player progress lives on the character
- global completion history lives server-side
- categories are indexed for gumps
- event updates are handled by the core
- completion is idempotent
- rewards cannot be duplicated
- global stats must not require scanning every character save file

The system should persist both personal progress and global completion history.

```text
SRC.ACHIEVEMENT.<ID>.*          // personal progress
SERV.ACHIEVEMENT.<ID>.*         // global achievement metadata and completion history
```

This gives the server owner everything needed for:

- personal achievement gumps
- server-wide achievement statistics
- first completion announcements
- rankings
- rare achievements
- admin audit tools
- seasonal systems
- future expansion into crafting, exploration, quests, skills and PvP