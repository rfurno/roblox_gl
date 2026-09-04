# Gachamon Legends — Studio DevLog (archived)

Copy of `ServerScriptService.Draft.DevLog` (that script is still in the Development place). Historical notes; [backlog](backlog.md) and [code-review](code-review.md) are the living docs.

---

2026-08-30

- Blacksmith RepairTool always returns remaining Damage (number); UI nil-safe. Playtested OK.
- ToolModule: GetToolInfo aliases GetToolById; harvest checks silent; GetTools no longer copies unused Durability.
- In-maze IsExit → hub spawn with join facing (SpawnLocation look). In-maze IsEntry → just outside that site's hub DungeonEntryDoorway (town side). Go Home still spawn. Playtested OK.
- DungeonMaterializerv3: Command Bar locals only (no script attributes). templateSet can be a 15-layout prefix (CoconanaOasisTemplate) or a single all-four-doors model (BuzzingSavannahTemplate). Unused doorways are hidden. Keep Disabled.

2026-08-29

- ToolModule require restored on ToolPurchaseHandler + PlayerInventoryManager (shop + tool harvest). MaterialReplenishModule / PlayerGearManager still omit unused require.
- Join always SetLocation HOME. TeleportHandler + SiteTeleportController require KEYS. GetStoreName falls back to development store. AddCoins rejects non-positive. CanToolHarvestItem nil-safe unknown toolId.
- SSS grouped: Teleport, Collect, Economy, World. ToolPurchaseHandler + ToolRepairHandler folded into PlayerGearManager remotes.
- Trimmed unused ToolModule durability APIs; RestorePlayerData; getToolIdFromPlayerToolId. REDEEM_CODES_ENABLED = false.
- FTUE forage/sell/journey wait on inventory/location signals (no print poll).

2026-08-28

- MusicMuted on profile; settings load/save; RandomMusicPlayer uses saved mute on play
- WOOD_1 AvailabilityEndDate 2025-11-30 → 2027-11-30
- DUNGEON1 removed; Buzzing Plains is DUNGEON14 in KEYS + DESTINATIONS (FolderName DUNGEON14, Level 3)
- Hub DUNGEON14.DungeonEntryDoorway: DungeonId DUNGEON14, ToRoom 4_2, ToDoorDirection S; landing Room_4_2 south IsEntry; TeleportTrigger tagged
- Profile Location DUNGEON1 → DUNGEON14 on load
- DUNGEON11 hub DungeonEntryDoorway was at (-743, 31, 1405), not on CoconanaOasisEntrance; snapped to the Coconana gate (same local offset as DUNGEON14 on Buzzing Plains)
- DUNGEON14 / Buzzing Plains hub walk-in playtested OK
- Buzzing Plains live site DUNGEON14 → DUNGEON5x5. Hub TeleportTrigger snapped to the old DUNGEON14 gate CFrame (BuzzingPlainsEntrance). Landing Room_5_3 south IsEntry. KEYS + dest block + Location DUNGEON1/DUNGEON14 → DUNGEON5x5.
- DUNGEON5x5 / Buzzing Plains hub walk-in playtested OK
- DungeonMaterializerv3: Studio-only bake (copy at DungeonMaterializerv3ORIGINAL). Attributes DungeonId/GridM/GridN/TemplateSet/SnapToSurveyEntrance; CoconanaOasisTemplate only; hub DungeonEntryDoorway; tag assert; post-bake door check. Keep Disabled.

2026-08-27

Dev place. Ties to docs/backlog.md + docs/code-review.md.

- SellAll: price * stack count; deleted Sale remote (P0 sell)
- CharacterAdded: no 8s re-Initialize / FTUE restack; GetPlayerData handshake; coin HUD pulls on spawn (P0 respawn / FTUE / shop snapshot)
- DeductCoins refuses insufficient funds; GetToolPrice / GetToolRepairCost / GetToolInfo nil-safe; HasTool by catalog id (P0 shop)
- One LoadingScreen for HOME ↔ site; HOME→entry unstick LookVector*3+(0,3,0) (P1 teleport)
- DUNGEON1.DungeonEntryDoorway DungeonId DUNGEON11 → DUNGEON1 (hub walk-in was sending Buzzing Plains gate to Coconana)
- Template.RedeemCodes = {}; Get/AddRedeemCode init the table if missing (P0 redeem)
- HUD LocalScripts (Depart, bag, coins, gear, store, blacksmith, codex, settings, notifications, sound, music) moved to StarterPlayerScripts; ScreenGuis ResetOnSpawn = false
- Collect TryCollect: tagged part, live MaterialItemId, server range, tool; KEYS-only replenish (no DUNGEON3/7/8 stock)

2026-03-22

- Feedback NPC near spawnbase

2026-03-20

- In-game feedback
- Only level 1 is enabled

2026-03-14

- New background music
- Buttons resizing and repositioning to avoid overlapping
- Announcements NEWS sound
