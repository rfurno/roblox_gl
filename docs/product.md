# Gachamon Legends — Product

Last updated: 2026-09-03  
Place under review: **Gachamon Legends (Development)** (`136894937108297`)

This document describes what the game is, who it is for, and what a session looks like. Implementation lives in Roblox Studio, not in this git repo.

---

## What it is

Gachamon Legends is a gathering / expedition game on Roblox. Players live in a hub town, collect materials with tools, sell them for coins, then travel into pre-built dungeon “sites” (mazes) to forage, mine, and excavate more.

The fantasy is field work: botanica, ores, fossils, and a traveling merchant (Benji) plus a blacksmith. A codex tracks what you have discovered. Tools wear out and can be bought, equipped, and repaired.

It is not a combat RPG. Damage exists as environmental hazards in dungeons (thorns, fire, lava, and similar tags). There is no player-level / XP loop.

---

## Places

| Place | Place ID | Profile store |
|---|---|---|
| Development (this instance) | `136894937108297` | `GLPlayerProfileDevelopment` |
| Testers | `72816619326760` | `GLPlayerProfileTesters` |
| Live | `115297023432140` | `GLPlayerProfileProd` | Published 2026-09-03 (place version 2305): L2 sites + Plains L1 door patch |

Walk speed is 20 in published servers and 28 in Studio.

---

## Core loop

1. **Spawn at HOME** (hub) with 50 coins and FTUE stage `Forage`.
2. **Collect** tagged nodes (forage / mining / excavating) using the required tool or bare hands.
3. **Sell** the bag to Benji (`Sell All Collected Samples`) for coins.
4. **Gear** — buy / equip / repair Willow tools at the traveling blacksmith.
5. **Depart** to a site by walking a hub `DungeonEntryDoorway` (Studio also has a Destinations debug picker). One `LoadingScreen` on HOME ↔ site. Room-to-room uses a short camera fade only.
6. **Explore** rooms, collect, avoid hazards. The in-maze **entry** door returns you just outside that site’s hub gate. The **exit** door and Go Home return you to hub spawn.
7. **Codex** records known and recent items.

New-player FTUE is linear:

`Forage` → `Sell` → `Journey` → `Feedback` → `Complete`

---

## Collecting

Three harvest types, driven by `ItemConfig` + CollectionService tags:

| Type | Typical tool | Examples |
|---|---|---|
| Foraging | Axe or bare hands | Oakren Wood, olives, bushes |
| Mining | Pickaxe | Ores (e.g. Copper Ore) |
| Excavating | Shovel | Chests, fossils |

Rarity weights: Common 60%, Rare 30%, Epic 9.9%, Legendary 0.1%. Nodes restock on a 120s interval in **enabled** site folders only. Collect is server `TryCollect` (node + range + **equipped** tool). Collect time and required tool come from item config.

The collect prompt’s object line shows the tool when one is required (`Requires Pickaxe`, `Requires Axe`, `Requires Shovel`). Bare-hands nodes leave that line empty. If the equipped tool cannot harvest:

- The hold cancels immediately (no harvest wait or harvest sound).
- The prompt flashes locally (`Need a Pickaxe` / `Equip an Axe` / `Repair your Shovel` / `Need a stronger Pickaxe`).
- The node gets a local red highlight and a short tool-icon billboard.
- Only `HARVEST_DENIED` plays. There is no HUD toast for a missing tool.

`TOOL_BROKEN` is still a HUD toast (repair is not on the node). Benji / blacksmith / feedback prompts are unchanged.

---

## Tools

Five named tiers exist in config (Novice → Legendary), including unused `Durability` / `SpeedFactor`. Unused types (Sickle, Knife, Hammer) are also in `TOOL_TYPES`. Only **tier 1 Willow** tools are defined and stored:

| Id | Name | Price |
|---|---|---|
| `PICKAXE_1` | Willow Pickaxe | 100 |
| `AXE_1` | Willow Axe | 100 |
| `SHOVEL_1` | Willow Shovel | 10 |

3D models live in `ServerStorage.Tools`. Gear is a data map (tool id, equipped, Damage 0–100). Equip is UI-only — nothing clones a Tool into the character yet. A broken tool unequips and fires the `TOOL_BROKEN` HUD toast. Wrong-tool collect feedback is on the prompt and the node.

---

## Sites (enabled)

Config enable list is `DestinationConfig.KEYS`. Depart sorts by `DisplayOrder`:

| Order | Key | Name | UI label | Difficulty | Folder | Unlock |
|---|---|---|---|---|---|---|
| 1 | `DUNGEON11` | Coconana Oasis | Level 1 | Easy | `Workspace.Destinations.DUNGEON11` | Always |
| 2 | `DUNGEON11L2` | Coconana Oasis Lvl.2 | Level 2 | Easy | `Workspace.Destinations.DUNGEON11L2` | Visit every Coconana L1 room |
| 3 | `DUNGEON5x5` | Buzzing Plains | Level 1 | Easy | `Workspace.Destinations.DUNGEON5x5` | Always |
| 4 | `DUNGEON5x5L2` | Buzzing Plains Lvl.2 | Level 2 | Easy | `Workspace.Destinations.DUNGEON5x5L2` | Visit every Plains L1 room |
| 5 | `DUNGEON10` | Blackthorn Mountain | Level 10 | Moderate | `Workspace.Destinations.DUNGEON10` | Always |

`HOME` is the hub, not a maze. Coconana L2 walk-in is `CoconanaOasisEntranceL2`. Buzzing Plains L2 walk-in is `BuzzingPlainsEntranceL2` (Savannah art). Each L2 stays locked until every room of that site’s L1 has been visited.

Folder id and “Level N” label still do not line up for Blackthorn (`DUNGEON10` / “Level 10”). See [backlog](backlog.md).

Live dungeon generation is disabled; mazes are pre-placed rooms. Studio bake (`DungeonMaterializerv3`) is Command Bar only: `CoconanaOasisTemplate` is a 15-layout set, `BuzzingSavannahTemplate` is a single all-four-doors room (unused doorways hidden at bake). Bake sources are `ServerStorage.SiteModelTemplates`.

The Destinations button in Depart is **Studio-only** (test skip). Testers/live enter a site by walking the hub `DungeonEntryDoorway` volume. Coconana L1/L2, Buzzing Plains L1/L2, and Blackthorn walk-ins are on. Buzzing Plains L1 lands on `Room_5_3` south. Plains L2 is 16 rooms (4×4 `BuzzingSavannahTemplate`), hub `ToRoom` `4_2` S on `BuzzingPlainsEntranceL2`. Coconana L2 is 16 rooms (4×4 bake).

What’s New is ConfigService `Announcements` on **each place**. If that key is missing, the game skips the board instead of erroring.

---

## Player data (saved)

ProfileStore session per user. Template:

- `Coins` (start 50)
- `FtueStage`
- `Inventory` (item id → count)
- `KnownItems` / `RecentItems` (codex)
- `ExploredRooms` (per-site room ids visited; unlocks that site’s L2)
- `Announcements`
- `RedeemCodes` (empty table; flag off, no UI)
- `MusicMuted` (default false; settings toggle)
- `Gear`
- `Location` (`HOME` or a dungeon key)
- `LastSession`

---

## Surfaces (UI)

| Gui | Role |
|---|---|
| `MainGui` | Hub controls (bag, gear, store, blacksmith, codex, settings) |
| `BagGui` | Inventory |
| `GearGui` | Equipped tools |
| `StoreGui` | Tool shop |
| `BlacksmithGui` | Repair |
| `CodexGui` | Discovery book, rarity filters |
| `Depart` | Site picker + Go Home |
| `CoinIndicator` | Coin HUD |
| `AnnouncementsGui` | What’s New |
| `LoadingScreen` | Maze load overlay (HOME ↔ site; trip name on the GUI) |
| `Testing` | Leftover test UI (Enabled=false). Confirm before deleting. |
| `ScreenOverlayNEW` | Unwired in-dungeon HUD (Enabled=false). Confirm before deleting. |

HUD ScreenGuis use `ResetOnSpawn = false`. **Never remove `Workspace.Avo's Workspace`.** Confirm before deleting any model or GUI.

---

## Hub world

`Workspace.Map` is the town:

- **Benji** + caravan (buyer / sell-all)
- **Traveling Blacksmith** + anvil
- NPCs including Orashi, Karuzen, FeedbackLevel1
- Survey-trip entrance models: Coconana Oasis, Coconana Oasis L2, Buzzing Plains, Buzzing Plains L2, Blackthorn Mountain, plus unused Willoria art and a second Blackthorn model
- Water, lights, barriers, foliage

`Workspace.FTUE` holds tutorial props (`Forage1`, `JourneyEntry`, `FeedbackSpawn`).

---

## Out of scope / not shipped in this instance

- Procedural dungeon generation at runtime (scripts exist, all Disabled)
- Player levels and XP
- Badge awards
- Redeem codes (template field + APIs; flag off; **no UI**)
- Additional tool tiers beyond Willow
- 3D tool-in-hand (equip is gear UI only)
- In-dungeon HUD (`ScreenOverlayNEW` present, Enabled=false, unwired — confirm before deleting)
- Enabled sites other than Coconana L1/L2, Buzzing Plains L1/L2, and Blackthorn (`DUNGEON10`)
