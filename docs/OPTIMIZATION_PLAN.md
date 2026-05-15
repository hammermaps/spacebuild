# Spacebuild – Optimization Plan

This document outlines concrete steps to improve the performance, stability, and
maintainability of the Spacebuild/CAF codebase.  
It is ordered by impact: items near the top give the largest benefit for the least
effort.

---

## 1  Replace `table.Count()` existence checks with `next()`

**Files affected**  
`lua/includes/modules/arraylist.lua`, `lua/includes/modules/hashmap.lua`,
`lua/caf/addons/server/resourcedistribution.lua`, `lua/autorun/server/sv_caf_autostart.lua`

**Problem**  
`table.Count(t) == 0` (and `> 0`) iterates the entire table to count its entries –
O(n) every call.  `IsEmpty()` and `Size()` are called frequently inside hot think
loops.

**Fix**
```lua
-- Before
if table.Count(t) == 0 then ...
-- After
if next(t) == nil then ...
```
Replace all "is empty" guards with `next(t) == nil` and all "has items" guards with
`next(t) ~= nil`.  For `Size()` that is used purely as a boolean, also prefer `next`.

**Expected gain**: measurable FPS improvement on servers with many entities.

---

## 2  Cache repeated network-amount lookups inside loops

**File**: `lua/entities/rd_pump/init.lua` (lines ~287–295)

**Problem**  
`RD.GetNetResourceAmount(self.netid, k)` is called 2–3 times per resource per think
tick inside a `pairs` loop.  Each call traverses the network table.

**Fix**
```lua
for k, v in pairs(self.ResourcesToSend) do
    local amt = RD.GetNetResourceAmount(self.netid, k)
    if amt > 0 then
        self:Send(k, amt < v and amt or v)
    end
end
```

---

## 3  Avoid modifying tables during `pairs` iteration

**Files**: `lua/caf/addons/server/resourcedistribution.lua` (lines ~927–942),
`lua/includes/modules/arraylist.lua` `Remove` (already fixed in this PR for the
first-match case).

**Problem**  
`table.remove` inside a `pairs` loop can skip elements or produce incorrect results
because the iterator state is not updated after removal.

**Fix pattern** – collect keys first, then remove:
```lua
local toremove = {}
for k, v in pairs(t) do
    if condition(v) then toremove[#toremove+1] = k end
end
for _, k in ipairs(toremove) do t[k] = nil end
```

---

## 4  Throttle `Think`-based entity updates with timers

**Files**: `lua/entities/storage_energy/init.lua`, and other entities that run
`ents.FindInSphere` or `ents.FindByClass` directly inside `Think`.

**Problem**  
`ents.FindInSphere` is an expensive engine call.  Running it every server tick
(~66 times/second) for every storage entity causes unnecessary load.

**Fix**  
Use a timer or a `CurTime()` throttle gate:
```lua
function ENT:Think()
    if CurTime() < (self._nextCheck or 0) then return end
    self._nextCheck = CurTime() + 1  -- run once per second at most
    -- expensive work here
end
```

---

## 5  Batch network messages

**File**: `lua/autorun/server/sv_caf_autostart.lua` `PlayerSpawn`, and
`lua/caf/addons/server/resourcedistribution.lua` network sends.

**Problem**  
Each active addon sends its own `net.Start / net.Send` message separately when a
player joins.  Many small messages increase network overhead and can cause packet
spam.

**Fix**  
Collect all addon names into a single net message:
```lua
net.Start("CAF_Addon_List")
    net.WriteUInt(#activeAddons, 16)
    for _, name in ipairs(activeAddons) do net.WriteString(name) end
net.Send(ply)
```
Add a matching client-side receiver that reconstructs the list.

---

## 6  Recycle cache timer IDs to prevent memory accumulation

**File**: `lua/includes/modules/cache.lua`

**Problem**  
The module-level `id` counter only increments and is never recycled.  Every cache
object created during a game session registers a timer with a unique ever-growing
name; old timers are never cleaned up when a cache object is garbage collected.

**Fix**  
Store the timer name inside the cache object and expose a `destroy()` method:
```lua
function list:destroy()
    timer.Remove(self._timerName)
    self.contents = nil
end
```
Call `cache:destroy()` wherever a cache object's lifetime ends.

---

## 7  Remove dead / commented-out code

**Files**  
- `lua/caf/addons/server/resourcedistribution.lua` – large blocks of commented-out
  `umsg` code (the old messaging system, lines ~24–202).
- `lua/caf/core/server/module_loader.lua` – commented test cache block.
- `lua/caf/core/shared/sh_general_caf.lua` – large commented test block.
- `lua/entities/base_rd3_entity/init.lua` – commented-out `AcceptInput` redefined
  below.

**Action**: Delete dead code or move it to a dedicated `docs/legacy/` archive.
Commented-out code creates confusion about what is active.

---

## 8  Add nil-safety guards in hot paths

**Files**: `lua/caf/addons/server/resourcedistribution.lua`,
`lua/entities/storage_energy/init.lua`

**Problem**  
- `list.Get("LSEntOverlayText")[tmpdata.ent:GetClass()]` – if `list.Get` returns nil
  the index crashes.
- `CAF.GetAddon("Life Support").ZapMe` – same pattern; crashes if addon is absent.

**Fix**  
```lua
-- Pattern
local addon = CAF.GetAddon("Life Support")
if addon then zapme = addon.ZapMe end
```
Apply this pattern everywhere an addon or list lookup result is immediately indexed.

---

## 9  Replace `pairs` with `ipairs` for sequential-array tables

**Files**: `lua/includes/modules/arraylist.lua`, `lua/includes/modules/hashmap.lua`

**Problem**  
The internal `.table` of `ArrayList` is always a sequential array (`table.insert`
appends).  Using `pairs` on sequential arrays adds overhead compared to `ipairs`
(which avoids hash-part iteration) and may visit keys in unexpected order.

**Fix**  
Replace every `for k, v in pairs(self.table)` in `arraylist.lua` with
`for k, v in ipairs(self.table)` where sequential order is required.

---

## 10  Move `AddCSLuaFile` path strings to constants

**File**: `lua/autorun/server/sv_caf_autostart.lua`

**Problem**  
`file.Find` is called with mixed-case paths (`"CAF/Core/client/*.lua"` vs
`"caf/core/server/*.lua"`).  On case-sensitive file systems (Linux) the uppercase
variants will silently find no files.

**Fix**  
Normalise all paths to lowercase and define them as constants at the top of the
file:
```lua
local PATH_SERVER  = "caf/core/server/"
local PATH_CLIENT  = "caf/core/client/"
local PATH_SHARED  = "caf/core/shared/"
```

---

## Summary Table

| # | Area | Effort | Impact |
|---|------|--------|--------|
| 1 | Replace `table.Count()` → `next()` | Low | High |
| 2 | Cache network amount lookups | Low | Medium |
| 3 | Safe table removal pattern | Medium | Medium |
| 4 | Throttle Think-based scans | Low | High |
| 5 | Batch net messages on join | Medium | Medium |
| 6 | Cache timer lifecycle | Low | Low |
| 7 | Remove dead code | Low | Maintainability |
| 8 | Nil-safety guards | Low | Stability |
| 9 | `ipairs` for sequential tables | Low | Low |
| 10 | Fix mixed-case file paths | Low | Correctness (Linux) |
