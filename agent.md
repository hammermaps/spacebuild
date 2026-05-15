# Agent Guide – Spacebuild

This document provides AI coding agents with a concise map of the repository so they can navigate and contribute effectively.

---

## Project Overview

**Spacebuild** is a [Garry's Mod](http://garrysmod.com/) gamemode / addon that simulates space environments, resource distribution, life-support systems, and space travel.  
The current stable release is **Spacebuild 3** (branch `master` / `sb3`).  
**Spacebuild 4** (`sb4`) is in active development.

Related repository: [Spacebuild Expansion Pack (SBEP)](https://github.com/spacebuild/sbep)

---

## Repository Layout

```
spacebuild/
├── addon.json              # GMod addon metadata (title, tags, ignore list)
├── gamemodes/spacebuild/   # Gamemode definition (init, cl_init, shared)
├── lua/
│   ├── autorun/
│   │   ├── client/         # Client-side autorun scripts
│   │   └── server/         # Server-side autorun scripts (CAF bootstrap)
│   ├── caf/                # Core Addon Framework
│   │   ├── core/
│   │   │   ├── client/     # CAF client core
│   │   │   ├── server/     # CAF server core (module_loader, hooks)
│   │   │   └── shared/     # Shared CAF utilities
│   │   ├── addons/
│   │   │   └── server/     # Resource distribution (RD), life support, etc.
│   │   ├── languagevars/   # Localisation strings
│   │   └── stools/         # Spacebuild tool definitions
│   ├── entities/           # Custom GMod entities (storage, pumps, RD base, …)
│   ├── effects/            # Particle / visual effects
│   ├── includes/modules/   # Shared Lua modules (arraylist, hashmap, cache, …)
│   ├── vgui/               # Custom VGUI panels
│   └── weapons/            # Spacebuild weapons
├── maps/                   # Example / bundled maps
├── materials/              # Textures and VMTs
├── models/                 # MDL models
├── particles/              # Particle system files
├── sound/                  # Sound assets
└── docs/                   # Developer documentation
    └── OPTIMIZATION_PLAN.md
```

---

## Architecture: Core Addon Framework (CAF)

CAF is the plugin/addon layer that sits between GMod and all Spacebuild subsystems.

| Component | Path | Role |
|-----------|------|------|
| Bootstrap | `lua/autorun/server/sv_caf_autostart.lua` | Loads CAF core and all registered addons on server start |
| Module loader | `lua/caf/core/server/module_loader.lua` | Discovers and initialises addon modules |
| Shared utilities | `lua/caf/core/shared/sh_general_caf.lua` | `CAF.GetAddon()`, hooks, global helpers |
| Resource Distribution (RD) | `lua/caf/addons/server/resourcedistribution.lua` | Network-based resource routing between entities |
| Data structures | `lua/includes/modules/` | `arraylist`, `hashmap`, `cache` used throughout CAF |

### Key globals
- `CAF` – framework namespace (addon registry, helpers)
- `RD` – resource distribution API

---

## Coding Conventions

- **Language**: Lua 5.1 (GMod's LuaJIT environment)
- **Style**: 4-space indentation, PascalCase for module tables, camelCase for local variables
- **Linter**: [luacheck](https://github.com/mpeterv/luacheck) – config in `.luacheckrc`
- **No global pollution**: use `local` for all helpers; register everything through `CAF` or `hook.Add`

### Performance rules (see `docs/OPTIMIZATION_PLAN.md` for full details)
1. Use `next(t) == nil` instead of `table.Count(t) == 0` for existence checks.
2. Cache `RD.GetNetResourceAmount()` results inside loops.
3. Never modify a table while iterating it with `pairs`; collect keys first.
4. Gate expensive `Think` work with a `CurTime()` throttle.
5. Use `ipairs` for sequential-array tables; `pairs` for hash tables only.

---

## Lint, Build & Test

```bash
# Lint (requires luacheck installed)
luacheck lua/

# There is no separate build step – Lua files are loaded at runtime by GMod.
# Tests are validated through the Travis CI configuration (.travis.yml).
```

The CI pipeline is defined in `.travis.yml` and runs luacheck on every push.

---

## Branches

| Branch | Purpose |
|--------|---------|
| `master` | Stable Spacebuild 3 release (synced with Steam Workshop) |
| `sb2` | Spacebuild 2 – no longer supported |
| `sb2.5` | Spacebuild 2.5 – no longer supported |
| `sb3` | Development branch for Spacebuild 3 |
| `sb4` | Spacebuild 4 – work in progress |

---

## Useful Links

- Steam Workshop: <https://steamcommunity.com/sharedfiles/filedetails/?id=693838486>
- Facepunch thread: <https://facepunch.com/showthread.php?t=1519499>
- Discord: <https://discord.gg/3A4dPhD>
- SBEP (Expansion Pack): <https://github.com/spacebuild/sbep>
- GLua API Reference: <https://samuelmaddock.github.io/glua-docs/>
