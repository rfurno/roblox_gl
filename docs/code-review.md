# Gachamon Legends — Code review

Date: 2026-08-27 (follow-up)  
Prior pass: 2026-08-26.  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`).  
Method: re-read live gameplay scripts after the 2026-08-27 Studio pass. Disabled generators and workspace Animate scripts were not line-audited.

Place version notes and `ServerScriptService.Draft.DevLog` (2026-08-27) list the same fixes.

---

## Summary

The 2026-08-26 P0 list is **closed** on the dev place: sell, respawn re-init, harvest remote, shop snapshot, DeductCoins, redeem template, loading GUI, HUD parent, and collect-vs-node. Teleport, session boot, and economy transactions are in decent shape for testers on **Coconana** and **Blackthorn**.

Nothing remaining looks play-breaking for those two sites. Highest leftover risks are **Buzzing Plains parked** (`DUNGEON1` still in `KEYS`, hub trigger untagged), **leftover dungeon doors** that still teleport but do not save location, **WOOD_1 spawn vs catalog dates**, and **world/script archive**. Do not enable more sites or tool tiers until Buzzing Plains is verified or dropped from `KEYS`.

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
| Hub `DUNGEON1` doorway `DungeonId` | Was `DUNGEON11`; now `DUNGEON1`. |

---

## Remaining P0

None for Coconana / Blackthorn testers.

`GetStoreName()` still returns **nil** on an unknown `PlaceId` (`PlayerDataInit`), which would make `ProfileStore.New(nil, …)` fail. Only hits if this place id is not development / testers / live.

---

## Remaining P1

### Sites / teleport

| Issue | Where | Notes |
|---|---|---|
| Buzzing Plains not playable | `Destinations.DUNGEON1.DungeonEntryDoorway` | `DungeonId` is `DUNGEON1`. A part named `TeleportTrigger` **still exists** but has **no** `TeleportTrigger` tag, so `SiteTeleportController` does not bind it. Still in `KEYS` — Studio Destinations can send players in. Parked until a site pass. |
| `IsEntry` and `IsExit` both home | `SiteTeleportController` | In-maze entry/exit markers send you HOME. Hub enter is a **non-**`IsEntry` doorway with `ToRoom` / `ToDoorDirection`. Document on markers. |
| Destinations button Studio-only | `DepartGuiController` | Intentional test skip. Production enter is hub `DungeonEntryDoorway`. |
| Site id vs “Level N” | `DestinationConfig` | DUNGEON11 = “Level 1”, DUNGEON1 = “Level 3”, DUNGEON10 = “Level 10”. |
| Leftover doors still move you | `DUNGEON3/7/8` tagged triggers | `SetLocation` ignores unknown keys. Character teleports; next join thinks HOME. Replenish no longer stocks those folders. |

### Economy / catalog / tools

| Issue | Where | Notes |
|---|---|---|
| `WOOD_1` expired | `ItemConfig` `AvailabilityEndDate = "2025-11-30"` | Codex `isItemAvailable` hides it; `GetItemListByType` (spawn) does **not** filter dates. Oakren Wood still appears in mazes. |
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
- World: `DUNGEON3/7/8`, `Avo's Workspace`, Workspace-root oasis templates, `ServerStorage.Old Map`, hub survey art for unenabled sites.
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
- `DestinationConfig.KEYS` enable list + Depart sort
- `TeleportModule` owns PivotTo + one loading GUI + `SetLocation`; unstick on HOME→entry
- Hub enter: untagged/`ToRoom` doorway (not Destinations menu)
- `TryCollect` + KEYS-only replenish
- `DeductCoins` / `SellAll` stack pricing / shop `GetPlayerData` + `GetToolInfo`
- HUD LocalScripts on `StarterPlayerScripts`; HUD GUIs persist across respawn
- Door debounce via `task.delay`

---

## Suggested next engineering

1. **Buzzing Plains site pass** — verify rooms/`DungeonId`/`IsEntry`, then restore a **tagged** `TeleportTrigger` on `DungeonEntryDoorway`, or remove `DUNGEON1` from `KEYS`.
2. **WOOD_1 dates** — refresh or drop; filter spawn with the same availability check as the catalog.
3. **Archive** — Draft, old materializers, Old Map, Avo’s Workspace, unused destination folders, test GUIs. Leftover tagged **doors** in DUNGEON3/7/8 still teleport.
4. Align site id / folder / “Level N”, or document the mapping next to `KEYS`.
5. `CanToolHarvestItem` nil-safe; pick one durability model; `AddCoins` reject negatives.
6. Quiet or event-drive FTUE forage/sell waits.

Do not enable more sites or tool tiers until (1) is done or `DUNGEON1` is dropped from `KEYS`.
