# roblox_gl

Documentation for **Gachamon Legends** (Roblox). The game itself lives in Roblox Studio; this repo is the written source of truth for product, architecture, and backlog.

Development place: `136894937108297` (Gachamon Legends — Development). Live/Alpha: `115297023432140`.

## Live logs (Open Cloud)

Studio MCP only sees the open Development place. Production Output is Open Cloud, not Studio.

```bash
export ROBLOX_API_KEY='…'   # from https://create.roblox.com/dashboard/credentials
```

Restart Grok in that same shell so the agent inherits the env. Never commit the key or paste it into chat. Scopes: `universe:read` (servers + logs), `universe.analytics:read` (FTUE/metrics). Endpoints and IDs: [docs/architecture.md](docs/architecture.md#live-observability-open-cloud).

## Studio edit rules

Models and ScreenGuis are art. Treat them that way.

- **Never** remove `Workspace.Avo's Workspace`.
- **Ask and wait for confirmation** before deleting, moving to an archive place, or otherwise removing any **model** or **GUI** (Workspace art, templates, tools, NPCs, `StarterGui`, `StarterPack`, survey-trip entrances, bake clones, test GUIs).
- Script and config edits are separate from that rule.

| Doc | Contents |
|---|---|
| [docs/product.md](docs/product.md) | What the game is, loop, sites, data, UI |
| [docs/architecture.md](docs/architecture.md) | Services, remotes, teleport, collect, FTUE, Studio bake, Open Cloud live logs |
| [docs/backlog.md](docs/backlog.md) | Working gaps (updated 2026-09-05; FTUE why-copy on Live 2315) |
| [docs/code-review.md](docs/code-review.md) | Studio review 2026-09-03; L2 sites on Live (published) |
| [docs/devlog.md](docs/devlog.md) | Copy of Studio `Draft.DevLog` |
| [docs/ads-campaign-2026-09-04.md](docs/ads-campaign-2026-09-04.md) | Ads watch 2026-09-04 12:00–16:00 EDT: live logs, issues, fix plan (no game edits) |
