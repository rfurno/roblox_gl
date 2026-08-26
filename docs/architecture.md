# Gachamon Legends — Architecture

Last updated: 2026-08-26  
Source of truth: Roblox Studio place **Gachamon Legends (Development)**. This git repo holds documentation only; there is no Rojo/Knit tree.

---

## Shape

Classic Roblox service layout, not a framework:

| Location | Role |
|---|---|
| `ServerScriptService` | Server scripts + modules (data, teleport, inventory, gear, FTUE, dungeon gen) |
| `ReplicatedStorage` | Shared config, remotes, catalog, collectible templates, GUI prefabs |
| `ReplicatedFirst` | Client boot (`Start` waits for load, then player data + FTUE) |
| `StarterPlayer` | LocalScripts (player + character) |
| `StarterGui` | ScreenGuis cloned per player |
| `ServerStorage` | Tools, site templates, old map, barriers |
| `Workspace` | Hub map, destination rooms, FTUE props, art sandboxes |

Communication is RemoteEvents / RemoteFunctions under `ReplicatedStorage.Events`, plus a few BindableEvents for client-only GUI state.

World interaction is **CollectionService tags** (`TagEnumType`) plus attributes on doorway markers. Config is **Luau tables** (`ItemConfig`, `ToolConfig`, `DestinationConfig`). Persistence is **ProfileStore**.

---

## Boot

```
ReplicatedFirst.Start
  waitForGameLoadedAsync
  PlayerData.Client.Start()
  FtueManagerClient.Start()

Server: PlayerDataInit
  ProfileStore session (store name from place id)
  Reconcile against Template
  Init inventory, gear, catalog
  Fire PlayerDataLoaded
  On character: FTUE + announcements
```

Studio vs live is selected in `ServerConfiguration` by `game.PlaceId`.

---

## Data

`PlayerDataManager.Profiles[player]` holds the live ProfileStore profile.

Important: `SetLocation` only accepts keys in `DestinationConfig.KEYS`. Depart listing only includes `DESTINATIONS` entries whose key is also in `KEYS`. That pair is the site enable list.

On leave: inventory, catalog (known/recent), and gear are written back, then the session ends. Duplicate session lock kicks the player.

---

## Gameplay systems

### Collect (`MaterialReplenishModule`)

Tagged destination parts get a weighted random item, a proximity prompt, and `MaterialItemId`. On collect, the server checks the equipped tool, adds to inventory, and updates the catalog. Neutral (empty) parts restock every 120 seconds.

### Inventory / sell

`PlayerInventoryManager` keeps `{ [userId] = { [itemId] = count } }` in memory. Persist via player data on remove.

Sell-all that works: `SellAllProximityPrompt` on the `Buyer` tag → `SellAll` → `PlayerDataManager.AddCoins`.

There is also a `Sale` RemoteEvent handler in `PlayerInventoryManager` that is **broken** (treats a number as an IntValue). Do not use it. See [backlog](backlog.md).

### Gear

`PlayerGearManager`: per-player map of instance ids → `{ ToolId, Equipped, Damage }`. Remotes for purchase, equip, repair. Max damage 100.

### Teleport

All character moves go through `TeleportModule`:

| API | When | Loading screen |
|---|---|---|
| `TeleportPlayerToDestination` | Depart HOME → site | Yes, site name (`StarterGui.LoadingScreen`) |
| `TeleportPlayerToRoom` | Door → matching door | Only if `GetLocation` ≠ this dungeon (first arrival) |
| `TeleportPlayerToSpawn` | Go Home / entry / exit door | Yes, unnamed (`MazeLoadingGui`) |

`TeleportHandler` listens to `TeleportRequest` (Depart UI) and routes to destination or spawn.

`SiteTeleportController` listens to `TeleportTrigger` parts parented to `DungeonDoorway` markers:

- `IsEntry` or `IsExit` → spawn (HOME)
- `UnusedDoor` → ignore
- else find the marker with matching `DungeonId` + `ToRoom` + `ToDoorDirection`, pivot with a LookVector offset so the player does not land on the trigger

`EnteredRoom` (dungeonId, roomId) drives `CameraTransitionControl` (scriptable camera per room, short blackout) and `uiCoordinator.OnSite`. `nil, nil` means hub: reset camera, Destinations visible / Go Home hidden.

Door debounce: `Touched` attribute, cleared after 0.5s via `task.delay`.

### FTUE

Server `FtueManagerServer` runs a stage handler to completion, then advances. Client mirrors with beams / CTA on hub props. Stages poll (e.g. forage waits until inventory is non-empty).

### Dungeons

Rooms are pre-baked under `Workspace.Destinations.DUNGEONn`. `DungeonGenerator` + `DungeonMaterializer` / V2 / v3 / ORIGINAL exist but are **Disabled**. `RoomFurnisherOLD` is leftover.

---

## Client

`ReplicatedFirst.Start` is the only client entry that waits for replication, then starts data + FTUE.

Controllers:

- **StarterPlayerScripts** (once per session): `PlayerInitialSetup`, `UICoordinator`, `LocalDataController`, `Depart` is **not** here — Depart is on the **character**. Camera, proximity prompt skins, announcements, jump parts, feedback NPC, maze loading screen.
- **StarterCharacterScripts** (every spawn): bag, gear, store, blacksmith, coins, depart, codex, notifications, settings, music.

`UICoordinator` is a tiny shared table (`OnSite`, `MusicMuted`, UI state enums). `GuiState` BindableEvent refreshes Depart button visibility.

Maze loading client (`MazeLoadingScreenController`) currently **splits on the second remote argument**:

- name present → enable `StarterGui.LoadingScreen`
- name absent → enable dynamically created `MazeLoadingGui`

That is why HOME → dungeon must pass the site name, and why room-to-room must not fire this remote.

---

## Events (`ReplicatedStorage.Events`)

| Remote | Direction | Use |
|---|---|---|
| `TeleportRequest` | C → S | Depart / Go Home |
| `EnteredRoom` | S → C | Camera + OnSite |
| `MazeLoadingScreen` | S → C | `"show"` / `"hide"`, optional site name |
| `PlayerDataLoaded` / `PlayerDataUpdated` | S → C | Profile |
| `CoinsChangedRemote` | S → C | Coin HUD |
| `PlayerInventoryLoaded` / `Updated` / `Add` | S ↔ C | Bag |
| `GetPlayerInventory` | C → S (fn) | Bag snapshot |
| `GetPlayerGear`, `EquipToolRemote`, `PurchaseTool`, `CheckToolPurchase`, `RepairToolRemote`, `CheckToolRepair`, `IsToolEquipped` | gear shop |
| `OpenBlacksmithUI` | S → C | Open repair UI |
| `UpdateItemCatalog` | S → C | Codex |
| `Sale` | C → S | **Unused / broken** — sell uses proximity prompt |
| `DisplayAnnouncement` / `AnnouncementRemote` | S → C | What’s New / toasts |
| `PlayClientSideSound` / `StopClientSideSound` | S → C | SFX |
| `MazeLoadingScreen` | see above | |
| `SettingsChanged` | Bindable | Music mute |
| `GuiState` | Bindable | Depart buttons |

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

### HOME → dungeon (Depart)

```
DepartGuiController
  → TeleportRequest(destinationKey)
    → TeleportHandler
      → TeleportModule.TeleportPlayerToDestination
           MazeLoadingScreen show(name)
           EnteredRoom(dungeonId, entryRoomId)
           PivotTo(entry doorway)
           MazeLoadingScreen hide
           SetLocation(dungeonId)
        → CameraTransitionControl: OnSite true, room camera
        → Depart: Destinations hidden, Go Home shown
```

### Room → room

```
TeleportTrigger.Touched
  → SiteTeleportController
      (skip if UnusedDoor / IsEntry / IsExit)
      find destination doorway by attributes
      TeleportModule.TeleportPlayerToRoom(..., showLoading = location ~= dungeonId)
      offset PivotTo so the player is off the trigger
        → EnteredRoom only (camera fade) when already in that dungeon
```

### Dungeon → HOME

```
Go Home button → TeleportRequest(HOME)
  or IsEntry / IsExit door
    → TeleportModule.TeleportPlayerToSpawn
         MazeLoadingScreen show (no name)
         EnteredRoom(nil, nil)
         PivotTo(SpawnLocation + 4y)
         SetLocation(HOME)
```

---

## What is *not* in this architecture

- Rojo file sync, packages, or a server framework
- A single client “app” module — many LocalScripts start themselves
- Runtime maze generation (disabled)
- Authoritative anti-cheat beyond “server owns coins / inventory / gear”
