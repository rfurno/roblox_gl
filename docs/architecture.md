# Gachamon Legends — Architecture

Last updated: 2026-08-30 (Studio bake: layout-set vs single 4-door templates)  
Source of truth: Roblox Studio place **Gachamon Legends (Development)**. This git repo holds documentation only; there is no Rojo/Knit tree.

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
| `ServerScriptService.LevelGeneration` | Studio bake only (Disabled) |
| `ReplicatedStorage` | Shared config, remotes, catalog, collectible templates, GUI prefabs |
| `ReplicatedFirst` | Client boot (`Start` waits for load, then player data + FTUE) |
| `StarterPlayer` | LocalScripts (`StarterPlayerScripts` — session HUDs; `StarterCharacterScripts` empty of HUD) |
| `StarterGui` | ScreenGuis cloned per player (`ResetOnSpawn = false` on HUD overlays) |
| `ServerStorage` | Tools, site templates, old map, barriers |
| `Workspace` | Hub map, destination rooms, FTUE props, art sandboxes |

Communication is RemoteEvents / RemoteFunctions under `ReplicatedStorage.Events`, plus a few BindableEvents for client-only GUI state.

World interaction is **CollectionService tags** (`TagEnumType`) plus attributes on doorway markers. Config is **Luau tables** (`ItemConfig`, `ToolConfig`, `DestinationConfig`). Persistence is **ProfileStore**.

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

Important: `SetLocation` only accepts keys in `DestinationConfig.KEYS`. Depart listing only includes `DESTINATIONS` entries whose key is also in `KEYS`. That pair is the site enable list. On join, `Location` is set to `HOME`. It is written again on teleport; it is not restored into a dungeon.

On leave: inventory, catalog (known/recent), and gear are written back, then the session ends. Duplicate session lock kicks the player.

`DeductCoins` returns `false` without changing balance if the amount is invalid or `Coins < amount`; `true` on success.

---

## Gameplay systems

### Collect (`MaterialReplenishModule`)

Tagged parts in **enabled** destination folders (`KEYS`) get a weighted random item, a proximity prompt, and `MaterialItemId`. Hub FTUE `Forage1` is set up separately.

Grant path is **`TryCollect(player, part)`**: part tagged, live `MaterialItemId`, server range (closest point vs `HumanoidRootPart`), equipped tool (or bare hands). Then bag, clear that node, wear tool, catalog. Neutral parts restock every 120 seconds.

`PlayerInventoryManager.AddItem` is still a tool-only grant used **after** `TryCollect`. Do not expose it on a remote.

### Inventory / sell

`PlayerInventoryManager` keeps `{ [userId] = { [itemId] = count } }` in memory. Persist via player data on remove.

Sell-all: `SellAllProximityPrompt` on the `Buyer` tag → `SellAll` (`price * count`) → `AddCoins`. There is no `Sale` remote.

### Gear

`PlayerGearManager`: per-player map of instance ids → `{ ToolId, Equipped, Damage }`. Remotes for purchase, equip, repair. Max damage 100. `HasTool` looks up catalog `toolId`. Shop afford-check uses `GetPlayerData` / `GetToolInfo`.

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
- `DungeonId` not in `KEYS` → ignore (workspace-root bake clones still carry tagged markers)
- else find the marker with matching `DungeonId` + `ToRoom` + `ToDoorDirection`, pivot with the unstick offset

`EnteredRoom` (dungeonId, roomId) drives `CameraTransitionControl` and `uiCoordinator.OnSite`. `nil, nil` means hub.

Door debounce: `Touched` attribute, cleared after 0.5s via `task.delay`.

**Buzzing Plains (`DUNGEON5x5`):** in `KEYS`. Hub `DungeonEntryDoorway` marker is `DungeonId` `DUNGEON5x5`, `ToRoom` `5_3`, `ToDoorDirection` `S`. Landing is `Room_5_3` south (`IsEntry`). Hub `TeleportTrigger` is tagged and aligned to the previous DUNGEON14 gate on `BuzzingPlainsEntrance`. Hub walk-in playtested 2026-08-28. In-maze `IsEntry` (town-side gate) and `IsExit` (spawn, join facing) playtested 2026-08-30.

### FTUE

Server `FtueManagerServer` runs one in-flight `HandleAsync` per player (cancel on leave / stage change). Client mirrors with beams / CTA. Forage / sell / journey wait on inventory and location signals.

### Dungeons

Live rooms are pre-baked under `Workspace.Destinations` (`DUNGEON11`, `DUNGEON5x5`, `DUNGEON10`). Runtime generation is off.

Studio bake lives in `ServerScriptService.LevelGeneration` and stays **Disabled**. Run `DungeonMaterializerv3` from the **Command Bar**: copy the source, edit the locals, paste, run. Command Bar has no `script` instance, so instance attributes are not knobs.

Bake locals: `dungeonId`, `M`, `N`, `templateSet`, `snapToSurvey`. Output is `Workspace.GeneratedLevels.<dungeonId>` (needs the `START` part). Move the folder into `Workspace.Destinations` when it should ship.

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

Controllers are **session-length** under `StarterPlayerScripts`: PlayerInitialSetup, UICoordinator, LocalDataController, camera, proximity skins, announcements, jump parts, feedback NPC, maze loading, **Depart, bag, coins, gear, store, blacksmith, codex, settings, notifications, sound, music**.

`StarterCharacterScripts` has no HUD LocalScripts.

`UICoordinator` is a tiny shared table (`OnSite`, `MusicMuted`, UI state enums). `MusicMuted` is loaded from / saved to the profile. `GuiState` BindableEvent refreshes Depart button visibility.

`MazeLoadingScreenController` always uses `StarterGui.LoadingScreen` and sets `SurveyTripName`. Leftover `MazeLoadingGui` is destroyed.

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
| `DisplayAnnouncement` / `AnnouncementRemote` | S → C | What’s New / toasts |
| `PlayClientSideSound` / `StopClientSideSound` | S → C | SFX |
| `SetMusicMuted` | C → S | Persist music mute on profile |
| `SettingsChanged` | Bindable | Apply mute to volume / icons |
| `GuiState` | Bindable | Depart buttons |

No `Sale` remote. No `PlayerInventoryAdd` remote.

---

## Config modules

| Module | What it gates |
|---|---|
| `DestinationConfigModule` | Enabled sites (`KEYS` + `DESTINATIONS`) |
| `ItemConfig` | Harvestables, rarity, tools required |
| `ToolConfig` | Tool ids, tiers, prices |
| `ClientConfiguration` | Notification copy / art |
| `ServerConfiguration` | Place ids, datastore names, walk speed |
| `PlayerDataKey` / `FtueStage` / `TagEnumType` | String enums |

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
