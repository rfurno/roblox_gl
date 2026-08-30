# Gachamon Legends — Code review

Date: 2026-08-30  
Prior pass: 2026-08-26 / 2026-08-27 / 2026-08-28 / 2026-08-29.  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`).  
Method: re-read live gameplay scripts, Destinations tree, hub gates, ScreenGuis, and disabled folders. Confirmed no Rojo; Studio remains source of truth.

---

## Summary

The live loop is still in decent shape: ProfileStore boot, `TryCollect`, sell-all, Willow gear, and teleport through `TeleportModule`. Do **not** add Rojo, Knit, or a client app module.

The place is still carrying Studio-only weight: bake pipeline, ~27k workspace-root room templates, `Avo's Workspace`, and test GUIs. `DUNGEON8`, `BagGUITEST`, and `Old Map` are gone. After archive, remaining bugs are small and local.

2026-08-30: `DungeonMaterializerv3` Command Bar locals; Savannah 4-door bake; in-maze entry/exit; `ToolModule` cleanup; blacksmith repair always returns Damage; Coconana `4_3`↔`3_3` south door; announcements tolerate missing ConfigService key.

2026-08-29 refactor (dev place): ToolModule requires restored on shop + harvest; join `SetLocation(HOME)`; KEYS on teleports; SSS folders (`Teleport`, `Collect`, `Economy`, `World`); purchase/repair remotes on `PlayerGearManager`; unused ToolModule durability APIs trimmed; FTUE waits on inventory/location. `Queue` stays (`Notifications`).

Playtest still needed: store buy, axe harvest, Buzzing Plains loading once, FTUE. Pass 5 (Depart Level N) not done.

---

## What to do, in order

Do not interleave new features with this list. Archive first so later edits are not fighting dead instances.

### 1. Archive (simplification, not a rewrite)

Delete or move to an unpublished archive place. Keep Disabled scripts out of the published DataModel.

| Remove | Why |
|---|---|
| Workspace-root `CoconanaOasisTemplate*` (~15 models) and `BuzzingSavannahTemplate` | Bake clones sitting in the live world (tagged doors). `ServerStorage.SiteModelTemplates` is the bake set. |
| `Workspace.Avo's Workspace` (~2k) | Art sandbox. |
| `Workspace.TorchTest` (3) + `StarterPack.TorchTest` | Leftover. Keep `GeneratedLevels.START` if you still bake here. |
| `ServerScriptService.Draft` (only `DevLog` left) | Keep a copy of `DevLog` text in git if needed, then delete. |
| `StarterGui.Testing` (Enabled=false) | Test UI. |
| `StarterGui.ScreenOverlayNEW` | Enabled=false, no owner script. Delete unless the in-dungeon HUD is next. |
| Hub art for Chimstone / Willoria / Spicy Savannah; duplicate `BlackthornMountainEntrance` | Unenabled survey-trip models. |
| Unused `ToolConfig.TIERS` Durability / SpeedFactor; Sickle / Knife / Hammer types | Live wear is `Damage` 0–100. `GetToolInfo` aliases `GetToolById`. |
| Commented level/XP/badge/`AnalyticsModule` in `PlayerDataInit` + `PLAYER_LEVEL_DEBUG` / `BADGE_AWARD_DEBUG` | Dead product surface. |

`ServerStorage.SiteModelTemplates` (~36k) does **not** replicate. Keep it while baking in this place; otherwise move it with LevelGeneration.

### 2. Close leftover play bugs

| Issue | Where | Fix |
|---|---|---|
| Workspace-root bake clones still tagged | `CoconanaOasisTemplate*` / `BuzzingSavannahTemplate` | Park or delete. `KEYS` already blocks unknown `DungeonId`; leftover markers are still live instances. |
| `CharacterAdded` forces hub HUD | `SiteTeleportController` fires `EnteredRoom(nil, nil)`; `CameraTransitionControl` also sets `OnSite = false` | Fine with Location hub-on-join. Do not also teleport-on-respawn into a dungeon. |
| `ScreenOverlayNEW` unwired | `StarterGui` | Delete or ship a controller. |

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
- Redeem codes: flag off; template + APIs; **no UI**.
- `ScreenOverlayNEW`: health / compass / alerts / quick slots. `DamageSystem` already applies tagged hazard damage to Humanoid; nothing drives this HUD.
- Two durability models: live wear is `Damage` 0–100 via `DamageEquippedTool`. Collect uses `ToolModule.CanToolHarvestItem`. `ToolConfig.TIERS[].Durability` and `HarvestWithTool` / `CheckDurability` are unused.

---

## Do not do

- Rojo / Wally / Knit / a single client “app” module. Scripts are instance-coupled (tags, doorway attributes, GUI prefabs). MCP + Studio is the edit path.
- Re-enable `DungeonMaterializerv3` in a published place (Studio-gated; run from Command Bar only).
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
| P1 `RepairTool` no nil guard | Returns remaining `Damage` (0 if instance missing). Remote always a number. |
| P1 two loading GUIs | One `StarterGui.LoadingScreen`; `SurveyTripName` set; leftover `MazeLoadingGui` destroyed. |
| P1 HOME→entry on trigger | Same unstick as room-to-room (`LookVector * 3 + (0, 3, 0)`). |
| P1 client FTUE before data | `PlayerDataClient.Start` invokes snapshot; `loaded.Event:Wait()`. |
| P1 character HUDs | HUD LocalScripts in `StarterPlayerScripts`. HUD ScreenGuis `ResetOnSpawn = false`. |
| P0 redeem vs template | `Template.RedeemCodes = {}`; Get/Add init if missing. Still no UI. |
| P1 replenish leftover dungeons | Stock only `DestinationConfig.KEYS` folders. |
| Hub `DUNGEON1` doorway `DungeonId` | Folder removed. Buzzing Plains is `DUNGEON5x5`; hub `DungeonId` `DUNGEON5x5`, `ToRoom` `5_3` `S`. |
| `WOOD_1` expired 2025-11-30 | Dates set to start `2025-01-01` / end `2027-11-30`. |
| Leftover `DUNGEON8` | Destination folder gone. |
| `BagGUITEST` Enabled | GUI gone. |
| Stale `Location` on join | `PlayerDataInit` `SetLocation("HOME")`. |
| Doors ignore `KEYS` | `SiteTeleportController` returns if `DungeonId` not in `KEYS`. |
| FTUE forage print-poll | Waits on inventory / location signals. |
| v3 bake attributes | Cleared. Knobs are Command Bar locals. `BuzzingSavannahTemplate` supported. |
| Blacksmith `remainingDamage > 0` on nil | Repair remote returns number; UI nil-safe. Playtested. |
| Live `AnnouncementModule` iterate nil | Missing ConfigService `Announcements` → `{}` + warn. |
| Coconana `4_3` north no landing | `Room_3_3` south doorway added on Dev. |

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

Intentional, not bugs: Destinations button is Studio-only. In-maze `IsExit` → spawn (join facing); `IsEntry` → just outside that site’s hub gate. Both playtested 2026-08-30.

Copying scripts to Alpha does **not** copy ConfigService `Announcements` or DataModel doors (Coconana `3_3` south). Keep `LevelGeneration` Disabled on published places.

Also: opening this place file under live `PlaceId` in Studio **warns** but still uses the live ProfileStore. `WorkspaceSetup` hard-coded fog CFrame can die with the archive pass.
