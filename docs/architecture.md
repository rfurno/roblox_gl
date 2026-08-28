# Gachamon Legends — Architecture

Last updated: 2026-08-28  
Source of truth: Roblox Studio place **Gachamon Legends (Development)**. This git repo holds documentation only; there is no Rojo/Knit tree.

---

## Shape

Classic Roblox service layout, not a framework:

| Location | Role |
|---|---|
| `ServerScriptService` | Server scripts + modules (data, teleport, inventory, gear, FTUE, dungeon gen) |
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
  ProfileStore session (store name from place id)
  Reconcile against Template
  Init inventory, gear, catalog
  Fire PlayerDataLoaded
  FtueManagerServer.onPlayerAdded once
  announcements once
```

Studio vs live is selected in `ServerConfiguration` by `game.PlaceId`.

---

## Data

`PlayerDataManager.Profiles[player]` holds the live ProfileStore profile.

Important: `SetLocation` only accepts keys in `DestinationConfig.KEYS`. Depart listing only includes `DESTINATIONS` entries whose key is also in `KEYS`. That pair is the site enable list. On load, profile `Location` `DUNGEON1` or `DUNGEON14` is rewritten to `DUNGEON5x5` (Buzzing Plains folder rename).

On leave: inventory, catalog (known/recent), and gear are written back, then the session ends. Duplicate session lock kicks the player.

`DeductCoins` returns `false` without changing balance if the amount is invalid or `Coins < amount`; `true` on success.

---

## Gameplay systems

### Collect (`MaterialReplenishModule`)

Tagged parts in **enabled** destination folders (`KEYS`, not leftover `DUNGEON8`) get a weighted random item, a proximity prompt, and `MaterialItemId`. Hub FTUE `Forage1` is set up separately.

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
| `TeleportPlayerToSpawn` | Go Home / in-maze `IsEntry` / `IsExit` | Yes, name `"Home"` (`LoadingScreen`) |

HOME → site from **Studio Depart** uses `TeleportPlayerToDestination` (unstick `LookVector * 3 + (0, 3, 0)` on the in-maze `IsEntry` marker).

**Production enter** is walking a hub `DungeonEntryDoorway` volume: marker is **not** `IsEntry`; it has `DungeonId` + `ToRoom` + `ToDoorDirection`. `SiteTeleportController` treats it as room-to-room into the first room.

`TeleportHandler` listens to `TeleportRequest` (Depart UI) and routes to destination or spawn.

`SiteTeleportController` listens to **tagged** `TeleportTrigger` parts parented to `DungeonDoorway` markers:

- `IsEntry` or `IsExit` → spawn (HOME)
- `UnusedDoor` → ignore
- else find the marker with matching `DungeonId` + `ToRoom` + `ToDoorDirection`, pivot with the unstick offset

`EnteredRoom` (dungeonId, roomId) drives `CameraTransitionControl` and `uiCoordinator.OnSite`. `nil, nil` means hub.

Door debounce: `Touched` attribute, cleared after 0.5s via `task.delay`.

**Buzzing Plains (`DUNGEON5x5`):** in `KEYS`. Hub `DungeonEntryDoorway` marker is `DungeonId` `DUNGEON5x5`, `ToRoom` `5_3`, `ToDoorDirection` `S`. Landing is `Room_5_3` south (`IsEntry`). Hub `TeleportTrigger` is tagged and aligned to the previous DUNGEON14 gate on `BuzzingPlainsEntrance`. Hub walk-in playtested 2026-08-28.

### FTUE

Server `FtueManagerServer` runs one in-flight `HandleAsync` per player (cancel on leave / stage change). Client mirrors with beams / CTA. Stages still poll (forage until bag non-empty).

### Dungeons

Rooms are pre-baked under `Workspace.Destinations.DUNGEONn`. Generators (`DungeonMaterializer` / V2 / v3 / ORIGINAL) are **Disabled**.

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

### Dungeon → HOME

```
Go Home button → TeleportRequest(HOME)
  or in-maze IsEntry / IsExit door
    → TeleportModule.TeleportPlayerToSpawn
         MazeLoadingScreen show("Home")
         EnteredRoom(nil, nil)
         PivotTo(SpawnLocation + 4y)
         SetLocation(HOME)
```

---

## What is *not* in this architecture

- Rojo file sync, packages, or a server framework
- A single client “app” module — many LocalScripts start themselves
- Runtime maze generation (disabled)
- Authoritative anti-cheat beyond server-owned coins / inventory / gear / `TryCollect`
