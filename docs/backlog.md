# Gachamon Legends — Backlog

Last updated: 2026-08-27 (DUNGEON1 entrance disabled)  
Items below were found while connecting Studio MCP, mapping the tree, fixing Depart / `KEYS`, and simplifying teleport. Studio DevLog (`ServerScriptService.Draft.DevLog`) is the in-place version comment.

Priority: **P0** play-breaking or data-wrong · **P1** wrong UX / easy to regress · **P2** dead code / naming / cleanup.

Status **Fixed (dev place)** means changed in the open Studio session this week, not necessarily published.

---

## Done this session (dev place)

| Item | Notes |
|---|---|
| `DestinationConfig.KEYS` cleaned | Only `HOME`, `DUNGEON1`, `DUNGEON10`, `DUNGEON11` |
| Depart list follows `KEYS` and sorts 1 → 10 → 11 | Removed stale commented Coconana-as-DUNGEON1 block |
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
| `MusicMuted` persisted on profile | Settings load/save; music volume on play. DevLog 2026-08-27 |

---

## P1 — teleport / loading / Depart

### `IsEntry` and `IsExit` both mean “go home”

Walking into the dungeon **entry** from inside is an exit. That may be intended, but it is not documented on the markers. HOME→entry now unsticks like room-to-room; first-room collisions are less fragile but the dual meaning remains.

### DUNGEON1 (Buzzing Plains) needs a site pass

Hub walk-in is `Workspace.Destinations.DUNGEON1.DungeonEntryDoorway` (not the Destinations debug menu). `DungeonId` was `DUNGEON11` (Coconana); corrected to `DUNGEON1` on 2026-08-27. Walk-in is **off**: a part named `TeleportTrigger` may still exist but has **no** `TeleportTrigger` CollectionService tag, so `SiteTeleportController` does not bind it. Still in `DestinationConfig.KEYS`, so Studio Destinations can still send a player there.

Restore the **tag** (and part if removed) only after rooms, `DungeonId`s, and first-room `IsEntry` (`Room_4_2` south) are verified for Buzzing Plains.

### Depart Destinations button is Studio-only

```lua
destinationsButton.Visible = isStudio and not uiCoordinator.OnSite
```

Intentional test skip. Production enter is each site’s `DungeonEntryDoorway` `TeleportTrigger`.

### Site numbering is inconsistent

| Key | Folder | UI description | Display name |
|---|---|---|---|
| `DUNGEON11` | 11 | “Level 1” | Coconana Oasis |
| `DUNGEON1` | 1 | “Level 3” | Buzzing Plains |
| `DUNGEON10` | 10 | “Level 10” | Blackthorn Mountain |

Product says “sites 1, 10, and 11.” Players see “Level 1 / 3 / 10.” Easy to enable the wrong folder.

### `SetLocation` silently drops unknown keys

Doors in leftover `DUNGEON3/7/8` can still teleport the character (`DungeonId` attribute) but location will not save. Next join snaps thinking you are at HOME.

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

`PlayerInventoryAdd` remote is gone. Collect goes through `MaterialReplenishModule.TryCollect` (tagged part, live `MaterialItemId`, server range, tool). `AddItem` is still a tool-only grant — do not call it from a new remote. Leftover `DUNGEON3/7/8` nodes are no longer replenished (`KEYS` only).

### Character-bound UI

Moved to `StarterPlayerScripts` (2026-08-27). Related ScreenGuis `ResetOnSpawn = false`. `ScreenOverlayNEW` still has no owner script.

### `ScreenOverlayNEW`

In-dungeon HUD (health, compass, alerts, quick slots) exists in StarterGui; no controller in this pass was found that clearly drives it.

---

## P2 — dead world and scripts

### Disabled / duplicate dungeon pipeline

All Disabled: `DungeonMaterializer`, `V2`, `v3`, `v3ORIGINAL`. Also `RoomFurnisherOLD`. `Draft` (`Draft`, `TerrainExportImport`, `DevLog`) is still in `ServerScriptService`.

### Leftover destinations

`Workspace.Destinations.DUNGEON3` (15 rooms), `DUNGEON7` (76), `DUNGEON8` (31) plus hundreds of doorway markers. Not in `KEYS`. Hub still has survey-trip art for Chimstone Ruins, Echoes of Willoria, Spicy Savannah.

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
| Buzzing Plains (`DUNGEON1`) playable | Hub `TeleportTrigger` **untagged** until the site is fixed; still in `KEYS` |
| One loading treatment | One `LoadingScreen`; name is the trip title |
| Sell uses inventory counts | `price * count` |
| Clean enable list for sites | `KEYS` is now clean; world still contains 3 extra dungeons |

---

## Suggested next engineering (after code review)

1. Fix **DUNGEON1 / Buzzing Plains** and restore a **tagged** `TeleportTrigger` on `DungeonEntryDoorway`, or drop `DUNGEON1` from `KEYS`.
2. Refresh or drop `WOOD_1` dates; filter spawn the same way as the catalog.
3. Align site id / folder / “Level N” copy, or document the mapping in UI.
4. Delete or archive Disabled materializers, Draft, Old Map, unused destination folders (their **doors** still teleport), test GUIs.

---

## Full code review

Follow-up 2026-08-27. See [code-review.md](code-review.md) for closed P0s and remaining P1. This backlog is the shorter working list.
