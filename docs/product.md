# Gachamon Legends — Product

Last updated: 2026-09-01  
Place under review: **Gachamon Legends (Development)** (`136894937108297`)

This document describes what the game is, who it is for, and what a session looks like. Implementation lives in Roblox Studio, not in this git repo.

---

## What it is

Gachamon Legends is a gathering / expedition game on Roblox. Players live in a hub town, collect materials with tools, sell them for coins, then travel into pre-built dungeon “sites” (mazes) to forage, mine, and excavate more.

The fantasy is field work: botanica, ores, fossils, and a traveling merchant (Benji) plus a blacksmith. A codex tracks what you have discovered. Tools wear out and can be bought, equipped, and repaired.

It is not a combat RPG. Damage exists as environmental hazards in dungeons (thorns, fire, lava, and similar tags). There is no live player-level / XP loop in the current data template (that code is commented out).

---

## Places

| Place | Place ID | Profile store |
|---|---|---|
| Development (this instance) | `136894937108297` | `GLPlayerProfileDevelopment` |
| Testers | `72816619326760` | `GLPlayerProfileTesters` |
| Live | `115297023432140` | `GLPlayerProfileProd` |

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
| Mining | Pickaxe | Ores (e.g. Copperpine) |
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

Five tiers exist in config (Novice → Legendary) with durability and speed. Only **tier 1 Willow** tools are actually defined and stored:

| Id | Name | Price |
|---|---|---|
| `PICKAXE_1` | Willow Pickaxe | 100 |
| `AXE_1` | Willow Axe | 100 |
| `SHOVEL_1` | Willow Shovel | 10 |

Tools live in `ServerStorage.Tools` and are cloned onto the player. Damage is 0–100 (100 = broken). A broken tool unequips and fires the `TOOL_BROKEN` HUD toast. Wrong-tool collect feedback is on the prompt and the node, not `TOOL_REQUIRED`.

---

## Sites (enabled)

Config enable list is `DestinationConfig.KEYS`. Journeys shown in Depart:

| Key | Name | UI label | Difficulty | Folder |
|---|---|---|---|---|
| `DUNGEON11` | Coconana Oasis | Level 1 | Easy | `Workspace.Destinations.DUNGEON11` |
| `DUNGEON5x5` | Buzzing Plains | Level 3 | Easy | `Workspace.Destinations.DUNGEON5x5` |
| `DUNGEON10` | Blackthorn Mountain | Level 10 | Moderate | `Workspace.Destinations.DUNGEON10` |

`HOME` is the hub, not a maze.

Folder id, marketing name, and “Level N” label do **not** line up (site 11 is “Level 1”; site `DUNGEON5x5` is “Level 3”). See [backlog](backlog.md).

Live dungeon generation is disabled; mazes are pre-placed rooms. Studio bake (`DungeonMaterializerv3`) is Command Bar only: `CoconanaOasisTemplate` is a 15-layout set, `BuzzingSavannahTemplate` is a single all-four-doors room (unused doorways hidden at bake).

The Destinations button in Depart is **Studio-only** (test skip). Testers/live enter a site by walking the hub `DungeonEntryDoorway` volume. Survey-trip models are art around those volumes. Coconana, Buzzing Plains, and Blackthorn walk-ins are on. Buzzing Plains hub door lands on `Room_5_3` south (`IsEntry`). Coconana `Room_4_3` north lands on `Room_3_3` south.

What’s New is ConfigService `Announcements` on **each place**. If that key is missing, the game skips the board instead of erroring.

---

## Player data (saved)

ProfileStore session per user. Template:

- `Coins` (start 50)
- `FtueStage`
- `Inventory` (item id → count)
- `KnownItems` / `RecentItems` (codex)
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
| `ScreenOverlayNEW` | In-dungeon HUD (health, compass, alerts, quick slots) — present, not fully wired in this review |

Leftover test GUI: `Testing` (Enabled=false). `ScreenOverlayNEW` is also Enabled=false.

---

## Hub world

`Workspace.Map` is the town:

- **Benji** + caravan (buyer / sell-all)
- **Traveling Blacksmith** + anvil
- NPCs including Orashi, Karuzen, FeedbackLevel1
- Survey-trip entrance models (Coconana Oasis, Buzzing Plains, Blackthorn Mountain, plus unused Chimstone / Willoria / Savannah)
- Water, lights, barriers, foliage

`Workspace.FTUE` holds tutorial props (`Forage1`, `JourneyEntry`, `FeedbackSpawn`).

---

## Out of scope / not shipped in this instance

- Procedural dungeon generation at runtime (scripts exist, all Disabled)
- Player levels and XP
- Badge awards (referenced, commented)
- Redeem codes (template field + APIs; flag off; **no UI**)
- Additional tool tiers beyond Willow
- Enabled sites other than Coconana (`DUNGEON11`), Buzzing Plains (`DUNGEON5x5`), and Blackthorn (`DUNGEON10`)
