# Gachamon Legends — Backlog

Last updated: 2026-09-01 (collect prompt tool requirement + local deny)  
Items below were found while connecting Studio MCP, mapping the tree, fixing Depart / `KEYS`, and simplifying teleport. Studio DevLog (`ServerScriptService.Draft.DevLog`) is the in-place version comment.

Priority: **P0** play-breaking or data-wrong · **P1** wrong UX / easy to regress · **P2** dead code / naming / cleanup.

Status **Fixed (dev place)** means changed in the open Studio session, not necessarily published.

---

## Done this session (dev place)

| Item | Notes |
|---|---|
| `DungeonMaterializerv3` two template kinds | Command Bar locals only (no script attributes). `CoconanaOasisTemplate` = 15-layout set. `BuzzingSavannahTemplate` = single all-four-doors model; unused doorways hidden. Bake confirmed working. Keep Disabled. |
| v3 instance attributes removed | Stale `DungeonId=DUNGEON14` / 4×4 knobs were never read from Command Bar. |
| `DestinationConfig.KEYS` cleaned | Only `HOME`, `DUNGEON10`, `DUNGEON11`, `DUNGEON5x5` |
| Buzzing Plains is `DUNGEON5x5` | Hub `ToRoom` `5_3` `S`; landing `Room_5_3` south `IsEntry`. Walk-in playtested 2026-08-28. |
| Join `SetLocation(HOME)` | `PlayerDataInit` always writes HOME after profile load. |
| Doors require `KEYS` | `SiteTeleportController` + `TeleportHandler` ignore unknown `DungeonId`. |
| `DUNGEON8` removed | No leftover destination folder. |
| `BagGUITEST` removed | `StarterGui.Testing` still exists, Enabled=false. |
| `ServerStorage.Old Map` removed | |
| FTUE waits on signals | Forage / sell / journey no longer print-poll. |
| Teleport centralized | `TeleportModule` owns move + loading + `SetLocation` |
| HUD LocalScripts → `StarterPlayerScripts` | Related ScreenGuis `ResetOnSpawn = false` |
| Collect `TryCollect` | Node + range + tool; KEYS-only replenish |
| `MusicMuted` persisted | |
| In-maze `IsEntry` / `IsExit` | Exit → spawn, join facing. Entry → town side of hub gate. Playtested. |
| `ToolModule` cleanup | `GetToolInfo` aliases `GetToolById`; harvest checks silent; `GetTools` does not copy unused `Durability`. |
| Blacksmith repair nil | `RepairToolRemote` always returns remaining `Damage`. UI nil-safe. Playtested. |
| Coconana `4_3` north | Unpaired bake door. Added `Room_3_3` south. Playtested on Dev. Copy the doorway (not a script) to published places. |
| Announcements on live | ConfigService `Announcements` missing → empty table, no crash. `GetAnnouncements` inits `{}`. Add the config key on Alpha to show What’s New. |
| Collect wrong-tool feedback | Prompt `ObjectText` = `Requires <tool>`. Local deny on prompt + part (highlight, tool-icon billboard, hold cancel). No `TOOL_REQUIRED` HUD. `HARVEST_DENIED` only. Playtested on Dev (axe node with shovel equipped; bare-hands collect still works). Copy `CustomProximityPrompts` + `CollectHarvestClient` + `MaterialReplenishModule` to published places. |

---

## P1 — still open

### In-maze `IsEntry` vs `IsExit` (playtested 2026-08-30)

- `IsExit` → hub `SpawnLocation`, facing the same way as join.
- `IsEntry` → town side of that site’s hub `DungeonEntryDoorway` (not spawn).
- Go Home still uses spawn.

### Depart Destinations button is Studio-only

Intentional test skip. Production enter is each site’s `DungeonEntryDoorway` `TeleportTrigger`.

### Site numbering is inconsistent

| Key | Folder | UI description | Display name |
|---|---|---|---|
| `DUNGEON11` | 11 | “Level 1” | Coconana Oasis |
| `DUNGEON5x5` | 5x5 | “Level 3” | Buzzing Plains |
| `DUNGEON10` | 10 | “Level 10” | Blackthorn Mountain |

`GetDestinationList` sorts by the first digits in the key (`5`, `10`, `11`), so Depart order is Buzzing Plains → Blackthorn → Coconana. Either match Description to the folder, or drop “Level N” and show the name only.

### Workspace-root bake clones replicate

~15 `CoconanaOasisTemplate*` models plus `BuzzingSavannahTemplate` sit at Workspace root. They include tagged `DungeonDoorwayMarker`s. `ServerStorage.SiteModelTemplates` is the bake set — keep that. Park or delete the Workspace copies so live doors cannot pick them up.

### Hub doors live under Destinations, not Map

`Workspace.Destinations.DUNGEON5x5|10|11.DungeonEntryDoorway` are CFramed onto survey-trip gates. Art and trigger stay in two trees.

---

## P1 — economy / client

### `ScreenOverlayNEW`

In-dungeon HUD (health, compass, alerts, quick slots) exists in StarterGui; Enabled=false; no owner script.

### Collect grant path

`PlayerInventoryAdd` remote is gone. Collect goes through `MaterialReplenishModule.TryCollect`. `AddItem` is still a tool-only grant — do not call it from a new remote.

---

## P2 — archive / cleanup (live DataModel, 2026-08-30)

Do these before more bake/feature work if they get in the way. Do **not** delete `ServerStorage.SiteModelTemplates` or `Workspace.GeneratedLevels.START` while baking in this place.

| Remove or park | Why |
|---|---|
| Workspace-root `CoconanaOasisTemplate*` + `BuzzingSavannahTemplate` | Replicating bake clones with tagged doors. Source is ServerStorage. |
| `Workspace.Avo's Workspace` (~2k descendants) | Art sandbox. |
| `Workspace.TorchTest` (3) + `StarterPack.TorchTest` | Leftover tools. |
| `StarterGui.Testing` | Enabled=false test UI. |
| Hub art: Chimstone Ruins, Echoes of Willoria, Spicy Savannah; duplicate `BlackthornMountainEntrance` | Unenabled survey-trip models. |
| `ServerScriptService.Draft` | Only `DevLog` left; copy text to git if you want it, then delete. |
| Commented level/XP/badge/`AnalyticsModule` in `PlayerDataInit` | Dead product surface until there is UI. |

Keep while baking here: `LevelGeneration.DungeonMaterializerv3` (+ ORIGINAL, `DungeonGenerator`, `RoomFurnisher`), `GeneratedLevels.START`, `SiteModelTemplates`. Move that whole bake kit to a Studio-only place if this place is what you publish.

`ReplicatedStorage.Queue` stays — `Notifications` requires it.

---

## Gaps (product vs code)

| Expectation | Actual |
|---|---|
| Runtime-generated mazes | Pre-baked rooms; Command Bar bake only |
| Full tool ladder (5 tiers in config) | Only Willow tier 1 items + 3 tools in ServerStorage |
| Redeem codes | Template field; flag off; no UI |
| Player level | Commented out |
| Non-Studio depart | Hub `DungeonEntryDoorway` volumes |
| Buzzing Plains (`DUNGEON5x5`) playable | Hub walk-in playtested; Savannah **template** bake works; live site art is still the existing 5x5 rooms until you replace Destinations |
| One loading treatment | One `LoadingScreen`; name is the trip title |
| Clean enable list for sites | `KEYS` + Destinations folders match (10 / 11 / 5x5) |
| `Location` on join | Reset to `HOME` |
| In-dungeon HUD | `ScreenOverlayNEW` exists, Enabled=false, no controller |

---

## Suggested next engineering

1. Park or delete Workspace-root bake clones and `Avo's Workspace` so tagged leftover doors cannot fire.
2. Playtest store buy, axe harvest, Buzzing Plains walk-in (loading once), Go Home, FTUE forage→sell→journey.
3. Playtest Coconana hub walk-in (`ToRoom` `4_2` S) and Blackthorn (`10_5` S, marker Y≈65). Coconana `4_3` north is fixed on Dev.
4. Align site id / folder / “Level N”, or drop Level N from Depart copy.
5. If the Savannah bake should **replace** live `DUNGEON5x5`, move `GeneratedLevels` output into Destinations and retarget the hub `ToRoom` / `ToDoorDirection`.
6. Keep `DungeonMaterializerv3` Disabled in published places. Add ConfigService `Announcements` on Alpha if What’s New should show.

Do not add Rojo. Do not re-enable dungeon generation at runtime.
