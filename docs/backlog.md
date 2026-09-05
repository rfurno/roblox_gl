# Gachamon Legends — Backlog

Last updated: 2026-09-05 (FTUE why-copy on Live 2315)  
Studio DevLog also lives in [docs/devlog.md](devlog.md) (`ServerScriptService.Draft.DevLog` is still in the place).

**Art rule:** never remove `Workspace.Avo's Workspace`. Confirm before removing any model or GUI.

Priority: **P0** play-breaking or data-wrong · **P1** wrong UX / easy to regress · **P2** dead code / naming / leftover art.

Status **Fixed (dev place)** means changed in Studio. **2026-09-05 Live/Alpha** is place version **2315** (FTUE why-copy). **2026-09-03** published 2305 with the L2 sites and Plains L1 door patch.

---

## Done this session (2026-09-05)

**FTUE why-copy** on Live **2315**. Scripts only (`ReplicatedStorage.FTUE.FtueWhyCopy`, `FtueBanner`, `FtueManagerClient`). No Workspace models, no `ServerScriptService.FTUE` change. Playtested in Live Studio after `SetFtueStage(Forage)` on `GLPlayerProfileProd`. A Dev reset does not affect Live. Old public 2314 jobs keep serving until they shut down.

---

## Done earlier (dev place, 2026-09-03)

**Coconana Oasis Lvl.2** (`DUNGEON11L2`): 16 connected rooms, L2 hub door, unlock after every `DUNGEON11` room.

**Buzzing Plains Lvl.2** (`DUNGEON5x5L2`): 16 connected rooms (4×4 Savannah bake), hub on `BuzzingPlainsEntranceL2`, unlock after every `DUNGEON5x5` room. Copied to Live and published.

**Buzzing Plains L1 doors:** was 10/13 reachable. Opened `5_3` E ↔ `5_2` W and `2_3` W ↔ `2_4` E. Now 13/13. `Avo's Workspace` untouched.

`DisplayOrder` is live (Coconana L1 → L2 → Plains L1 → Plains L2 → Blackthorn). Locked L2 gates use a proximity prompt, not a HUD toast.

Earlier (2026-08-27 … 2026-09-01): KEYS cleaned, Buzzing Plains `DUNGEON5x5`, join `SetLocation(HOME)`, `TryCollect`, in-maze entry/exit, ToolModule/blacksmith/announcements, collect wrong-tool feedback. See [devlog](devlog.md) and [code-review](code-review.md).

---

## P1 — still open

### In-maze `IsEntry` vs `IsExit` (playtested 2026-08-30)

- `IsExit` → hub `SpawnLocation`, facing the same way as join.
- `IsEntry` → town side of that site’s hub `DungeonEntryDoorway` (not spawn).
- Go Home still uses spawn.

### Depart Destinations button is Studio-only

Intentional test skip. Production enter is each site’s `DungeonEntryDoorway` `TeleportTrigger`.

### Site numbering is still inconsistent

| Key | Folder | UI description | Display name |
|---|---|---|---|
| `DUNGEON11` | 11 | “Level 1” | Coconana Oasis |
| `DUNGEON5x5` | 5x5 | “Level 1” | Buzzing Plains |
| `DUNGEON10` | 10 | “Level 10” | Blackthorn Mountain |

Depart sorts by `DisplayOrder`. Coconana and Plains L1/L2 labels match (Level 1 / Level 2). Blackthorn is still “Level 10” on folder `10`.

### Hub doors live under Destinations, not Map

`Workspace.Destinations.<KEY>.DungeonEntryDoorway` (including `DUNGEON11L2` and `DUNGEON5x5L2`) are CFramed onto survey-trip gates. Art and trigger stay in two trees.

### Equip is UI-only

`ServerStorage.Tools` has Willow models; `ToolConfig.ToolName` names them. No script clones a Tool onto the character. Collect still keys off the equipped gear record.

---

## P1 — economy / client

### Collect grant path

`PlayerInventoryAdd` remote is gone. Collect goes through `MaterialReplenishModule.TryCollect`. `AddItem` is still a tool-only grant — do not call it from a new remote.

---

## P2 — leftover art / bake kit (confirm before deleting)

**Never remove `Workspace.Avo's Workspace`.** Confirm before deleting any other model or GUI. Do not delete `ServerStorage.SiteModelTemplates` or `Workspace.GeneratedLevels.START` while baking here.

| Candidate | Why it is leftover | In place after file restore |
|---|---|---|
| Workspace-root `CoconanaOasisTemplate*` + `BuzzingSavannahTemplate` | Replicating bake clones with tagged doors. Source is ServerStorage. | Yes (20 templates) |
| `Workspace.Avo's Workspace` | Art sandbox. **Do not remove.** | Yes (~1951 descendants) |
| `Workspace.TorchTest` + `StarterPack.TorchTest` | Leftover tools. | Yes (3 + pack) |
| `Workspace.Rig` | Unreferenced dummy. | Yes |
| `StarterGui.Testing` | Enabled=false test UI. | Yes |
| `StarterGui.ScreenOverlayNEW` | Enabled=false, no owner script. | Yes |
| Hub art: Echoes of Willoria | Unenabled survey-trip model. Savannah is Plains L2; Coconana L2 uses `CoconanaOasisEntranceL2`. | Yes |
| Duplicate `BlackthornMountainEntrance` at ≈(-209, 91, 149) | Second copy; live gate is ≈(-402, 73, 483). | Yes |
| `ServerStorage.MoonAnimator2Saves` | Animator leftovers; not referenced. | Yes |
| `ServerStorage.Tools` | Willow 3D models for a future in-hand clone. | Yes |
| `Workspace.GachamonView` | Transparent part, no script refs. | Yes |
| `LevelGeneration.*` (Disabled) | Command Bar bake. Keep off in published places. | Yes |

`ReplicatedStorage.Queue` stays — `Notifications` requires it.

---

## Gaps (product vs code)

| Expectation | Actual |
|---|---|
| Runtime-generated mazes | Pre-baked rooms; Command Bar bake only |
| Full tool ladder (5 named tiers) | Only Willow tier 1 items + 3 tools in ServerStorage |
| Tool-in-hand | Gear UI equip only |
| Redeem codes | Template field; flag off; no UI |
| Player level | Commented out in `PlayerDataInit`; not in template |
| Non-Studio depart | Hub `DungeonEntryDoorway` volumes |
| Buzzing Plains L1 playable | 13 rooms, all connected (doors patched 2026-09-03). L2 is a separate 16-room Savannah bake |
| One loading treatment | One `LoadingScreen`; name is the trip title |
| Clean enable list for sites | `KEYS` includes Coconana L1/L2, Plains L1/L2, Blackthorn. Depart sorts by `DisplayOrder` |
| `Location` on join | Reset to `HOME` |
| In-dungeon HUD | `ScreenOverlayNEW` was Enabled=false, no controller. Confirm before deleting. |

---

## Suggested next engineering

1. Never remove `Avo's Workspace`. Confirm before any other model/GUI delete.
2. Playtest Plains L1 side rooms `5_2` and `2_4`/`3_4` after the door patch.
3. Playtest Plains L2: locked until all 13 L1 rooms are visited; then Savannah gate, entry/exit/Go Home.
4. Drop or rewrite leftover “Level 10” Depart copy for Blackthorn.
5. Keep `DungeonMaterializerv3` Disabled in published places. ConfigService `Announcements` on Live is optional hygiene (empty key silences warns). There is no What’s New content; do not treat that as a player bug.
6. Clone `ServerStorage.Tools` onto the character on equip, or drop `ToolName` until that ships.

Do not add Rojo. Do not re-enable dungeon generation at runtime.
