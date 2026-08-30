# Gachamon Legends — Backlog

Last updated: 2026-08-29 (SSS folders, Location HOME on join, FTUE waits)  
Items below were found while connecting Studio MCP, mapping the tree, fixing Depart / `KEYS`, and simplifying teleport. Studio DevLog (`ServerScriptService.Draft.DevLog`) is the in-place version comment. Review pass 2026-08-29 confirmed the list below against the live DataModel.

Priority: **P0** play-breaking or data-wrong · **P1** wrong UX / easy to regress · **P2** dead code / naming / cleanup.

Status **Fixed (dev place)** means changed in the open Studio session this week, not necessarily published.

---

## Done this session (dev place)

| Item | Notes |
|---|---|
| `DestinationConfig.KEYS` cleaned | Only `HOME`, `DUNGEON10`, `DUNGEON11`, `DUNGEON5x5` |
| Depart list follows `KEYS` and sorts 10 → 11 → 14 | Removed stale commented Coconana-as-DUNGEON1 block |
| Buzzing Plains is `DUNGEON5x5` | Replaced `DUNGEON14`. Hub trigger snapped to old DUNGEON14 gate CFrame; `ToRoom` `5_3` `S`; landing `Room_5_3` south `IsEntry`. Profile `Location` `DUNGEON1`/`DUNGEON14` → `DUNGEON5x5`. Hub walk-in playtested 2026-08-28. |
| Depart no longer sets `OnSite` when opening the picker | `OnSite` comes from `EnteredRoom` |
| Maze loading not on room-to-room | `SiteTeleportController` no longer always `"show"` |
| Maze loading on HOME → dungeon | `TeleportModule` passes site **name** so `StarterGui.LoadingScreen` enables |
| Teleport centralized | `TeleportModule` owns move + loading + `SetLocation`; door script only routes |
| Door `Touched` debounce | Always clears after 0.5s (early `return` used to leave the door stuck) |
| `SellAll` pays `price * count`; `Sale` remote deleted | DevLog 2026-08-27 |
| `CharacterAdded` no longer re-Initialize / restack FTUE | Handshake via `GetPlayerData`; coin HUD pulls on spawn. DevLog 2026-08-27 |
| `DeductCoins` refuses if broke; shop lookups nil-safe | `GetToolInfo` added. DevLog 2026-08-27 |
| One `LoadingScreen` for HOME ↔ site; HOME→entry unstick | `LookVector * 3 + (0, 3, 0)`. DevLog 2026-08-27 |
| `Template.RedeemCodes = {}`; redeem APIs init if missing | DevLog 2026-08-27 |
| HUD LocalScripts → `StarterPlayerScripts`; GUIs `ResetOnSpawn = false` | Depart, bag, coins, gear, store, blacksmith, codex, settings, notifications, sound, music. DevLog 2026-08-27 |
| Collect `TryCollect`: node + range + tool; leftover dungeons not stocked | DevLog 2026-08-27 |
| `MusicMuted` persisted on profile | Settings load/save; music volume on play. DevLog 2026-08-28 |
| `WOOD_1` dates refreshed | `AvailabilityEndDate = "2027-11-30"`. Spawn still does not filter dates. |

---

## P1 — teleport / loading / Depart

### `IsEntry` and `IsExit` both mean “go home”

Walking into the dungeon **entry** from inside is an exit. That may be intended, but it is not documented on the markers. HOME→entry now unsticks like room-to-room; first-room collisions are less fragile but the dual meaning remains.

### Depart Destinations button is Studio-only

```lua
destinationsButton.Visible = isStudio and not uiCoordinator.OnSite
```

Intentional test skip. Production enter is each site’s `DungeonEntryDoorway` `TeleportTrigger`.

### Stale `Location` after rejoin

Profile `Location` is written on teleport but **not reset on join**. Players always spawn at hub. If the last session left `Location = DUNGEON5x5`, walking the Buzzing Plains gate skips the loading screen (`GetLocation == dungeonId`) and Journey FTUE can complete without leaving hub this session. Reset to `HOME` on session start (or restore into the dungeon — do not leave it dangling).

### `BagGUITEST` is Enabled

`StarterGui.BagGUITEST` clones to every player. `Testing` and `ScreenOverlayNEW` are at least disabled.

### Hub doors live under Destinations, not Map

`Workspace.Destinations.DUNGEON5x5|10|11|8.DungeonEntryDoorway` are CFramed onto survey-trip gates. `DUNGEON8` hub volume at `(-236, 39, 21)` still teleports. `SiteTeleportController` does not check `KEYS`.

### Workspace-root bake clones replicate

~15 `CoconanaOasisTemplate*` models plus `BuzzingSavannahTemplate` sit at Workspace root (~27k descendants). They include tagged `DungeonDoorwayMarker`s. `ServerStorage.SiteModelTemplates` already has the bake set.

### Site numbering is inconsistent

| Key | Folder | UI description | Display name |
|---|---|---|---|
| `DUNGEON11` | 11 | “Level 1” | Coconana Oasis |
| `DUNGEON5x5` | 5x5 | “Level 3” | Buzzing Plains |
| `DUNGEON10` | 10 | “Level 10” | Blackthorn Mountain |

Product says “sites 10, 11, and 14.” Players see “Level 1 / 3 / 10.” Easy to enable the wrong folder.

### `SetLocation` silently drops unknown keys

Doors in leftover `DUNGEON8` can still teleport the character (`DungeonId` attribute) but location will not save. Next join snaps thinking you are at HOME.

---

## P1 — economy / FTUE / client

### FTUE forage handler busy-waits and prints forever

```lua
while PlayerInventoryManager.IsEmpty(player) do
    print("Player hasn't harvested anything yet")
    task.wait(3)
end
```

Spam in the server log for every new player until first collect. Same polling style on Journey (`wait` until `Location ~= HOME`).

### Collect grant path

`PlayerInventoryAdd` remote is gone. Collect goes through `MaterialReplenishModule.TryCollect` (tagged part, live `MaterialItemId`, server range, tool). `AddItem` is still a tool-only grant — do not call it from a new remote. Leftover `DUNGEON8` nodes are no longer replenished (`KEYS` only).

### Character-bound UI

Moved to `StarterPlayerScripts` (2026-08-27). Related ScreenGuis `ResetOnSpawn = false`. `ScreenOverlayNEW` still has no owner script.

### `ScreenOverlayNEW`

In-dungeon HUD (health, compass, alerts, quick slots) exists in StarterGui; no controller in this pass was found that clearly drives it.

---

## P2 — dead world and scripts

### Disabled / duplicate dungeon pipeline

All Disabled: `DungeonMaterializer`, `V2`, `v3`, `v3ORIGINAL`. Also `RoomFurnisherOLD`. `Draft` (`Draft`, `TerrainExportImport`, `DevLog`) is still in `ServerScriptService`.

### Leftover destinations

`Workspace.Destinations.DUNGEON8` (leftover, not in `KEYS`) plus doorway markers. Hub still has survey-trip art for Chimstone Ruins, Echoes of Willoria, Spicy Savannah.

### Art sandboxes in the live tree

- `Workspace.Avo's Workspace` (~172 instances)
- Coconana oasis templates at **Workspace root** (not under Map or ServerStorage)
- `ServerStorage.Old Map` (large)
- `StarterGui.Testing`, `BagGUITEST`

### Commented product systems

Level/XP on `PlayerDataManager`, `PlayerLevelManager`, `BadgeHandler`, `AnalyticsModule` (except FTUE analytics). Config still has `PLAYER_LEVEL_DEBUG` / `BADGE_AWARD_DEBUG`.

### Config typos / noise

- `PROFILE_STORE_DEBUB`
- `REDEEM_CODES_MAX_LENGHT`
- `MazeLoadingScreenController` now only drives `LoadingScreen` (MazeLoadingGui leftover is destroyed)

---

## Gaps (product vs code)

| Expectation | Actual |
|---|---|
| Runtime-generated mazes | Pre-baked rooms; generators off |
| Full tool ladder (5 tiers in config) | Only Willow tier 1 items + 3 tools in ServerStorage |
| Redeem codes | `Template` has `RedeemCodes = {}`; still no UI |
| Player level | Commented out |
| Non-Studio depart | Hub `DungeonEntryDoorway` volumes (Destinations menu is Studio-only) |
| Buzzing Plains (`DUNGEON5x5`) playable | Hub walk-in playtested; lands `Room_5_3` south |
| One loading treatment | One `LoadingScreen`; name is the trip title |
| Sell uses inventory counts | `price * count` |
| Clean enable list for sites | `KEYS` is now clean; world still contains `DUNGEON8` plus workspace-root templates |
| `Location` persists across sessions | Written on teleport; not restored and not reset on join |
| In-dungeon HUD | `ScreenOverlayNEW` exists, Enabled=false, no controller |

---

## Suggested next engineering

Passes 0–4 of the 2026-08-29 refactor are on the **dev place**: ToolModule requires restored, `Location` HOME on join, KEYS on doors, SSS folders, purchase/repair on `PlayerGearManager`, unused ToolModule APIs trimmed, FTUE waits on signals. `Queue` stays (Notifications). Bake pipeline stays.

Still open:

1. Playtest store buy, axe harvest, Buzzing Plains walk-in (loading once), Go Home, FTUE forage→sell→journey.
2. Playtest Coconana (`ToRoom` `4_2` S) and Blackthorn (`10_5` S, marker Y≈65) hub walk-ins.
3. Align site id / folder / “Level N”, or drop Level N from Depart copy. Filter spawn with `isItemAvailable`.
4. Park workspace-root bake templates / Avo’s Workspace under a `_Studio` folder if they get in the way. Do not delete bake.

Do not add Rojo. Do not re-enable dungeon generation in this place.
