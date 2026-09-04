# Gachamon Legends — Code review

Date: 2026-09-03  
Prior pass: 2026-08-26 / 2026-08-27 / 2026-08-28 / 2026-08-29 / 2026-08-30 / 2026-09-01.  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`).  
Method: re-read live gameplay scripts, Destinations tree, hub gates, ScreenGuis, and leftover folders. Confirmed no Rojo; Studio remains source of truth.

**Art rule:** never remove `Workspace.Avo's Workspace`. Confirm before removing any model or GUI.

---

## Summary

The live loop is still in decent shape: ProfileStore boot, `TryCollect`, sell-all, Willow gear, and teleport through `TeleportModule`. Do **not** add Rojo, Knit, or a client app module.

2026-09-03: Coconana Oasis Lvl.2 and Buzzing Plains Lvl.2 are in Destinations (16 connected rooms each). L2 unlock is visit-every-L1-room; lock prompt on the hub gate (no HUD toast). Plains L1 doors patched to 13/13. Models, scripts, and UI copied to Live/Alpha (`115297023432140`) and **published** (place version 2305). `Avo's Workspace` is not on live.

Remaining Studio weight: workspace-root bake clones, `SiteModelTemplates`, `GeneratedLevels.START`, Disabled `LevelGeneration`, unused Willoria hub art. Confirm before deleting any of it. `Avo's Workspace` stays.

Playtest still needed: store buy, axe harvest, Buzzing Plains loading once, FTUE. “Level N” Depart copy is still marketing, not folder id.

---

## What to do, in order

### 1. Archive — wait for confirmation

Never remove `Avo's Workspace`. Confirm before deleting any other model or GUI.

| Candidate | Why | In place (file restore) |
|---|---|---|
| Workspace-root `CoconanaOasisTemplate*` + `BuzzingSavannahTemplate` | Bake clones with tagged doors. Source is `ServerStorage.SiteModelTemplates`. | Yes |
| `Workspace.Avo's Workspace` (~2k) | Art sandbox. **Do not remove.** | Yes |
| `Workspace.TorchTest` + `StarterPack.TorchTest` | Leftover. | Yes |
| `ServerScriptService.Draft` (`DevLog`) | Historical notes. Copy in [docs/devlog.md](devlog.md). | Yes |
| `StarterGui.Testing` | Test UI. | Yes |
| `StarterGui.ScreenOverlayNEW` | Enabled=false, no owner script. | Yes |
| Duplicate `BlackthornMountainEntrance` at ≈(-209, 91, 149) | Live gate is ≈(-402, 73, 483). | Yes |
| `Workspace.Rig` | Unreferenced dummy. | Yes |

`ServerStorage.SiteModelTemplates` does **not** replicate. Keep it while baking in this place; otherwise move it with LevelGeneration.

### 2. Close leftover play bugs

| Issue | Where | Status |
|---|---|---|
| Workspace-root bake clones still tagged | `CoconanaOasisTemplate*` / `BuzzingSavannahTemplate` | Still in Workspace; `KEYS` already blocks unknown `DungeonId`. Confirm before parking. |
| `CharacterAdded` forces hub HUD | `SiteTeleportController` fires `EnteredRoom(nil, nil)` | Fine with Location hub-on-join. Do not also teleport-on-respawn into a dungeon. |
| `ScreenOverlayNEW` unwired | `StarterGui` | Present, Enabled=false, no owner. Confirm before deleting. |

### 3. Naming and config

| Key | Folder | UI label | Display name | Hub `ToRoom` | Rooms |
|---|---|---|---|---|---|
| `DUNGEON11` | 11 | Level 1 | Coconana Oasis | `4_2` S | 8 |
| `DUNGEON11L2` | 11L2 | Level 2 | Coconana Oasis Lvl.2 | `4_2` S | 16, connected |
| `DUNGEON5x5` | 5x5 | Level 1 | Buzzing Plains | `5_3` S | 13, connected (patched) |
| `DUNGEON5x5L2` | 5x5L2 | Level 2 | Buzzing Plains Lvl.2 | `4_2` S | 16, connected |
| `DUNGEON10` | 10 | Level 10 | Blackthorn Mountain | `10_5` S | 71 |

Depart sorts by `DisplayOrder`. Plains L1 is **Level 1** in config and on the gate billboard (`Buzzing Plains (Lvl.1)`). Blackthorn is still “Level 10”.

### 4. Product gaps (do not start these until playtest)

- Runtime maze gen: keep off. Pre-baked rooms are the game.
- Tool ladder: five named tiers remain; live tools are three Willow items. Either ship tier 2 or leave the names as future labels.
- Tool-in-hand: `ServerStorage.Tools` is unused by scripts. Equip is gear UI only.
- Redeem codes: flag off; template + APIs; **no UI**.
- In-dungeon HUD: health / compass / alerts / quick slots. `DamageSystem` already applies tagged hazard damage to Humanoid. `ScreenOverlayNEW` is present, Enabled=false, no owner. Confirm before deleting.

---

## Do not do

- Remove `Workspace.Avo's Workspace`. Confirm before deleting any other model or GUI.
- Rojo / Wally / Knit / a single client “app” module. Scripts are instance-coupled (tags, doorway attributes, GUI prefabs). MCP + Studio is the edit path.
- Re-enable `DungeonMaterializerv3` in a published place (Studio-gated; run from Command Bar only).
- Map `Workspace` into git.
- Expose `PlayerInventoryManager.AddItem` on a remote. Collect stays on `TryCollect`.

---

## Closed since 2026-08-26

| Was | Now |
|---|---|
| P0 `PlayerInventoryAdd` client grant | Remote gone. Collect is `MaterialReplenishModule.TryCollect`. |
| P0 `Sale` remote wipes bag | Remote and handler deleted. Sell is Benji prompt → `SellAll` → `AddCoins`. |
| P0 `SellAll` once per id | `price * count`. |
| P0 respawn 8s `Initialize` | `CharacterAdded` only walk speed / `IsAlive`. FTUE starts once after profile load. |
| P0 stacked FTUE loops | One in-flight handler per player; `task.cancel` on leave / new stage. |
| P0 shop `GetPlayerData` missing | Server API + `ReplicatedStorage.Events.GetPlayerData`. |
| P1 `DeductCoins` never refuses | Returns `false` if invalid/insufficient; `true` on success. |
| P1 `GetToolPrice` / repair cost nil-index | Return `nil`. `GetToolInfo` added. |
| P1 `HasTool` wrong key | Looks up catalog `toolId`. |
| P1 `RepairTool` no nil guard | Returns remaining `Damage` (0 if instance missing). Remote always a number. |
| P1 two loading GUIs | One `StarterGui.LoadingScreen`. |
| P1 HOME→entry on trigger | Same unstick as room-to-room (`LookVector * 3 + (0, 3, 0)`). |
| P1 client FTUE before data | `PlayerDataClient.Start` invokes snapshot; `loaded.Event:Wait()`. |
| P1 character HUDs | HUD LocalScripts in `StarterPlayerScripts`. HUD ScreenGuis `ResetOnSpawn = false`. |
| P0 redeem vs template | `Template.RedeemCodes = {}`; Get/Add init if missing. Still no UI. |
| P1 replenish leftover dungeons | Stock only `DestinationConfig.KEYS` folders. |
| Hub `DUNGEON1` doorway `DungeonId` | Folder removed. Buzzing Plains is `DUNGEON5x5`. |
| `WOOD_1` expired 2025-11-30 | Dates set to start `2025-01-01` / end `2027-11-30`. |
| Leftover `DUNGEON8` / `BagGUITEST` / `Old Map` | Gone. |
| Stale `Location` on join | `PlayerDataInit` `SetLocation("HOME")`. |
| Doors ignore `KEYS` | `SiteTeleportController` returns if `DungeonId` not in `KEYS`. |
| FTUE forage print-poll | Waits on inventory / location signals. |
| v3 bake attributes | Cleared. Knobs are Command Bar locals. |
| Blacksmith `remainingDamage > 0` on nil | Repair remote returns number; UI nil-safe. |
| Live `AnnouncementModule` iterate nil | Missing ConfigService `Announcements` → `{}` + warn. |
| Coconana `4_3` north no landing | `Room_3_3` south doorway added on Dev. |
| Wrong-tool collect was a HUD toast | Prompt `ObjectText` + local deny. `TOOL_REQUIRED` remains in `ClientConfiguration` unused. |
| Unused tool types / Durability fields | Still in `ToolConfig` after file restore. Script-only trim proposed. |
| Depart digit-sort (`5`, `10`, `11`) | `DisplayOrder` on Dev (Coconana L1 → L2 → Plains L1 → Plains L2 → Blackthorn). |
| Plains L1 10/13 rooms reachable | Opened `5_3` E ↔ `5_2` W and `2_3` W ↔ `2_4` E. 13/13. |

---

## What is in decent shape

- ProfileStore session lock + reconcile + GDPR `AddUserId` + `RedeemCodes` on template
- Session: `CharacterAdded` is character-only; FTUE once per join; client waits on `loaded`
- `DestinationConfig.KEYS` enable list (`HOME`, `DUNGEON10`, `DUNGEON11`, `DUNGEON11L2`, `DUNGEON5x5`, `DUNGEON5x5L2`)
- `TeleportModule` owns PivotTo + one loading GUI + `SetLocation`; unstick on HOME→entry
- Hub enter: `Destinations.<KEY>.DungeonEntryDoorway` CFramed onto survey-trip gates (Buzzing Plains playtested)
- `TryCollect` + KEYS-only replenish; collect prompt shows required tool; local deny if equipped tool cannot harvest
- `DeductCoins` / `SellAll` stack pricing / shop `GetPlayerData` + `GetToolInfo`
- HUD LocalScripts on `StarterPlayerScripts`; HUD ScreenGuis `ResetOnSpawn = false`
- Door debounce via `task.delay`

Intentional, not bugs: Destinations button is Studio-only. In-maze `IsExit` → spawn (join facing); `IsEntry` → just outside that site’s hub gate. Both playtested 2026-08-30.

Copying scripts to Alpha does **not** copy ConfigService `Announcements` or DataModel doors (Coconana `3_3` south). Keep `LevelGeneration` Disabled on published places.

Also: opening this place file under live `PlaceId` in Studio **warns** but still uses the live ProfileStore. `WorkspaceSetup` still hard-codes the Fog model CFrame.
