# AI-OOC Narrative & Systems Blueprint

This document operationalizes the narrative and system requirements for the 2D branching-story game inspired by _Person of Interest_. It converts the provided outline into implementable structures, responsibilities, and data schemas for Godot 4.

## 1. Game positioning & core loop
- **Genre**: 2D branching narrative (chapter-based) with interleaved POI case events.
- **Macro loop per chapter**: _Intro → Investigation → Conflict → Choice → Consequence_ with auto-save after each stage.
- **Main arc**: Machine vs. Samaritan AI conflict; ethics-driven outcomes. Chapters default to 1→9 but can branch/insert alternates based on state.

## 2. Characters & data structures
### CharacterDef (Resource/JSON importable)
| Field | Purpose |
| --- | --- |
| `id: StringName` | Stable identifier (e.g., Finch/Reese/Root/Shaw/Carter/Fusco/Bear/Elias/Greer). |
| `display_name_cn/en` | Localized display names. |
| `portrait: Texture2D` | Default portrait. |
| `tags: PackedStringArray` | Roles (protagonist/antagonist/npc/animal). |
| `base_traits: Dictionary` | UI-facing traits (intellect/combat/morality…). |
| `voice_bank` / `sfx_set` (optional) | Audio hooks. |

### Global relationship state (in `GameState`)
- `alive: { "reese": true, "finch": true, ... }`
- `affinity: { "root_shaw": 0, "reese_carter": 0, ... }`
- `trust: { "team": 0, "finch": 0, "root": 0 }`

Principles: dialogues **read/write variables only**; effect resolution lives in centralized gameplay systems (GameState/ChapterManager/encounter controllers), not inline scripts. Missing vars default safely with logged warnings.

## 3. Chapters & storyline structure
### ChapterDef
| Field | Purpose |
| --- | --- |
| `chapter_id` | Numeric key (1..9). |
| `title` | Display title. |
| `entry_conditions` | Expression, e.g., `alive.root == true`. |
| `scenes` | Ordered list of scene ids (intro/investigation/conflict/settlement). |
| `key_choices` | Choice ids for timeline recall. |
| `poi_cases` | Optional POI case ids embedded in chapter. |

### ChapterManager responsibilities
- Evaluate `entry_conditions` against save state to select next chapter (supports parallel/branch chapters).
- Execute the five-segment loop; emit auto-saves after each segment and key choices.
- Allow jump/insert of branch chapters when state dictates.

## 4. Variables & choice system
### GameState (Autoload singleton)
- Stores canonical `vars: Dictionary` (trust/ai_freedom/alive/war_state/poi_results…).
- `history: Array` of choice records (timestamp, chapter, choice id, option, patch diff).
- API: `get_value(path)`, `set_value(path, value)`, `apply_patch(patch)`, `emit_changed`.
- Enforces type checks; safe defaults for absent paths; all reads via `get_value`.

### ChoiceDef
| Field | Purpose |
| --- | --- |
| `choice_id` | Stable key (e.g., `recruit_shaw`). |
| `prompt` | Display text. |
| `options[]` | Each includes `option_id`, `text`, `visible_if`, `enabled_if`, `effects` (patch, e.g., `trust.team += 1`, `alive.shaw = true`), optional `goto`, and `immediate_feedback`. |
| `is_major` | Highlights timeline-worthy branches. |

Supports “illusion choices” (different feedback, converging path) and true branches (state and chapter divergence).

## 5. Dialogue & branching system
- Use a dialogue engine plugin (Dialogic 2.x for Godot 4 or Dialogue Manager 1.x+) with a **DialogueService** adapter:
  - `start(dialogue_id)` / `stop()` to control sessions.
  - `set_var/get_var` bridged to `GameState`.
  - `on_event(tag, payload)` for gameplay hooks (combat, scene swap, clue gain).
- Dialogue data must support text, portrait/expression, options with conditions, cross-node/scene jumps, and variable writes.
- Logging: entering choice/critical dialogue nodes appends to `GameState.history`. MVP allows fast-forward of read text and intra-chapter rollback to auto-saves.

## 6. Ending system
### EndingDef
| Field | Purpose |
| --- | --- |
| `ending_id` | A/B/C/D (extensible). |
| `title` | Display title. |
| `conditions` | Expression on trust/ai_freedom/alive/critical events. |
| `priority` | Resolve conflicts when multiple match. |
| `epilogue_dialogue_id` | Narrative outro. |
| `credits_scene` (optional) | Scene reference. |

Trigger at chapter 9 or earlier (chapters 7/8) on catastrophic states.

## 7. AI war system (parallel progression)
- `WarState` within `GameState` or own Autoload.
- Vars: `war_progress.machine/samaritan (0..100)`, `heat_level`, `poi_spike_count`.
- `apply_war_delta(chapter_id, results)` table-driven updates after chapters/choices.
- Conflict scenes read `heat_level` to scale difficulty (enemy count/alert speed/timers) without altering narrative text.

## 8. Godot 4 scene & node architecture
```
Main.tscn
├── SceneRouter   (menu/chapter/battle/cutscene switching)
├── GameWorld     (container for current chapter scenes)
├── UI
│   ├── DialogueLayer
│   ├── ChoiceLayer
│   ├── HUDLayer
│   └── TimelineLayer
├── AudioManager
└── DebugOverlay  (dev-only)
```

Autoloads: `GameState.gd`, `SaveSystem.gd`, `SceneRouter.gd`, `AudioManager.gd`, optional `EventBus.gd`. Cross-scene communication uses autoloads/signals, not deep node paths.

## 9. UI & experience
- DialogueBox with RichText + portrait + name; OptionPanel with conditional buttons (keyboard/controller/touch).
- Timeline UI lists chapter key choices and POI results with variable deltas.
- Machine-overlay styling (scan box, SSN popups, target outlines) as CanvasLayer/post-process placeholder in MVP.
- InputMap: `ui_accept`, `ui_cancel`, `ui_up`, `ui_down`, `advance_dialogue`.
- Layout: Containers/anchors responsive from 16:9 to 20:9 with web/android-safe areas.
- Assets/performance: bundle fonts; avoid heavy textures/particles to keep mobile/web stable.

## 10. Save system
- Contents: `game_version`, `current_chapter_id`, `current_scene_id`, `vars` (alive/trust/ai_freedom/war_state/poi_results), `history`, optional `checksum`.
- Resilience: fill defaults for missing fields with logged warnings; `migrate(from_version, data)` to evolve saves safely.
- Auto-saves after each chapter segment and major choices.

## 11. MVP milestones
1. **Narrative skeleton**: ChapterManager + GameState + SaveSystem; simple JSON/Resource dialogues; choices UI; basic timeline; Chapter 1 complete loop.
2. **Branches & endings**: Chapters 1–3 critical choices; ending evaluation UI (start with A/B).
3. **AI war & conflicts**: Integrate `heat_level/war_progress` into conflict scenes; optional LimboAI (behavior-tree AI addon) for enemy behaviors.
