# Gachamon Legends — Code review

Date: 2026-08-26  
Scope: Studio DataModel on **Gachamon Legends (Development)** (`136894937108297`). No code was changed in this pass.  
Method: full read of live gameplay scripts (data, inventory, gear, collect, teleport, FTUE, client boot, shop/repair). Disabled generators and workspace Animate scripts were sampled, not line-audited.

---

## Summary

The game is a readable tagged-world + config-table + ProfileStore loop. Teleport was recently centralized and is in better shape than the economy and data boot paths. The highest risks are **client-authoritative harvest**, a **`Sale` remote that wipes inventory**, **respawn re-init that overwrites live bags**, and **FTUE stage loops that can stack**. Shop “can I buy?” is currently broken (`GetPlayerData` does not exist). Refactoring should start with one server session-owner and one harvest API, not more features.

---

## P0 — fix before testers / live

### 1. `PlayerInventoryAdd` is a client-trusted grant

- **Where:** `ServerScriptService.PlayerInventoryManager` ~132–138; client `ReplicatedStorage.FTUE.FtueManagerClient.StageHandlers.ForageFtueStage` ~54–58
- **What’s wrong:** Any client can `FireServer(itemId)`. The server only checks “does this item exist and does the equipped tool (or bare hands) allow it.” No distance, no node ownership, no rate limit. Tutorial forage uses this on purpose; dungeon collect uses a *second* path (`MaterialReplenishModule` prompt → `AddItem` directly). Exploiters can fill a bag without visiting a node.
- **Fix:** Remove the remote for gameplay grants. Collect only from the proximity handler (pass the part or a server-issued token). Keep FTUE on the same server path.

### 2. `Sale` remote empties the bag and awards nothing

- **Where:** `PlayerInventoryManager` ~116–125
- **What’s wrong:** `OnServerEvent` calls `SellAll` (clears inventory, notifies client) then:

  ```lua
  local coins = 0
  coins.Value = coins.Value + total
  ```

  That errors. Net effect if anything fires `Sale`: **items gone, no coins**. Working sell is `SellAllProximityPrompt` → `AddCoins`.
- **Fix:** Delete the `Sale` handler (and the remote if unused). Do not “repair” it without also fixing stack pricing (below).

### 3. `SellAll` pays once per item id, not per stack

- **Where:** `PlayerInventoryManager.SellAll` ~100–108
- **What’s wrong:** `total += GetItemBasePrice(itemId)` ignores `count`. Ten olives sell as one.
- **Fix:** `total += price * count`. Add a unit test or Studio dump with mixed stacks.

### 4. Respawn re-init overwrites live inventory/gear

- **Where:** `ServerScriptService.Data.PlayerDataInit` ~166–172, `Initialize` ~60–91, `InitInventory` ~46–55
- **What’s wrong:** `CharacterAdded` waits **8 seconds** then calls `Initialize` again. `InitInventory` writes profile snapshot counts into the live table (`inv[itemId] = count`), wiping anything collected since join/last save. Gear `InitPlayerTools` merges into the same live map. Also re-fires `PlayerDataLoaded` / catalog / coins and **restarts FTUE** (`onCharacterAdded` → `FtueManagerServer.onPlayerAdded`).
- **Fix:** `CharacterAdded` should only set walk speed / alive. Persist or don’t touch inventory on respawn. Never start a second FTUE loop.

### 5. FTUE can run two stage loops at once

- **Where:** `FtueManagerServer.UpdateFtueStage` ~38–56; called from `onPlayerAdded` and again after the 8s respawn path
- **What’s wrong:** Each call `task.spawn`s `HandleAsync` with no cancellation. Forage/Sell poll forever (`while … task.wait(3)`). A second spawn means two loops calling `UpdateFtueStage(next)` — stages skip, analytics double-log, or `stageHandler` is nil if `nextStage` is unexpected.
- **Fix:** One in-flight handler per player; cancel on leave / Complete. Don’t call `UpdateFtueStage` just to refresh UI.

### 6. Shop afford-check is dead

- **Where:** `ToolPurchaseHandler` ~16–36
- **What’s wrong:** `PlayerDataManager.GetPlayerData` **does not exist**. `CheckToolPurchase` errors or always fails. Actual purchase uses `GetCoins` and can still run. UI that asks “can I buy?” is lying.
- **Fix:** Use `GetCoins`. Add `GetPlayerData` only if you want a real snapshot API.

---

## P1 — wrong, racy, or easy to regress

### Economy / tools

| Issue | Where | Notes |
|---|---|---|
| `DeductCoins` never refuses | `PlayerDataManager` ~58–68 | Subtracts then clamps to 0. Two overlapping purchases can both pass `coins < price` and both grant tools. Deduct should return false if `Coins < amount`. |
| `GetToolPrice` / `GetToolRepairCost` nil-index | `ToolModule` ~15–17, ~126–127 | Invalid `toolId` throws instead of returning nil. Purchase handler assumes nil. |
| `HasTool` looks up the wrong key | `PlayerGearManager` ~229–235 | Gear map is keyed by **instance** id; this indexes by catalog `toolId`. Always false. |
| `RepairTool` no nil guard | `PlayerGearManager` ~202–208 | Bad `playerToolId` errors. |
| Repair cost vs durability | `ToolRepairHandler` + `ToolConfig.Durability` | Repair is a flat `5 * Tier * 5` coins for 5 damage. Config `Durability` (30/60/…) is unused; a parallel `Damage` 0–100 system is what actually runs. Two durability models. |
| `CheckToolPurchase` vs `PurchaseTool` | `ToolPurchaseHandler` | Check is broken (above). Purchase is a RemoteFunction named like an Event in comments. |

### Collect / catalog

| Issue | Where | Notes |
|---|---|---|
| `WOOD_1` expired | `ItemConfig` `AvailabilityEndDate = "2025-11-30"` | Codex `isItemAvailable` hides it; spawn (`GetItemListByType`) does **not** filter dates. Oakren Wood still appears in mazes, then looks “unknown/unavailable” in catalog. |
| `RestorePlayerData` uses `known`/`recent` | `ItemCatalogModule` ~188–193 | Live code uses `Known`/`Recent`. If restore is ever called, known items vanish. `UpdatePlayerData` is the path used today. |
| Replenish hits disabled dungeons | `MaterialReplenishModule.getAllMaterialParts` | Every tagged part under `Destinations`, including DUNGEON3/7/8. Extra prompts and 120s work. |
| Prompt connections leak | `setPartActive` | Each replenish `ClearPrompts` then adds new `Triggered` connections. Confirm `ClearPrompts` destroys the prompt instance; if it only hides, handlers stack. |
| Collect uses `item.Id` vs `item.Key` | `MaterialReplenishModule` ~87 vs ~91 | `getItemDetails` sets both; they match today. Fragile if a config row’s `Id` diverges from the dictionary key. |

### Teleport / loading / Depart

| Issue | Where | Notes |
|---|---|---|
| Two loading GUIs | `MazeLoadingScreenController` | Name → `StarterGui.LoadingScreen`; no name → `MazeLoadingGui`. HOME→site vs site→HOME look different. One remote, one GUI. |
| Entry pivot on the trigger | `TeleportModule.TeleportPlayerToDestination` | Pivots to marker CFrame. Room-to-room offsets `LookVector * 3 + (0,3,0)`. `IsEntry` doors send you HOME. First-room landing is luck. |
| `IsEntry` and `IsExit` both home | `SiteTeleportController` | Document or split. |
| Depart button Studio-only | `DepartGuiController` ~51 | Testers/live have no picker unless another entry exists. |
| Site id vs “Level N” | `DestinationConfig` | DUNGEON11 = “Level 1”, DUNGEON1 = “Level 3”. |
| `GetStoreName()` can return nil | `PlayerDataInit` ~35–49 | Unknown place id → `ProfileStore.New(nil, …)`. |

### Client boot / FTUE

| Issue | Where | Notes |
|---|---|---|
| FTUE starts without waiting for data | `ReplicatedFirst.Start` ~29–43 | `HasLoaded()` false → `task.wait(2)` then `FtueManagerClient.Start()` → `PlayerDataClient.Get` **asserts**. Slow join = client error, no tutorial. Should `loaded:Wait()`. |
| Character HUDs | `StarterCharacterScripts` | Bag, gear, store, blacksmith, coins, **Depart**, codex die with the character. |
| `ScreenOverlayNEW` | `StarterGui` | Health/compass/alerts/quick slots — no owner script found. |

### Misc

| Issue | Where | Notes |
|---|---|---|
| Redeem codes | Template vs `PlayerDataManager` | Flag on; no `RedeemCodes` on template; first `table.find(nil)` errors. |
| Deduct/Add coins unsigned | `PlayerDataManager` | Negative `AddCoins` is a deduct with no floor except Deduct’s clamp. |
| `PlayerDataInit` session | `GetStoreName` | Studio + live place id only **warns**, still uses live store if PlaceId matches live. |

---

## P2 — debt (do not expand)

- Disabled: `DungeonMaterializer` V1/V2/v3/ORIGINAL, `RoomFurnisherOLD`, `Draft.*`, `PlayerBarrier`.
- World: `DUNGEON3/7/8`, `Avo's Workspace`, Workspace-root oasis templates, `ServerStorage.Old Map`, hub survey art for unenabled sites.
- UI: `Testing`, `BagGUITEST`.
- Commented: levels/XP, badges, old analytics, loading-screen boot in `ReplicatedFirst.Start`.
- Typos: `PROFILE_STORE_DEBUB`, `REDEEM_CODES_MAX_LENGHT`.
- `WorkspaceSetup` hard-codes a fog CFrame.
- Water Block scripts duplicated across Map, templates, and dungeon rooms.
- `ToolModule.HarvestWithTool` / `CheckDurability` unused; live wear is `DamageEquippedTool` random 0–4.

---

## Important refactoring (order)

Do these as sequenced PRs in Studio (or Rojo, if you ever extract). Each should be playable alone.

### 1. Server session owner

One module started from `PlayerDataInit`:

- Owns ProfileStore session
- `CharacterAdded` only applies character (speed, alive)
- `PlayerRemoving` flushes inventory, gear, catalog, location once
- No 8-second `Initialize`

This unblocks FTUE and bag correctness.

### 2. One harvest API

- Server: `tryCollect(player, part)` — validates tag, `MaterialItemId`, prompt range, tool, then grants, damages tool, neutrals the part, marks catalog.
- Delete `PlayerInventoryAdd` remote (or restrict to Studio).
- FTUE forage should call the same function with the hub `Forage1` part, not a client FireServer.

### 3. Economy transactions

- `DeductCoins` returns false if insufficient; no clamp-as-success.
- `SellAll`: `price * count`; single function used by Benji prompt; remove `Sale` remote.
- Shop/repair: one `TryPurchaseTool` / `TryRepairTool` with validate → deduct → mutate. Fix `GetPlayerData`. Pick **one** durability model (0–100 damage **or** config Durability).

### 4. Teleport / loading (small follow-up)

Already centralized. Remaining:

- One loading GUI; always pass a reason/name.
- Same unstick offset for HOME→entry as room-to-room.
- Non-Studio Depart (always show Destinations, or wire hub survey models).

### 5. Client shell

- Move Depart, bag, gear, store, coins, codex to `StarterPlayerScripts` (or a single `ClientApp` required from `ReplicatedFirst.Start` after data loads).
- `FtueManagerClient.Start` only after `PlayerDataClient.loaded`.
- Cancel FTUE handlers on stage change / leave.

### 6. Delete or archive

Move Draft, old materializers, Old Map, Avo’s Workspace, unused destination folders, test GUIs to a separate unpublished place or `ServerStorage.Archive` with scripts Disabled. Shrinks tag queries (`GetTagged` on forage/mining walks unused dungeons today).

### 7. Config hygiene

- `KEYS` remains the enable list (good).
- Align site number / folder / Level label, or show folder id in UI for debug.
- Drop or refresh `WOOD_1` dates; apply the same availability filter to spawn as to catalog.
- Add `RedeemCodes = {}` to Template or set `REDEEM_CODES_ENABLED = false`.

---

## What is in decent shape

- ProfileStore session lock + reconcile + GDPR `AddUserId`
- `DestinationConfig.KEYS` as the enable list + Depart sort
- `TeleportModule` as the only PivotTo + loading + `SetLocation` owner
- Door debounce via `task.delay` (no stuck `Touched` on early return)
- Tag-driven world (buyer, blacksmith, harvest, doors) — right idea
- Item/tool config tables — right idea, dates and dual durability need cleanup
- FTUE stage enum + analytics step map — structure is fine; lifecycle is not

---

## Suggested review order for the next implementation pass

1. P0 #2–5 (sell, respawn, FTUE loops) — data loss  
2. P0 #1 (harvest remote) — economy exploit  
3. P0 #6 + DeductCoins — shop  
4. Loading GUI unify + entry offset  
5. Client boot wait + HUD parent  
6. World/script archive  

Do not enable more sites or tool tiers until 1–3 are done.
