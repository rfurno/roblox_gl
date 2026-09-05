# Ads campaign watch — 2026-09-04

Live/Alpha Open Cloud watch for the ads campaign. **No game, Studio, or script updates in this pass.** This file is the issues log and the fix plan.

| | |
|---|---|
| Place | Live/Alpha `115297023432140` (Gachamon Legends [Alpha]) |
| Universe | `8330572807` |
| Window | **2026-09-04 12:00–16:00 America/New_York (EDT, UTC−4)** |
| Cadence | Hourly snapshots at 12:00, 13:00, 14:00, 15:00, 16:00 EDT |
| Auth | `$ROBLOX_API_KEY` with `universe:read` (do not print the key) |

Timezone is the machine’s `America/New_York` DST (EDT). If the ads buy is actually Pacific, 12:00 PDT = 15:00 EDT — say so and the window should shift.

How to pull logs: [architecture — Live observability](architecture.md#live-observability-open-cloud). Studio MCP cannot see published Output.

---

## Watch protocol (each hour)

Do not publish, copy scripts, or edit Studio.

1. Confirm `$ROBLOX_API_KEY` is set. Do not print the value.
2. Public snapshot: `GET https://games.roblox.com/v1/games?universeIds=8330572807` and `GET https://games.roblox.com/v1/games/115297023432140/servers/Public`.
3. Current versions: `GET https://apis.roblox.com/server-management/v1/universes/8330572807/places/115297023432140/game-servers:filter-options` (place-version-history is 403 on this key).
4. List servers for the **latest** `PlaceVersion` (baseline latest is **2308**, not the 2305 in older docs) and any other version that still has jobs.
5. Fetch logs for every **active** job. If none are active, fetch the newest shutdown/crashed job so the hour still has Output.
6. Append the matching hourly section below. Add new rows to [Issues and fix plan](#issues-and-fix-plan) only. Do not implement.

Stop after the 16:00 snapshot. Cancel the hourly scheduler.

---

## Baseline (2026-09-03 22:14 EDT)

Taken ~14h before the campaign window. Open Cloud access works.

| Field | Value |
|---|---|
| Playing | 0 |
| Visits | 59 |
| Favorites | 3 |
| Public servers | none |
| Latest place version | **2308** (place `updateTime` 2026-09-04 01:07 UTC) |
| Engine | `0.737.0.7371583` (also saw `0.736.0.7361346` on older jobs) |
| `serverSize` | 10 |

### Version 2308

One job, already down:

| jobId | Created (UTC) | Uptime | Status | Occ / max |
|---|---|---|---|---|
| `c1513afd-4547-4862-9e84-1c8d1b25918e` | 2026-09-04 01:06:30 | 25s | shut_down | 0 / 10 |

Logs (retry after a one-off **504**; second call 200):

```
[2] 2026-09-04T01:06:32.27Z  ConfigService: Config value not found for key "Announcements".
[2] 2026-09-04T01:06:32.27Z  ConfigService Announcements is missing; What's New will stay empty until the key exists
```

### Version 2305 (15 shutdown jobs)

Newest public-style jobs (type 3, max 10) plus one Studio/reserved type-4 (max 60). Four jobs on **2026-08-31** are `crashed`. No active jobs.

| jobId prefix | Created (UTC) | Uptime | Status | Notes |
|---|---|---|---|---|
| `09974e54` | 09-04 00:15 | 8m 58s | shut_down | type 4, max 60, **0 log lines** |
| `26cc1141` | 09-03 17:41 | 13m 51s | shut_down | same two Announcements warns |
| `964df545` | 09-03 16:03 | 1m 13s | shut_down | same two Announcements warns |
| `17b91fb4` | 08-31 17:40 | 31m | **crashed** | Output still only the two warns |
| `d28860d3` | 08-31 16:55 | 9m | **crashed** | |
| `e27393b4` | 08-31 13:51 | 21m | **crashed** | |
| `4ab847be` | 08-31 13:17 | 27m | **crashed** | |

Older 2305 public jobs (Sep 1–2) ran 20–40 minutes then shut down normally. Filter-options also lists 2296–2300; those jobs are all shutdown/crashed and predate this publish.

Open Cloud Output on every job that had logs was **only** the two ConfigService warnings. Either the game is quiet at print/info, or this API is returning warn+ only.

---

## Issues and fix plan

Collected before the campaign. Hourly checks should add rows, not implement.

| ID | Sev | Status | Symptom | Likely cause | Fix plan (do not do now) |
|---|---|---|---|---|---|
| A1 | P3 | Not a player bug. Key missing on Live; **no announcement content exists**, so the board staying hidden is correct. Warns still fire on every 2314 boot (including 22:13). | Server Output: `ConfigService: Config value not found for key "Announcements"` then the module warn. Players do not see a broken What’s New — they see nothing, which matches “there is none.” | ConfigService key absent on Live. `AnnouncementModule` uses `{}` and skips. | Optional hygiene only: add an **empty** `Announcements` table on Live so boots stop warning. Do **not** treat this as a player-facing fix or as a prerequisite for ads work. |
| A2 | P2 | Done in docs 2026-09-04 | Docs still said Live last published as **2305**. Live is **2314**. | Docs lagged the 12:14 EDT publish. | product / architecture / backlog now say 2314. Keep using `filter-options`. |
| A3 | P2 | Open | `GET .../place-version-history-api/v1/115297023432140/history` → **403** `Scope not authorized`. | API key has `universe:read` (servers + logs) but not the version-history scope. | Either add the missing credential scope in Creator Dashboard, or drop step 3 of the agent workflow and use `game-servers:filter-options` + `cloud/v2/.../places/{id}` `updateTime`. |
| A4 | P2 | Open | `POST analytics-query-api/.../metrics` → **400** (`request` required; `granularity` not a `MetricGranularity`). Auth did not 403. | Request body does not match the public schema (`metric` / `granularity` enum / wrapping `request`). | Look up the analytics-query-api schema, send a valid body, then FTUE/CCU can be pulled during a later watch. Not blocking Output tails. |
| A5 | P1 | Watch — **no new crash in the ads window** | Four **crashed** jobs on 2305, all 2026-08-31 (~9–31 min uptime). Open Cloud logs have no stack — only A1 warns. | Unknown historically. 2314 public jobs in-window (82 min and 93 min) both `shut_down` cleanly. | Historical Aug 31: check Creator Hub crash reports if any. Do not republish on speculation. |
| A6 | P3 | Noted | One **504 Gateway Time-out** on 2308 logs; retry 200 with the same two lines. | Open Cloud flakiness. | Retry logs with a 60s curl timeout. A 504 is not a game bug unless it persists for the hour. |
| A7 | P2 | Watch — peak occ **3/10** at 20:50 (was 1/10 in the 12–16 window). | Live `serverSize` is **10**. | Game Settings max players. | Still headroom. Revisit only if occupancy sticks at 10 with a queue. |
| A8 | P3 | Watch, confirmed evening | Open Cloud Output is warn+ only. Evening 2314 jobs with logs still only A1. Three short jobs had **n=0** (`86157d1f` 14s, `178865c8` 19s, `1a1bc078` 5m fps=0). | API filters info/`print`, or those jobs died before the first warn flushed. | Do not spam `print`. After A1, add a few `warn`s on FTUE stage enter/exit, first collect, first sell, teleport fail — so the next log pull can explain A9 bounces. |
| A9 | P1 | **Done on Live 2315** (playtested in Live Studio 2026-09-05) | After 16:00, **21 new public 2314 jobs** (19 by 20:50 + 2 more). Many last **14s–2 min**. Latest: `0a1f1030` 7m, `eea906ff` **1m 02s**, both A1 only. Visits 66→**123**, favorites 3→**8**. Peak occ 3/10. No crash. | FTUE mechanics work; they never explain **why** collect / sell / journey. Ads players can leave before the loop is obvious. Not a What’s New bug. | Banner + why-copy on Live 2315 ([product](product.md#core-loop), [architecture](architecture.md#ftue)). Client scripts only. Existing Complete profiles stay hidden until `SetFtueStage(Forage)` on prod. Optional later: A8 `warn`s and A4 funnel. Do not raise max players. |
| A10 | P3 | Still one-off at 22:13 | Job `1a1bc078` (19:24 EDT, 5m 02s) shut down with **fps=0**, mem ~786MB (others ~850–890MB). Not `crashed`. n=0 logs. New 22:13 jobs ran at fps 60. | Possible hitch or a job that never rendered; still a clean `shut_down`. | No repeat on the two newest jobs. Keep watching; ignore unless fps=0 returns with a stack. |

Already handled in code, not a live crash: missing `Announcements` no longer iterates nil (`AnnouncementModule` uses `{}`). The remaining gap is empty What’s New, not a boot exception.

Art rule still applies to any later fix work: never remove `Workspace.Avo's Workspace`; confirm before deleting models/GUIs.

---

## Hourly snapshots

Fill these in during the window. Copy the table shape. Note **new** log lines and **new** crashed/active jobs only.

### 12:00 EDT — 2026-09-04 12:00:25 EDT

Public CCU **0**. Visits **62** (baseline 59, **+3**). Favorites 3. Public servers **none**. Latest place version still **2308**. Place `updateTime` moved to **2026-09-04 15:53:07 UTC** (11:53 EDT, ~7 min before this snapshot) — description unchanged (`v0.2.0 ALPHA`); no 2309+ jobs exist. `serverSize` 10. Engine `0.737.0.7371583`.

| Field | Value |
|---|---|
| Playing | 0 |
| Visits | 62 |
| Public servers | 0 |
| Active jobs | 1 (type 4 / reserved-or-Studio, not public) |
| New crashes | none |
| New Output vs A1 | none — same two Announcements warns on every public job that had logs |

**2308 jobs** (5 total):

| jobId | Created EDT | Uptime | Status | Occ / max | Type | Logs |
|---|---|---|---|---|---|---|
| `1bc1d075-dc6d-4cfe-834c-44db10b07d97` | 11:28 | 31m 55s (still up) | **active** | **1 / 60** | 4 | empty (`n=0`) |
| `913d629d-9569-4a19-a391-700f3d508257` | 10:31 | 8m 59s | shut_down | 0 / 10 | 3 | same A1 warns |
| `21798195-5ee9-41ad-8e57-a193ebf642b9` | 09:50 | 40m 19s | shut_down | 0 / 10 | 3 | same A1 warns |
| `37c55619-28e5-46f8-b971-c24d3d95a821` | 09:27 | 2m 56s | shut_down | 0 / 10 | 3 | same A1 warns |
| `c1513afd-…` | 21:06 previous night | 25s | shut_down | 0 / 10 | 3 | baseline A1 |

2305: still 15 shutdown jobs, same four Aug 31 crashes. No new 2305 activity.

Ads window just opened; morning traffic looks like 3 public joins before noon (matches +3 visits). No public players at 12:00. The occupancy-1 job is type 4 (Studio/reserved), so it is not the ads CCU.

**Issues this hour:** A1 still open. No new IDs.

### 13:00 EDT — 2026-09-04 13:00:29 EDT

Public CCU **1**. Visits **63** (+1 since 12:00). Favorites 3. Public servers **1**. Latest place version **2314** (new since 12:00). Place `updateTime` **2026-09-04 16:14:51 UTC** (12:14 EDT). `serverSize` 10. Engine `0.737.0.7371583`. Public server fps 60, ping 100.

| Field | Value |
|---|---|
| Playing | 1 |
| Visits | 63 |
| Public servers | 1 |
| Active jobs | 1 public (type 3, occ 1/10) on **2314** |
| New crashes | none |
| New Output vs A1 | none — A1 still on the live 2314 boot |

**New / changed jobs this hour:**

| jobId | Ver | Created EDT | Uptime | Status | Occ / max | Type | Logs |
|---|---|---|---|---|---|---|---|
| `33f49144-0d86-4600-96af-1b1846e81217` | **2314** | 12:48 | 12m 44s (still up) | **active** | **1 / 10** | 3 | same A1 warns (retry after 504) |
| `0b93de0d-2299-4f28-9ce7-f3a9ade24a6b` | 2313 | 12:13 | 1m 13s | shut_down | 0 / 60 | 4 | empty (`n=0`) |
| `61cac9d5-9287-41c0-9bed-c089566825cc` | 2308 | 12:04 | 17s | shut_down | 0 / 10 | 3 | empty (`n=0`) |
| `1bc1d075-…` | 2308 | 11:28 | 38m 57s | shut_down (was active at 12:00) | 0 / 60 | 4 | (unchanged) |

2305: same 15 historical jobs; no new 2305 activity. No new `crashed` jobs.

Live was republished during the window (2313 then **2314** at 12:14 EDT). First ads-window public player is on 2314. A1 survived the publish — ConfigService `Announcements` is still missing.

**Issues this hour:** A1 still open on 2314. A2 now tracks 2314. No new IDs. Peak playing this hour: **1**.

#### Update 13:02 EDT

Same CCU **1**, visits **63**, version **2314**. Active job `33f49144-…` still up (uptime 14m 37s, occ 1/10, fps 60, ping 50). Same A1 warns; no new jobs, crashes, or Output lines.

### 14:00 EDT — 2026-09-04 14:00:35 EDT

Public CCU **1**. Visits **64** (+1 since 13:00). Favorites 3. Public servers **1**. Latest place version still **2314**. Place `updateTime` unchanged (12:14 EDT). `serverSize` 10. Public server fps 60, ping 19.

| Field | Value |
|---|---|
| Playing | 1 |
| Visits | 64 |
| Public servers | 1 |
| Active jobs | 1 public (type 3, occ 1/10) on **2314** |
| New crashes | none |
| New Output vs A1 | none — same A1 warns on the same live job |

**Jobs this hour:**

| jobId | Ver | Created EDT | Uptime | Status | Occ / max | Type | Logs |
|---|---|---|---|---|---|---|---|
| `33f49144-0d86-4600-96af-1b1846e81217` | 2314 | 12:48 | **1h 12m 49s** (still up) | **active** | **1 / 10** | 3 | same A1 warns |

No new 2314/2313/2308 jobs. Same player session has been up since 12:48 EDT (~73 min). Visits ticked 63→64 with occupancy still 1 (join/leave or a second visit on the same server).

**Issues this hour:** A1 still open. No new IDs. Peak playing this hour: **1**. Server not full (A7).

#### Update 14:02 EDT

Same CCU **1**, visits **64**, version **2314**. Active job `33f49144-…` still up (uptime 1h 14m 15s, occ 1/10, fps 60, ping 18). Same A1 warns; no new jobs, crashes, or Output lines.

### 15:00 EDT — 2026-09-04 15:00:38 EDT

Public CCU **1**. Visits **65** (+1 since 14:00). Favorites 3. Public servers **1**. Latest place version still **2314**. Place `updateTime` unchanged (12:14 EDT). `serverSize` 10. Public server fps 60, ping 23.

| Field | Value |
|---|---|
| Playing | 1 |
| Visits | 65 |
| Public servers | 1 |
| Active jobs | 1 public (type 3, occ 1/10) on **2314** — **new job** |
| New crashes | none |
| New Output vs A1 | none — same A1 warns on the new job boot |

**Jobs this hour:**

| jobId | Ver | Created EDT | Uptime | Status | Occ / max | Type | Logs |
|---|---|---|---|---|---|---|---|
| `353f9221-916c-47e1-b0ef-6a20c22de357` | 2314 | 14:17 | 43m 42s (still up) | **active** | **1 / 10** | 3 | same A1 warns |
| `33f49144-0d86-4600-96af-1b1846e81217` | 2314 | 12:48 | 1h 22m 09s | shut_down at 14:10 EDT | 0 / 10 | 3 | (A1 from 13:00; no extra lines) |

The ~82 min session ended cleanly (not crashed). A new public job started ~7 min later and is still occupied. A1 still fires on 2314 boot.

**Issues this hour:** A1 still open. No new IDs. Peak playing this hour: **1**.

#### Update 15:02 EDT

Same CCU **1**, visits **65**, version **2314**. Active job `353f9221-…` still up (uptime 45m 06s, occ 1/10, fps 60, ping 20). Same A1 warns; no new jobs, crashes, or Output lines.

### 16:00 EDT — 2026-09-04 16:00:45 EDT (final)

Public CCU **0**. Visits **66** (+1 since 15:00; **+4** in the 12:00–16:00 window from 62). Favorites 3. Public servers **none**. Latest place version still **2314**. Place `updateTime` unchanged (12:14 EDT). `serverSize` 10.

| Field | Value |
|---|---|
| Playing | 0 |
| Visits | 66 |
| Public servers | 0 |
| Active jobs | none |
| New crashes | none |
| New Output vs A1 | none — last job still only A1 warns |

**Jobs this hour:**

| jobId | Ver | Created EDT | Uptime | Status | Occ / max | Type | Logs |
|---|---|---|---|---|---|---|---|
| `353f9221-916c-47e1-b0ef-6a20c22de357` | 2314 | 14:17 | **1h 33m 23s** | shut_down at **15:50 EDT** | 0 / 10 | 3 | same A1 warns |

That session ended cleanly (~10 min before this snapshot). No live job at 16:00. No new Output lines, no crash.

**Issues this hour:** A1 still open. No new IDs. Peak playing this hour: **1** (before 15:50).

### Window summary (12:00–16:00 EDT)

| | |
|---|---|
| Live version | **2314** (published ~12:14 EDT; 2313 was a 1m type-4 blip) |
| Peak playing | **1** |
| Visits | 62 → **66** (+4) |
| Public 2314 sessions | `33f49144` 12:48–14:10 (82 min, clean shut_down); `353f9221` 14:17–15:50 (93 min, clean shut_down) |
| Crashes in window | **none** |
| Only live Output | A1 ConfigService `Announcements` missing (both 2314 boots) |
| Servers full | never (max occ 1/10) |

Do not implement from this watch. Highest-priority leftover: **A1** add ConfigService `Announcements` on Live/Alpha, then publish.

---

## Post-window pull — 2026-09-04 20:50 EDT

Same Open Cloud path as the hourly watch. Live still **2314** (no newer place version). Engine `0.737.0.7371583`.

| Field | 16:00 EDT | 20:50 EDT |
|---|---|---|
| Playing (games API) | 0 | 1 (stale vs occupancy) |
| Public server occupancy | 0 | **3 / 10** |
| Visits | 66 | **117** (+51) |
| Favorites | 3 | **7** (+4) |
| Public servers | 0 | 1 |
| Active jobs | none | 1 |
| New crashes | none | none |
| Place `updateTime` | 12:14 EDT | unchanged |

**Active**

| jobId | Ver | Created EDT | Uptime | Occ / max | fps / ping | Logs |
|---|---|---|---|---|---|---|
| `f89cef52-b3a1-4da0-a248-57275c449120` | 2314 | 20:29 | 21m | **3 / 10** | 60 / 137 | same A1 warns |

**New 2314 public jobs since 16:00** (19 + the still-up one). All `shut_down` except `f89cef52`. Times EDT:

| jobId prefix | Created | Uptime | Notes |
|---|---|---|---|
| `01e01e28` | 17:08 | 49m 22s | longest evening session; A1 |
| `178865c8` | 18:08 | 19s | n=0 logs |
| `65951a0a` | 19:04 | 45s | A1 |
| `075c72c2` | 19:07 | 36s | A1 |
| `86157d1f` | 19:23 | 14s | n=0 |
| `147c3320` | 19:24 | 21s | A1 |
| `1a1bc078` | 19:24 | 5m 02s | **fps=0**, n=0 (A10) |
| `789bc638` | 19:26 | 42s | A1 |
| `c37b9e11` | 19:34 | 1m 31s | A1 |
| `54ee7f22` | 19:36 | 4m 45s | A1 |
| `1a56d8cc` | 19:43 | 2m 28s | A1 |
| `9e00d750` | 19:55 | 2m 36s | A1 |
| `99b18da1` | 19:59 | 1m 11s | A1 |
| `d0247449` | 20:01 | 10m 15s | A1 |
| `678a19fd` | 20:12 | 2m 12s | A1 |
| `bbef6859` | 20:17 | 26s | A1 |
| `e9b3c6d7` | 20:18 | 4m 47s | A1 |
| `a82cf843` | 20:23 | 28s | A1 |
| `f89cef52` | 20:29 | still up | occ 3, A1 |

No 2315+ jobs. Historical 2305 crashes unchanged.

---

## Post-window pull — 2026-09-04 22:13 EDT

Same Open Cloud path. Live still **2314**. No active jobs.

| Field | 20:50 EDT | 22:13 EDT |
|---|---|---|
| Playing (games API) | 1 | **0** |
| Public occupancy | 3 / 10 | **0** |
| Visits | 117 | **123** (+6) |
| Favorites | 7 | **8** (+1) |
| Public servers | 1 | 0 |
| New crashes | none | none |
| New Output | A1 only | A1 only |

`f89cef52` (the 3-player server) shut down cleanly at **20:59 EDT** after **29m 50s**.

**New 2314 jobs since 20:50**

| jobId prefix | Created EDT | Uptime | Status | Logs |
|---|---|---|---|---|
| `0a1f1030` | 21:59 | 7m 08s | shut_down 22:06 | same A1 warns |
| `eea906ff` | 22:10 | **1m 02s** | shut_down 22:11 | same A1 warns |

Fix order unchanged: A1 first, then A9 playtest / FTUE warns. A10 did not repeat (both new jobs fps 60).

---

## Recommended fix order (do not implement in this pass)

A1 is **not** a player-facing bug (no What’s New content by design). Do not block other work on it.

1. **A9 — FTUE why-copy** (P1). **Done on Live 2315** (banner + copy; playtested in Live Studio). Plan in [product](product.md#core-loop).
2. **A8 — a few `warn`s on FTUE / collect / sell / teleport fail** so the next log pull can explain remaining bounce. Optional: publish the Dev shop-billboard disable (`ShopTitle.SellSamplesBillboard`) to Live if 2314 still has the floating shop icon (Live `updateTime` never moved after 12:14 EDT).
3. **A2 — docs** (done 2026-09-05: product / architecture / backlog say 2315).
4. **A4 — analytics query body** so FTUE funnel can explain A9 (P2).
5. **A1 hygiene** — empty `Announcements` key on Live to silence warns. Not required for players.
6. **A3, A6, A10**; **A7** leave `serverSize` 10.

Art rule still applies to any later Studio work: never remove `Workspace.Avo's Workspace`; confirm before deleting models/GUIs.

---

## End-of-window checklist

- [x] Five snapshots present (12–16 EDT)
- [x] New Output lines (if any) copied into the issues table — none besides A1
- [x] New `crashed` jobs listed with jobId + uptime — none in window
- [x] Peak `playing` / occupancy recorded — peak 1 / 10
- [x] Hourly scheduler deleted
- [x] Still no Studio/publish changes from this watch

