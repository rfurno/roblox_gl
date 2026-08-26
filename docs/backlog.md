# Gachamon Legends — Backlog

Last updated: 2026-08-26  
Items below were found while connecting Studio MCP, mapping the tree, fixing Depart / `KEYS`, and simplifying teleport. This is **not** a full code review (that pass is next).

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

---

## P0 — correctness

### Sell-all via `Sale` remote is broken

`PlayerInventoryManager.onSaleFired` does:

```lua
local coins = 0
coins.Value = coins.Value + total
```

That errors if anything fires `Sale`. The working path is `SellAllProximityPrompt` → `AddCoins`. Dead remote is a trap.

### `SellAll` ignores stack count

Pricing is `GetItemBasePrice(itemId)` **once per id**, not `price * count`. A bag of 10 items of one type pays as one.

### Redeem codes vs template

`ServerConfiguration.REDEEM_CODES_ENABLED = true` and `PlayerDataManager` reads `profile.Data.RedeemCodes`, but `Template` has **no** `RedeemCodes` field. First redeem will nil-index unless ProfileStore reconcile magically adds it (it will not without a template key).

### `PlayerDataInit` re-inits on every `CharacterAdded`

After an **8 second** wait it calls `Initialize` again (reload inventory/gear/catalog, fire loaded remotes). Respawns can duplicate UI work, double-init FTUE side effects, and fight in-flight teleports.

---

## P1 — teleport / loading / Depart

### Two loading GUIs

`MazeLoadingScreenController`:

- `"show", name` → `StarterGui.LoadingScreen`
- `"show"` with no name → generated `MazeLoadingGui`

HOME → site uses the designed LoadingScreen. Site → HOME uses the other overlay. Easy to “fix” one path and break the other (already happened once).

### Entry pivot vs door trigger

`TeleportPlayerToDestination` pivots to the **entry marker CFrame**. Room-to-room adds `LookVector * 3 + (0, 3, 0)` so the player does not stand on the trigger. Landing on `IsEntry` would send the player **straight home**. If entry still “works,” it is because of trigger size/orientation, not because the code avoids it.

### `IsEntry` and `IsExit` both mean “go home”

Walking into the dungeon **entry** from inside is an exit. That may be intended, but it is not documented on the markers. Combined with the pivot issue above, first-room collisions are fragile.

### Depart Destinations button is Studio-only

```lua
destinationsButton.Visible = isStudio and not uiCoordinator.OnSite
```

Testers/live cannot open the site picker from that control. Hub survey-trip models look like entrances but are not confirmed as `TeleportTrigger`s. Need a shipped way to depart.

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

### Client harvest can be spoofed

`PlayerInventoryAdd` is a **client-fired** RemoteEvent. Server checks “can harvest this item id with equipped tool” but not that the player is near a live node or that the node still has that item. A cheater can add any harvestable they have a tool for.

### Character-bound UI

Bag, gear, store, blacksmith, coins, **Depart**, codex all live in `StarterCharacterScripts`. They tear down and rebuild on every death/respawn. Depart in particular can lose picker state mid-dungeon if the character is replaced.

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
- `MazeLoadingScreenController` still builds a second GUI that is only used on unnamed `"show"`

---

## Gaps (product vs code)

| Expectation | Actual |
|---|---|
| Runtime-generated mazes | Pre-baked rooms; generators off |
| Full tool ladder (5 tiers in config) | Only Willow tier 1 items + 3 tools in ServerStorage |
| Redeem codes | Flag on, no template field, no UI found in this pass |
| Player level | Commented out |
| Non-Studio depart | Destinations button hidden outside Studio |
| One loading treatment | Two GUIs, name-discriminated |
| Sell uses inventory counts | Pays once per item id |
| Clean enable list for sites | `KEYS` is now clean; world still contains 3 extra dungeons |

---

## Suggested next engineering (after code review)

1. Unify maze loading to **one** GUI; pass a name (or a dedicated `reason`) for both enter and return.
2. Offset HOME → entry pivot the same way as room-to-room, or disable the entry trigger until the player steps off.
3. Delete or archive Disabled materializers, Draft, Old Map, Avo’s Workspace, unused destination folders, test GUIs.
4. Fix `SellAll` to `price * count`; remove or repair `Sale` remote.
5. Add `RedeemCodes = {}` to `Template` or turn the flag off.
6. Move Depart (and other HUDs) to `StarterPlayerScripts` so they survive respawn.
7. Stop `CharacterAdded` + 8s `Initialize` in `PlayerDataInit`.
8. Align site id / folder / “Level N” copy, or document the mapping in UI.
9. Ship a non-Studio depart entry (hub prompt on survey-trip models, or always-on Destinations button).
10. Validate collect against a real tagged part near the player.

---

## Full code review

Not started. When requested, review should treat Studio DataModel scripts as the codebase and use this backlog as the known-issue list rather than rediscovering the items above.
