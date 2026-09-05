# Gachamon Legends — Architecture

Last updated: 2026-09-05 (FTUE why-copy on Live 2315)  
Source of truth: Roblox Studio place **Gachamon Legends (Development)**. Live/Alpha is `115297023432140` (universe `8330572807`). This git repo holds documentation only; there is no Rojo/Knit tree.

---

## Shape

Classic Roblox service layout, not a framework:

| Location | Role |
|---|---|
| `ServerScriptService.Teleport` | `TeleportModule`, `TeleportHandler`, `SiteTeleportController` |
| `ServerScriptService.Collect` | `MaterialReplenishModule` + init |
| `ServerScriptService.Economy` | inventory, gear (purchase/equip/repair remotes), sell-all, blacksmith prompt |
| `ServerScriptService.World` | `DamageSystem`, `WorkspaceSetup` |
| `ServerScriptService.Data` | ProfileStore session, `PlayerDataManager`, Template |
| `ServerScriptService.FTUE` | stage handlers |
| `ServerScriptService.Analytics` | FTUE onboarding funnel (`FtueAnalytics`) |
| `ServerScriptService.LevelGeneration` | Studio bake only (Disabled) |
| `ReplicatedStorage` | Shared config, remotes, catalog, collectible templates, GUI prefabs |
| `ReplicatedFirst` | Client boot (`Start` waits for load, then player data + FTUE) |
| `StarterPlayer` | LocalScripts (`StarterPlayerScripts` — session HUDs; `StarterCharacterScripts` empty) |
| `StarterGui` | ScreenGuis cloned per player (`ResetOnSpawn = false` on HUD overlays) |
| `ServerStorage` | Tools (unused by scripts today), site templates (`SiteModelTemplates`), barriers |
| `Workspace` | Hub map, destination rooms, FTUE props, `Avo's Workspace` (do not remove) |

Communication is RemoteEvents / RemoteFunctions under `ReplicatedStorage.Events`, plus a few BindableEvents for client-only GUI state.

World interaction is **CollectionService tags** (`TagEnumType`) plus attributes on doorway markers. Config is **Luau tables** (`ItemConfig`, `ToolConfig`, `DestinationConfig`) plus **ConfigService** `Announcements` (What’s New). Persistence is **ProfileStore**.

---

## Boot

```
ReplicatedFirst.Start
  waitForGameLoadedAsync
  PlayerDataClient.Start()     -- listen + GetPlayerData invoke if FireClient was missed
  PlayerDataClient.loaded.Event:Wait()
  FtueManagerClient.Start()

Server: PlayerDataInit
  CharacterAdded → walk speed / IsAlive only (connected before ProfileStore yield)
  ProfileStore session (store name from place id; unknown id → development store)
  Reconcile against Template
  SetLocation(HOME) — join always starts at hub
  Init inventory, gear, catalog
  Fire PlayerDataLoaded
  FtueManagerServer.onPlayerAdded once
  announcements once
```

Studio vs live is selected in `ServerConfiguration` by `game.PlaceId`.

---

## Data

`PlayerDataManager.Profiles[player]` holds the live ProfileStore profile.

Important: `SetLocation` only accepts keys in `DestinationConfig.KEYS`. Depart listing includes `DESTINATIONS` entries whose key is also in `KEYS`, sorted by `DisplayOrder` (Coconana L1 → Coconana L2 → Buzzing Plains L1 → Buzzing Plains L2 → Blackthorn). Sites with `RequiresComplete` stay in KEYS (so collect can stock them) but teleport and Depart hide them until every `Room_*` of that prerequisite is in `ExploredRooms`. On join, `Location` is set to `HOME`. It is written again on teleport; it is not restored into a dungeon.

`ExploredRooms` is `{ [dungeonId] = { [roomId] = true } }`. `TeleportModule` marks a room when `dungeonId` and `roomId` are set on a land. Hub `EnteredRoom(nil, nil)` does not mark.

On leave: inventory, catalog (known/recent), and gear are written back, then the session ends. Duplicate session lock kicks the player.

`DeductCoins` returns `false` without changing balance if the amount is invalid or `Coins < amount`; `true` on success.

`GetAnnouncements` / `AddAnnouncement` init `profile.Data.Announcements = {}` if the field is missing (same pattern as `RedeemCodes`).

---

## Gameplay systems

### Collect (`MaterialReplenishModule`)

Tagged parts in **enabled** destination folders (`KEYS`) get a weighted random item, a proximity prompt, and `MaterialItemId`. Hub FTUE `Forage1` is set up separately.

`setPartActive` builds the prompt as `"Collect " .. item.Name` with `ObjectText` `Requires <tool type>` when `GetItemRequiredTool` is not `BareHands` (empty otherwise). Attribute `CollectPrompt = true` (Benji / blacksmith / feedback prompts do not set this). Style is custom `Example`. `PromptButtonHoldBegan` plays the harvest sound only if `CanPlayerHarvestItem` is true.

Grant path is **`TryCollect(player, part)`**: part tagged, live `MaterialItemId`, server range (closest point vs `HumanoidRootPart`), equipped tool (or bare hands). Then bag, clear that node, wear tool, catalog. Neutral parts restock every 120 seconds.

Wrong tool: `TryCollect` returns false, plays `HARVEST_DENIED`, and does **not** fire a HUD toast. The client already showed the deny on the prompt and the part.

`PlayerInventoryManager.AddItem` is still a tool-only grant used **after** `TryCollect`. Do not expose it on a remote.

Do not flip shared prompt `ActionText` / `HoldDuration` / `Enabled` or `part.Color` per player. Deny UI is local only.

### Inventory / sell

`PlayerInventoryManager` keeps `{ [userId] = { [itemId] = count } }` in memory. Persist via player data on remove.

Sell-all: `SellAllProximityPrompt` on the `Buyer` tag → `SellAll` (`price * count`) → `AddCoins`. There is no `Sale` remote.

### Gear

`PlayerGearManager`: per-player map of instance ids → `{ ToolId, Equipped, Damage }`. Remotes for purchase, equip, repair. Max damage 100 (live wear). `HasTool` looks up catalog `toolId`. Shop afford-check uses `GetPlayerData` / `GetToolInfo` (`GetToolInfo` is an alias of `GetToolById`).

`RepairToolRemote` always returns remaining **Damage** as a number (current value if the repair did not run: already 0 or broke). Blacksmith UI is nil-safe.

`ToolModule` is the catalog + harvest check: `CanToolHarvestItem` (silent `false` on miss), `GetTools`, `GetToolPrice`, `GetToolRepairCost` (`5 * Tier`). Collect goes `TryCollect` → `CanPlayerHarvestItem` → `CanToolHarvestItem`, then `DamageEquippedTool`.

Equip does not clone a Tool into the character. `ServerStorage.Tools` holds Willow models and `ToolConfig.ToolName` points at them, but no live script parents those tools.

### Announcements

`AnnouncementModule` loads What’s New from **ConfigService** `GetValue("Announcements")`. If that key is missing (live after a script copy), it uses `{}` and warns — it does not iterate nil. Copying scripts does **not** copy ConfigService values. Add the same `Announcements` table on each published place to show What’s New.

### Teleport

All character moves go through `TeleportModule`:

| API | When | Loading screen |
|---|---|---|
| `TeleportPlayerToDestination` | Studio Destinations HOME → site | Yes, site name (`LoadingScreen`) |
| `TeleportPlayerToRoom` | Door → matching door (including **hub `DungeonEntryDoorway`**) | Only if `GetLocation` ≠ this dungeon (first arrival) |
| `TeleportPlayerToSpawn` | Go Home / in-maze `IsExit` | Yes, name `"Home"` (`LoadingScreen`). Facing matches `SpawnLocation` (join). |
| `TeleportPlayerOutsideSite` | In-maze `IsEntry` | Yes, `"Home"`. Lands on the town side of that site’s hub `DungeonEntryDoorway`. |

HOME → site from **Studio Depart** uses `TeleportPlayerToDestination` (unstick `LookVector * 3 + (0, 3, 0)` on the in-maze `IsEntry` marker).

**Production enter** is walking a hub `DungeonEntryDoorway` volume: marker is **not** `IsEntry`; it has `DungeonId` + `ToRoom` + `ToDoorDirection`. `SiteTeleportController` treats it as room-to-room into the first room.

Hub doors are parented under `Workspace.Destinations.<KEY>.DungeonEntryDoorway` (not under `Workspace.Map`). The models are CFramed onto the survey-trip gates in `Map.Decor.SurveyTripEntrances`. Art and trigger live in two trees.

`TeleportHandler` listens to `TeleportRequest` (Depart UI) and routes to destination or spawn.

`SiteTeleportController` listens to **tagged** `TeleportTrigger` parts parented to `DungeonDoorway` markers:

- `IsExit` → hub spawn (`TeleportPlayerToSpawn`), same facing as join
- `IsEntry` → just outside this dungeon’s hub `DungeonEntryDoorway` (town side; hub trigger debounced so you do not walk back in)
- `UnusedDoor` → ignore
- `DungeonId` not in `KEYS` → ignore
- Hub walk-in to a site with `RequiresComplete` while that L1 is unfinished → no teleport (no HUD toast). A `SiteLockPrompt` on the hub doorway shows `Locked` / `Complete <L1 name> <L1 description> first` (large custom prompt). `SiteLockPromptClient` hides it locally once unlocked.
- else find the marker with matching `DungeonId` + `ToRoom` + `ToDoorDirection`, pivot with the unstick offset

`EnteredRoom` (dungeonId, roomId) drives `CameraTransitionControl` and `uiCoordinator.OnSite`. `nil, nil` means hub.

Door debounce: `Touched` attribute, cleared after 0.5s via `task.delay`.

**Buzzing Plains (`DUNGEON5x5`):** 13 rooms, all reachable from the hub. Entry `Room_5_3` south (`IsEntry`); exit `Room_1_3` north. Paired doors `5_3` E ↔ `5_2` W and `2_3` W ↔ `2_4` E were opened 2026-09-03 so `5_2` and the `2_4`/`3_4` island join the maze. Hub walk-in playtested 2026-08-28.

**Buzzing Plains L2 (`DUNGEON5x5L2`):** display name `Buzzing Plains Lvl.2`. 16 rooms (same size as Coconana L2; 4×4 `BuzzingSavannahTemplate` bake). Hub `DungeonEntryDoorway` on `BuzzingPlainsEntranceL2` (Savannah art). `ToRoom` `4_2` `S`. `RequiresComplete = DUNGEON5x5`. Locked gate uses the same `Locked` proximity prompt. Maze rooms offset +2500 Z from START so they do not overlap L1 / Coconana L2.

**Coconana (`DUNGEON11`):** hub `ToRoom` `4_2` `S`. `Room_4_3` north pairs with `Room_3_3` south (south doorway added 2026-08-30 after a bake miss). That pair playtested.

**Coconana L2 (`DUNGEON11L2`):** display name `Coconana Oasis Lvl.2`. 16 rooms, all connected (4×4 `CoconanaOasisTemplate` bake, `targetRooms=16`). Hub on `CoconanaOasisEntranceL2`. `ToRoom` `4_2` `S`. Exit `1_2` N. `RequiresComplete = DUNGEON11`. Maze rooms around X≈4900–5360.

**Blackthorn (`DUNGEON10`):** hub `ToRoom` `10_5` `S`. Marker around Y≈65, snapped to `BlackthornMountainEntrance` at ≈(-402, 73, 483).

### FTUE

Server `FtueManagerServer` runs one in-flight `HandleAsync` per player (cancel on leave / stage change). Client `FtueManagerClient` maps `FtueStage` → handlers:

| Stage | Client hint today |
|---|---|
| `Forage` | Beam + CTA arrow on `Workspace.FTUE.Forage1` |
| `Sell` | Beam + CTA on Benji |
| `Journey` | Beam + CTA on `Workspace.FTUE.JourneyEntry`; server then sets `Complete` |
| `Feedback` | Unused by the linear FTUE. Hub `FeedbackSpawn` prompt still exists (`SocialService:PromptFeedbackSubmissionAsync`). Not a funnel step. |
| `Complete` | No handler |

Forage / sell / journey wait on inventory and location signals. Stage changes log to Creator Hub via `FtueAnalytics` (**Forage=1, Sell=2, Journey=3, Complete=4**). `Feedback` is not logged.

**Why-copy (Live 2315):** client-only. `FtueManagerClient.onFtueStageUpdated` calls `FtueBanner.ShowStage`. Copy lives in `ReplicatedStorage.FTUE.FtueWhyCopy`. The banner is a Frame parented at runtime to `PlayerGui.MainGui` — **no new ScreenGui**, no `ServerScriptService.FTUE` change. Forage why+Do from second 0; intro is a session-local extra line (X / ~8s / 8+ studs closer to `Forage1`). Hide on `Complete`, `Feedback`, and while `LoadingScreen.Enabled`. Keep beams/CTA. Strings: [product](product.md#core-loop). Playtested in Live Studio after `SetFtueStage(Forage)` on the prod profile.

### Dungeons

Live rooms are pre-baked under `Workspace.Destinations` (`DUNGEON11`, `DUNGEON11L2`, `DUNGEON5x5`, `DUNGEON5x5L2`, `DUNGEON10`). Runtime generation is off.

Studio bake lives in `ServerScriptService.LevelGeneration` and stays **Disabled**. Run `DungeonMaterializerv3` from the **Command Bar**: copy the source, edit the locals, paste, run. Command Bar has no `script` instance, so instance attributes are not knobs.

Bake locals: `dungeonId`, `M`, `N`, `targetRooms`, `templateSet`, `snapToSurvey`. `GenerateDungeon` accepts optional `targetRooms` and then pairs extra doors so every room is reachable from entry. Output is `Workspace.GeneratedLevels.<dungeonId>` (needs the `START` part). Move the folder into `Workspace.Destinations` when it should ship. Offset rooms so a new bake does not overlap existing mazes.

`templateSet` is auto-detected from `ServerStorage.SiteModelTemplates`:

| Kind | Example | How rooms are chosen |
|---|---|---|
| Layout set | `CoconanaOasisTemplate` + suffixes `1DE` … `4D` | Pick the model whose pre-baked doors match the cell. Closed walls are art. |
| Single 4-door | `BuzzingSavannahTemplate` (exact name, all four `DungeonDoorway_*`) | Clone that model every cell. Walls without a maze door hide the doorway (transparent, no collide/touch, `UnusedDoor`). |

If any suffix model exists, layout-set wins. Do not enable these scripts in a published place.

`DungeonMaterializerv3ORIGINAL` is the unmodified copy. `RoomFurnisher` still runs on each cloned room during bake.

---

## Client

`ReplicatedFirst.Start` waits for replication, data handshake, then FTUE.

Controllers are **session-length** under `StarterPlayerScripts`: PlayerInitialSetup, UICoordinator, LocalDataController, camera, proximity skins, announcements, jump parts, feedback NPC, maze loading, **Depart, bag, coins, gear, store, blacksmith, codex, settings, notifications, sound, music, SiteLockPromptClient**.

Collect deny lives on the custom prompt path:

- `CustomProximityPrompts` (`Example` skin). Collect `ObjectText` is 18px on a 168×88 billboard. Site-lock prompts use 26px action / 28px object text on a 340×140 billboard.
- `CustomProximityPrompts.CollectHarvestClient` caches `GetPlayerGear`, evaluates `CanToolHarvestItem` against the **equipped** tool, cancels the hold with `InputHoldEnd`, and plays local deny (prompt shake/red copy, `Highlight` on the part, tool-icon `BillboardGui`). Copy: `Need a …` / `Equip a …` / `Repair your …` / `Need a stronger …`. Gear is refreshed while a collect prompt is shown.
- `Notifications` handles `TOOL_BROKEN`. `TOOL_REQUIRED` is still in `ClientConfiguration` but collect does not fire it.

`StarterCharacterScripts` has no HUD LocalScripts.

`UICoordinator` is a tiny shared table (`OnSite`, `MusicMuted`, UI state enums). `MusicMuted` is loaded from / saved to the profile. `GuiState` BindableEvent refreshes Depart button visibility.

`MazeLoadingScreenController` always uses `StarterGui.LoadingScreen` and sets `SurveyTripName`.

---

## Events (`ReplicatedStorage.Events`)

| Remote | Direction | Use |
|---|---|---|
| `TeleportRequest` | C → S | Depart / Go Home |
| `EnteredRoom` | S → C | Camera + OnSite |
| `MazeLoadingScreen` | S → C | `"show"` / `"hide"` + trip name |
| `PlayerDataLoaded` / `PlayerDataUpdated` | S → C | Profile |
| `GetPlayerData` | C → S (fn) | Snapshot if `PlayerDataLoaded` was missed; coin HUD pull |
| `CoinsChangedRemote` | S → C | Coin HUD |
| `PlayerInventoryLoaded` / `Updated` | S → C | Bag |
| `GetPlayerInventory` | C → S (fn) | Bag snapshot |
| `GetPlayerGear`, `EquipToolRemote`, `PurchaseTool`, `CheckToolPurchase`, `RepairToolRemote`, `CheckToolRepair`, `IsToolEquipped` | gear shop |
| `OpenBlacksmithUI` | S → C | Open repair UI |
| `UpdateItemCatalog` | S → C | Codex |
| `DisplayAnnouncement` / `AnnouncementRemote` | S → C | What’s New / `TOOL_BROKEN` toast. Collect deny does not use this. |
| `PlayClientSideSound` / `StopClientSideSound` | S → C | SFX |
| `SetMusicMuted` | C → S | Persist music mute on profile |
| `SettingsChanged` | Bindable | Apply mute to volume / icons |
| `GuiState` | Bindable | Depart buttons |

No `Sale` remote. No `PlayerInventoryAdd` remote.

---

## Config modules

| Module | What it gates |
|---|---|
| `DestinationConfigModule` | Enabled sites (`KEYS` + `DESTINATIONS` + `DisplayOrder` + `RequiresComplete`) |
| `ItemConfig` | Harvestables, rarity, tools required |
| `ToolConfig` | Tool ids, types, prices. Live wear is gear `Damage` 0–100. `TIERS[].Durability` / `SpeedFactor` and Sickle / Knife / Hammer types are unused. |
| `ClientConfiguration` | Notification copy / art. Live toast: `TOOL_BROKEN`. `TOOL_REQUIRED` is unused leftover. |
| `ServerConfiguration` | Place ids, datastore names, walk speed, unused debug flags, redeem flag (off) |
| `PlayerDataKey` / `FtueStage` / `TagEnumType` | String enums |
| ConfigService `Announcements` | What’s New. Per-place; not in the DataModel script tree |

---

## Diagrams

### HOME → dungeon (Studio Destinations)

```
DepartGuiController
  → TeleportRequest(destinationKey)
    → TeleportHandler
      → TeleportModule.TeleportPlayerToDestination
           MazeLoadingScreen show(name)
           EnteredRoom(dungeonId, entryRoomId)
           PivotTo(IsEntry marker + unstick)
           MazeLoadingScreen hide
           SetLocation(dungeonId)
```

### HOME → dungeon (production walk-in)

```
Hub DungeonEntryDoorway.TeleportTrigger (tagged) Touched
  → SiteTeleportController
      DungeonId + ToRoom + ToDoorDirection (not IsEntry)
      TeleportModule.TeleportPlayerToRoom(..., showLoading = first arrival)
```

### Room → room

```
TeleportTrigger.Touched
  → SiteTeleportController
      (skip if UnusedDoor / IsEntry / IsExit)
      find destination doorway by attributes
      TeleportModule.TeleportPlayerToRoom(..., showLoading = location ~= dungeonId)
```

### Dungeon → HOME (exit)

```
Go Home button → TeleportRequest(HOME)
  or in-maze IsExit door
    → TeleportModule.TeleportPlayerToSpawn
         MazeLoadingScreen show("Home")
         EnteredRoom(nil, nil)
         PivotTo(SpawnLocation + 4y, SpawnLocation facing)
         SetLocation(HOME)
```

### Dungeon → hub gate (entry door from inside)

```
in-maze IsEntry door
  → TeleportModule.TeleportPlayerOutsideSite(dungeonId)
       MazeLoadingScreen show("Home")
       EnteredRoom(nil, nil)
       PivotTo(town side of Destinations.<id>.DungeonEntryDoorway)
       SetLocation(HOME)
```

---

## What is *not* in this architecture

- Rojo file sync, packages, or a server framework
- A single client “app” module — many LocalScripts start themselves
- Runtime maze generation (disabled; Command Bar bake in `LevelGeneration`, keep off in published places)
- Authoritative anti-cheat beyond server-owned coins / inventory / gear / `TryCollect`
- Session restore into a dungeon — join always `SetLocation(HOME)`
- 3D tool-in-hand. In-dungeon HUD (`ScreenOverlayNEW`) is present and unwired; confirm before deleting. Never remove `Workspace.Avo's Workspace`.

---

## Copying Development → published (Alpha / Testers)

**2026-09-03:** models, scripts, and UI copied to Live/Alpha (`115297023432140`) and published (place version 2305). Live Destinations include `DUNGEON11`, `DUNGEON11L2` (16 rooms), `DUNGEON5x5` (13 rooms, doors patched), `DUNGEON5x5L2` (16 rooms), `DUNGEON10`. Hub art on live: `CoconanaOasisEntranceL2`, `BuzzingPlainsEntranceL2`. `LevelGeneration` scripts are present and **Disabled**. `Avo's Workspace` is not on live.

**2026-09-05:** FTUE why-copy scripts copied to Live and published (**2315**). Client-only: `ReplicatedStorage.FTUE.FtueWhyCopy`, `FtueBanner`, `FtueManagerClient`. No Workspace models, no new ScreenGui, no `ServerScriptService.FTUE` change. Playtested in Live Studio.

Scripts are the same tree. Gameplay to copy after later Dev edits:

- `ReplicatedStorage.FTUE.FtueWhyCopy`, `FtueBanner`, `FtueManagerClient` (Live 2315)
- `Teleport.TeleportModule`, `Teleport.SiteTeleportController`, `Teleport.TeleportHandler`
- `StarterPlayerScripts.CameraTransitionControl`, `DepartGuiController`, `SiteLockPromptClient`
- `ReplicatedStorage.Config.DestinationConfigModule`, `ToolConfig`, `ClientConfiguration`
- `ReplicatedStorage.ToolModule`, `StarterPlayerScripts.StoreController`
- `Economy.PlayerGearManager`, `StarterPlayerScripts.BlacksmithController`
- `Announcements.AnnouncementModule`, `Data.PlayerDataManager`, `Data.PlayerDataInit`, `Data.Template`
- `Config.ServerConfiguration`
- `Collect.MaterialReplenishModule`
- `StarterPlayerScripts.CustomProximityPrompts` (includes child `CollectHarvestClient`)

Do **not** copy (or keep **Disabled**): `LevelGeneration.*`, `Draft.DevLog`.

Not scripts:

- ConfigService `Announcements` — add on the published place or What’s New stays empty
- DataModel doors (e.g. Coconana `Room_3_3` south) — copy the instance or the destination folder, not a script

---

## Live observability (Open Cloud)

Studio MCP talks only to the open Studio instance (usually Development, Edit mode). It cannot read published-server Output. The game itself has no outbound HTTP APIs (`HttpService` is disabled; ProfileStore uses it only for GUIDs).

FTUE steps go to Creator Hub via `AnalyticsService:LogOnboardingFunnelStepEvent` (`ServerScriptService.Analytics.FtueAnalytics`). That is not the same as server Output.

### Auth

Create a key at [Creator Dashboard → Credentials](https://create.roblox.com/dashboard/credentials). Export it in the shell **before** starting Grok so the agent inherits it:

```bash
export ROBLOX_API_KEY='…'
```

Never commit the key, put it in this repo, or paste it into chat. An export in an already-running Grok session is invisible until Grok is restarted from that shell.

| Scope | What it unlocks |
|---|---|
| `universe:read` | List game servers, read per-server Output |
| `universe.analytics:read` | Time-series metrics (CCU, FTUE funnel) |

Header: `x-api-key: $ROBLOX_API_KEY`. Base: `https://apis.roblox.com`.

### IDs

| Place | Place ID | Universe ID |
|---|---|---|
| Development | `136894937108297` | `9865188944` |
| Testers | `72816619326760` | `8925744545` |
| Live / Alpha | `115297023432140` | `8330572807` |

Live last seen 2026-09-05 as place version **2315** (`filter-options`; 2314 public jobs can keep serving until they shut down). Confirm current version with `game-servers:filter-options` — `place-version-history-api` 403s on the `universe:read` key.

### Agent workflow

When asked to check live logs or servers:

1. Confirm `$ROBLOX_API_KEY` is set (`printenv ROBLOX_API_KEY` — do not print the value).
2. Public snapshot (no key): `GET https://games.roblox.com/v1/games?universeIds=8330572807` and `GET https://games.roblox.com/v1/games/115297023432140/servers/Public`.
3. Current place version: `GET https://apis.roblox.com/place-version-history-api/v1/115297023432140/history`
4. Servers: `GET https://apis.roblox.com/server-management/v1/universes/8330572807/places/115297023432140/versions/{version}/game-servers`
5. Logs for a `jobId`: `GET https://apis.roblox.com/server-management/v1/universes/8330572807/places/115297023432140/versions/{version}/game-servers/{jobId}/logs`
6. Optional metrics: `POST https://apis.roblox.com/analytics-query-api/v1/universes/8330572807/metrics` with `universe.analytics:read`.

If the key is missing, those Open Cloud routes return **403** `Invalid authentication data provided`. Empty public `servers` + `playing: 0` means there is no live job to tail.

Related: `GET .../game-servers:filter-options`, `GET .../universes/{id}/restarts`, `GET .../cloud/v2/universes/{id}`.
