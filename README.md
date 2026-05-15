[![Issue Count](https://codeclimate.com/github/spacebuild/spacebuild/badges/issue_count.svg)](https://codeclimate.com/github/spacebuild/spacebuild)
[![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=master)](https://travis-ci.org/spacebuild/spacebuild)

# Spacebuild

[Garry's Mod][garrysmod] Spacebuild Project – space environments, resource distribution, and life-support systems for GMod.

| | |
|---|---|
| 💬 **Discord** | [Join here][discord] |
| 🧵 **Facepunch** | [Thread][facepunch] |
| 🎮 **Steam Workshop** | [Spacebuild 3][workshop] |
| 🔧 **Expansion Pack** | [SBEP repository](https://github.com/spacebuild/sbep) |

> **Note:** Please do not upload to the Workshop. Use the **official** Workshop version – this repository is kept in sync with it.

---

## Build Status

| Branch | Status |
|--------|--------|
| `master` | [![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=master)](https://travis-ci.org/spacebuild/spacebuild) |
| `sb3` | [![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=sb3)](https://travis-ci.org/spacebuild/spacebuild) |
| `sb4` | [![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=sb4)](https://travis-ci.org/spacebuild/spacebuild) |
| `sb2` | [![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=sb2)](https://travis-ci.org/spacebuild/spacebuild) |
| `sb2.5` | [![Build Status](https://travis-ci.org/spacebuild/spacebuild.svg?branch=sb2.5)](https://travis-ci.org/spacebuild/spacebuild) |

---

## Status

| Version | State |
|---------|-------|
| **Spacebuild 3** (`master`) | ✅ Current stable release |
| **Spacebuild 4** (`sb4`) | 🚧 Work in progress |
| **Spacebuild 2 / 2.5** | ❌ No longer supported |

**Official Workshop version:** <http://steamcommunity.com/sharedfiles/filedetails/?id=693838486>

---

## Workshop Installation

Spacebuild 3 is available via the Steam Workshop. Go to [its Workshop page][workshop] and press **Subscribe** – it will automatically appear in Garry's Mod.

---

## Manual Installation

How to use TortoiseGit to clone/pull:
<http://steamcommunity.com/groups/spacebuild/discussions/0/144513670980243163/>

Clone this repository into your `addons` folder:

```bat
cd "%programfiles(x86)%\Steam\SteamApps\common\GarrysMod\garrysmod\addons"
git clone https://github.com/spacebuild/spacebuild.git spacebuild
```

---

## Contributing

Pull requests are welcome! Please keep your code clean and follow the existing style.

Current contributors: @snakesvx · @generalwrex · @X-Coder · @CaveeJohnson

> **For AI agents and automated tooling:** see [`agent.md`](agent.md) for a full guide to the repository structure, architecture, coding conventions, and performance rules.

---

## Documentation

- [`docs/OPTIMIZATION_PLAN.md`](docs/OPTIMIZATION_PLAN.md) – performance and maintainability roadmap
- [`agent.md`](agent.md) – repository guide for AI coding agents

---

## License

Copyright 2009-2016 SB Dev Team

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

[garrysmod]: <http://garrysmod.com/>
[workshop]: <http://steamcommunity.com/sharedfiles/filedetails/?id=693838486>
[facepunch]: <https://facepunch.com/showthread.php?t=1519499&p=50363396>
[discord]: <https://discord.gg/3A4dPhD>
