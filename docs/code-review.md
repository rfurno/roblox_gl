# Gachamon Legends — Code review

Date: 2026-08-28 (follow-up)  
Prior pass: 2026-08-26 / 2026-08-27.  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`).  
Method: re-read live gameplay scripts after the 2026-08-27/28 Studio pass. Disabled generators and workspace Animate scripts were not line-audited.

Place version notes and `ServerScriptService.Draft.DevLog` (2026-08-28) list the same fixes.

---

## Summary

The 2026-08-26 P0 list is **closed** on the dev place: sell, respawn re-init, harvest remote, shop snapshot, DeductCoins, redeem template, loading GUI, HUD parent, and collect-vs-node. Teleport, session boot, and economy transactions are in decent shape for testers on **Coconana**, **Buzzing Plains**, and **Blackthorn**.

Highest leftover risks are **leftover dungeon doors** (`DUNGEON8`) that still teleport but do not save location, and **world/script archive**. Buzzing Plains hub walk-in is playtested.

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

## Remaining P0

None for Coconana / Buzzing Plains / Blackthorn testers.

`GetStoreName()` still returns **nil** on an unknown `PlaceId` (`PlayerDataInit`), which would make `ProfileStore.New(nil, …)` fail. Only hits if this place id is not development / testers / live.

---

## Remaining P1

### Sites / teleport

| Issue | Where | Notes |
|---|---|---|
| `IsEntry` and `IsExit` both home | `SiteTeleportController` | In-maze entry/exit markers send you HOME. Hub enter is a **non-**`IsEntry` doorway with `ToRoom` / `ToDoorDirection`. Document on markers. |
| Destinations button Studio-only | `DepartGuiController` | Intentional test skip. Production enter is hub `DungeonEntryDoorway`. |
| Site id vs “Level N” | `DestinationConfig` | DUNGEON11 = “Level 1”, DUNGEON5x5 = “Level 3”, DUNGEON10 = “Level 10”. |
| Leftover doors still move you | `DUNGEON8` tagged triggers | `SetLocation` ignores unknown keys. Character teleports; next join thinks HOME. Replenish no longer stocks that folder. |

### Economy / catalog / tools

| Issue | Where | Notes |
|---|---|---|
| Spawn ignores availability dates | `GetItemListByType` | Codex uses `isItemAvailable`; spawn does not. `WOOD_1` is in window again (`2027-11-30`); this can recur when a date lapses. |
| `RestorePlayerData` uses `known`/`recent` | `ItemCatalogModule` | Live path uses `Known`/`Recent`. Unused today (`UpdatePlayerData`). |
| Two durability models | `ToolConfig.Durability` vs `Damage` 0–100 | Repair is `5 * Tier * 5` coins for 5 damage. Config 30/60/… unused. Live wear is `DamageEquippedTool`. |
| `CanToolHarvestItem` nil-index | `ToolModule` | Guards `not toolId` but a **unknown string** still does `TOOLS[toolId].Type`. |
| `AddCoins` unsigned | `PlayerDataManager` | Negative amount is a deduct with no floor. |
| `AddItem` still tool-only | `PlayerInventoryManager` | Gameplay must use `TryCollect`. Do not add a remote on `AddItem`. |

### Client / FTUE

| Issue | Where | Notes |
|---|---|---|
| FTUE forage/sell/journey poll + print | Stage `HandleAsync` | `while … print … task.wait(3)` until collect/sell/leave hub. Lifecycle cancel exists; log spam remains. |
| `ScreenOverlayNEW` | `StarterGui` | Health/compass/alerts/quick slots — no owner script. |
| Studio vs live ProfileStore | `GetStoreName` | Opening this place file under live `PlaceId` in Studio **warns** but still uses the live store. |

---

## Remaining P2 (do not expand)

- Disabled: `DungeonMaterializer` V1/V2/v3/ORIGINAL, `RoomFurnisherOLD`, `Draft.*`, `PlayerBarrier`.
- World: leftover `DUNGEON8`, `Avo's Workspace`, Workspace-root oasis templates, `ServerStorage.Old Map`, hub survey art for unenabled sites.
- UI: `Testing`, `BagGUITEST`.
- Commented: levels/XP, badges, old analytics, loading-screen boot in `ReplicatedFirst.Start`.
- Typos: `PROFILE_STORE_DEBUB`, `REDEEM_CODES_MAX_LENGHT`.
- `WorkspaceSetup` hard-coded fog CFrame.
- Water Block scripts duplicated across Map, templates, and rooms.
- `ToolModule.HarvestWithTool` / `CheckDurability` unused.

---

## What is in decent shape

- ProfileStore session lock + reconcile + GDPR `AddUserId` + `RedeemCodes` on template
- Session: `CharacterAdded` is character-only; FTUE once per join; client waits on `loaded`
- `DestinationConfig.KEYS` enable list (`HOME`, `DUNGEON10`, `DUNGEON11`, `DUNGEON5x5`) + Depart sort
- `TeleportModule` owns PivotTo + one loading GUI + `SetLocation`; unstick on HOME→entry
- Hub enter: untagged/`ToRoom` doorway (not Destinations menu)
- `TryCollect` + KEYS-only replenish
- `DeductCoins` / `SellAll` stack pricing / shop `GetPlayerData` + `GetToolInfo`
- HUD LocalScripts on `StarterPlayerScripts`; HUD GUIs persist across respawn
- Door debounce via `task.delay`

---

## Suggested next engineering

1. **Archive** — Draft, old materializers, Old Map, Avo’s Workspace, unused destination folders, test GUIs. Leftover tagged **doors** in `DUNGEON8` still teleport.
2. Align site id / folder / “Level N”, or document the mapping next to `KEYS`.
3. Filter spawn with `isItemAvailable` (same dates as catalog).
4. `CanToolHarvestItem` nil-safe; pick one durability model; `AddCoins` reject negatives.
5. Quiet or event-drive FTUE forage/sell waits.
6. Confirm Coconana (`DUNGEON11`) hub walk-in after the doorway was snapped onto `CoconanaOasisEntrance`.
