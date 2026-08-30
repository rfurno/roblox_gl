# Gachamon Legends — Code review

Date: 2026-08-29  
Prior pass: 2026-08-26 / 2026-08-27 / 2026-08-28.  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`).  
Method: re-read live gameplay scripts, Destinations tree, hub gates, ScreenGuis, and disabled folders. Confirmed no Rojo; Studio remains source of truth.

---

## Summary

The live loop is still in decent shape: ProfileStore boot, `TryCollect`, sell-all, Willow gear, and teleport through `TeleportModule`. Do **not** add Rojo, Knit, or a client app module.

The place is still carrying a second game: bake pipeline, leftover `DUNGEON8`, ~27k workspace-root room templates, `Avo's Workspace`, Old Map, and test GUIs. That is the main simplification. After archive, the remaining bugs are small and local.

2026-08-29 refactor (dev place): ToolModule requires restored on shop + harvest; join `SetLocation(HOME)`; KEYS on teleports; SSS folders (`Teleport`, `Collect`, `Economy`, `World`); purchase/repair remotes on `PlayerGearManager`; unused ToolModule durability APIs trimmed; FTUE waits on inventory/location. `Queue` stays.

Playtest still needed: store buy, axe harvest, Buzzing Plains loading once, FTUE. Pass 5 (Depart Level N / spawn dates) not done.

---

## What to do, in order

Do not interleave new features with this list. Archive first so later edits are not fighting dead instances.

### 1. Archive (simplification, not a rewrite)

Delete or move to an unpublished archive place. Keep Disabled scripts out of the published DataModel.

| Remove | Why |
|---|---|
| `Workspace.Destinations.DUNGEON8` (~3.1k descendants + hub door at `-236, 39, 21`) | Doors still fire; `SetLocation` drops `DUNGEON8`. |
| Workspace-root `CoconanaOasisTemplate*` (~15 models, ~27k descendants) and `BuzzingSavannahTemplate` | Bake clones sitting in the live world. `ServerStorage.SiteModelTemplates` already holds the bake set. |
| `Workspace.Avo's Workspace` (~2k) | Art sandbox. |
| `Workspace.GeneratedLevels`, `Workspace.TorchTest` (3) | Leftover. |
| `ServerStorage.Old Map` (~4.8k) | Unused. |
| `ServerScriptService.LevelGeneration` (all Disabled; v3 still tagged `DungeonId=DUNGEON14`) | Runtime gen is out of product. Bake belongs in a Studio-only place. |
| `ServerScriptService.Draft` (`Draft` requires missing `CollectibleSpawner` / `LevelGenerator`) | Keep a copy of `DevLog` text in git if needed, then delete. |
| `ServerScriptService.Utilities.PlayerBarrier` | Disabled; `ReplicatedFirst.PlayerBarrier` does not exist. |
| `StarterGui.Testing` (Enabled=false), **`StarterGui.BagGUITEST` (Enabled=true)** | Test UI. BagGUITEST clones to every player today. |
| `StarterGui.ScreenOverlayNEW` | Enabled=false, no owner script. Delete unless the in-dungeon HUD is next. |
| Hub art for Chimstone / Willoria / Spicy Savannah; duplicate `BlackthornMountainEntrance` | Unenabled survey-trip models. |
| Unused: `ReplicatedStorage.Queue`; leftover `ToolModule` APIs (`HarvestWithTool`, `CheckDurability`, `GetHarvestableItems`, `GetHarvestTime`, `GetToolsByTier` / `ByType`, `GetToolIdByName`, `GetUsableToolsForItemTier`) | `ToolModule` itself is live (collect, shop, repair, gear UI). `MaterialReplenishModule` requires it but does not call it — harvest goes `TryCollect` → `PlayerInventoryManager.CanPlayerHarvestItem` → `CanToolHarvestItem`. Drop the unused require and the unused durability APIs only. |
| Commented level/XP/badge/`AnalyticsModule` in `PlayerDataInit` + `PLAYER_LEVEL_DEBUG` / `BADGE_AWARD_DEBUG` / redeem flag until there is UI | Dead product surface. |

`ServerStorage.SiteModelTemplates` (~36k) does **not** replicate. Keep it only if you still bake in this place; otherwise move it with LevelGeneration.

### 2. Close leftover play bugs

| Issue | Where | Fix |
|---|---|---|
| Leftover doors still move you | `SiteTeleportController` | Before `TeleportPlayerToRoom`, require `DestinationConfig.KEYS[dungeonId]`. Then delete `DUNGEON8`. |
| Stale `Location` after rejoin / death | `PlayerDataInit` | Always spawn at hub. On session start, `SetLocation(HOME)` unless you add restore-to-dungeon. Today a leftover `DUNGEON5x5` location makes hub→Buzzing Plains skip the loading screen (`GetLocation == dungeonId`) and finishes Journey FTUE without leaving hub this session. |
| `CharacterAdded` forces hub HUD | `SiteTeleportController` fires `EnteredRoom(nil, nil)`; `CameraTransitionControl` also sets `OnSite = false` | Fine once Location is hub-on-join. Do not also teleport-on-respawn into a dungeon. |
| `BagGUITEST` Enabled | `StarterGui` | Delete. |
| `GetStoreName` can return nil | `PlayerDataInit` | Kick / fallback to development store. Never `ProfileStore.New(nil, …)`. |
| `CanToolHarvestItem` indexes `TOOLS[toolId]` after a truthy unknown string | `ToolModule` | `if not TOOLS[toolId] then return false`. |
| `AddCoins` has no floor | `PlayerDataManager` | Reject non-positive / NaN the same way `DeductCoins` does. |
| Spawn ignores availability dates | `GetItemListByType` | Filter with `isItemAvailable`. Codex already does. |
| FTUE forage/sell `print` + `task.wait(3)` | stage `HandleAsync` | Bind inventory/location events; drop the print. Journey poll is quieter but the same shape. |
| `RestorePlayerData` writes `known`/`recent` | `ItemCatalogModule` | Dead; live path is `UpdatePlayerData` with `Known`/`Recent`. Delete or align. |
| `getToolIdFromPlayerToolId` nil-indexes | `PlayerGearManager` | Unused today; delete or guard. |

### 3. Naming and config (stop enabling the wrong folder)

| Key | Folder | UI “Level N” | Display name | Hub `ToRoom` |
|---|---|---|---|---|
| `DUNGEON11` | 11 | Level 1 | Coconana Oasis | `4_2` S — `Room_4_2` exists; playtest walk-in |
| `DUNGEON5x5` | 5x5 | Level 3 | Buzzing Plains | `5_3` S — playtested |
| `DUNGEON10` | 10 | Level 10 | Blackthorn Mountain | `10_5` S — hub marker at Y≈65 |

Either make Description match the folder, or stop showing “Level N” and show the name only. `GetDestinationList` sorts by the first digits in the key (`5`, `10`, `11`), so Depart order is Buzzing Plains → Blackthorn → Coconana.

### 4. Product gaps (do not start these until 1–3)

- Runtime maze gen: keep off. Pre-baked rooms are the game.
- Tool ladder: config has 5 tiers + unused types (Sickle, Knife, Hammer); live tools are three Willow items. Either ship tier 2 or strip the unused config.
- Redeem codes: `REDEEM_CODES_ENABLED = true` and APIs exist; **no UI**. Turn the flag off or ship the UI.
- `ScreenOverlayNEW`: health / compass / alerts / quick slots. `DamageSystem` already applies tagged hazard damage to Humanoid; nothing drives this HUD.
- Two durability models: live wear is `Damage` 0–100 via `DamageEquippedTool`. Collect uses `ToolModule.CanToolHarvestItem`. `ToolConfig.TIERS[].Durability` and `HarvestWithTool` / `CheckDurability` are unused.

---

## Do not do

- Rojo / Wally / Knit / a single client “app” module. Scripts are instance-coupled (tags, doorway attributes, GUI prefabs). MCP + Studio is the edit path.
- Re-enable `DungeonMaterializerv3` in a published place (it is Studio-gated, but attributes still say `DUNGEON14`).
- Map `Workspace` into git.
- Expose `PlayerInventoryManager.AddItem` on a remote. Collect stays on `TryCollect`.

---

## Closed since 2026-08-26

| Was | Now |
|---|---|
| P0 `PlayerInventoryAdd` client grant | Remote gone. Collect is `MaterialReplenishModule.TryCollect` (tag, `MaterialItemId`, range, tool). FTUE forage uses the same prompt path. |
| P0 `Sale` remote wipes bag | Remote and handler deleted. Sell is Benji prompt → `SellAll` → `AddCoins`. |
| P0 `SellAll` once per id | `price * count`. |
| P0 respawn 8s `Initialize` | `CharacterAdded` only walk speed / `IsAlive`. FTUE starts once after profile load. |
| P0 stacked FTUE loops | One in-flight handler per player; `task.cancel` on leave / new stage. |
| P0 shop `GetPlayerData` missing | Server API + `ReplicatedStorage.Events.GetPlayerData`. Client handshake + coin HUD pull. |
| P1 `DeductCoins` never refuses | Returns `false` if invalid/insufficient; `true` on success. |
| P1 `GetToolPrice` / repair cost nil-index | Return `nil`. `GetToolInfo` added. |
| P1 `HasTool` wrong key | Looks up catalog `toolId`. |
| P1 `RepairTool` no nil guard | Returns `nil` if instance missing. |
| P1 two loading GUIs | One `StarterGui.LoadingScreen`; `SurveyTripName` set; leftover `MazeLoadingGui` destroyed. |
| P1 HOME→entry on trigger | Same unstick as room-to-room (`LookVector * 3 + (0, 3, 0)`). |
| P1 client FTUE before data | `PlayerDataClient.Start` invokes snapshot; `loaded.Event:Wait()`. |
| P1 character HUDs | HUD LocalScripts in `StarterPlayerScripts`. HUD ScreenGuis `ResetOnSpawn = false`. |
| P0 redeem vs template | `Template.RedeemCodes = {}`; Get/Add init if missing. Still no UI. |
| P1 replenish leftover dungeons | Stock only `DestinationConfig.KEYS` folders. |
| Hub `DUNGEON1` doorway `DungeonId` | Folder removed. Buzzing Plains is `DUNGEON5x5`; hub `DungeonId` `DUNGEON5x5`, `ToRoom` `5_3` `S`. |
| `WOOD_1` expired 2025-11-30 | Dates set to start `2025-01-01` / end `2027-11-30`. |

---

## What is in decent shape

- ProfileStore session lock + reconcile + GDPR `AddUserId` + `RedeemCodes` on template
- Session: `CharacterAdded` is character-only; FTUE once per join; client waits on `loaded`
- `DestinationConfig.KEYS` enable list (`HOME`, `DUNGEON10`, `DUNGEON11`, `DUNGEON5x5`)
- `TeleportModule` owns PivotTo + one loading GUI + `SetLocation`; unstick on HOME→entry
- Hub enter: `Destinations.<KEY>.DungeonEntryDoorway` CFramed onto survey-trip gates (Buzzing Plains playtested)
- `TryCollect` + KEYS-only replenish
- `DeductCoins` / `SellAll` stack pricing / shop `GetPlayerData` + `GetToolInfo`
- HUD LocalScripts on `StarterPlayerScripts`; HUD ScreenGuis `ResetOnSpawn = false`
- Door debounce via `task.delay`

Intentional, not bugs: Destinations button is Studio-only; in-maze `IsEntry` / `IsExit` both send you HOME.

Also: opening this place file under live `PlaceId` in Studio **warns** but still uses the live ProfileStore. Typos `PROFILE_STORE_DEBUB` / `REDEEM_CODES_MAX_LENGHT` and `WorkspaceSetup` hard-coded fog CFrame can die with the archive pass.
