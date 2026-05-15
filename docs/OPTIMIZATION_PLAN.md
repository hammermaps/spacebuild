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

---

## 11  Migrate active `umsg` usage to `net` library

**Files**  
`lua/entities/base_rd3_entity/init.lua` (lines ~60–65),
`lua/caf/addons/server/spacebuild.lua` (`SendColorAndBloom`, `SendSunBeam`, lines ~88–134)

**Problem**  
`umsg` (the old user-message system) is deprecated in the GLua reference and was
replaced by the `net` library.  Two active code paths still use `umsg`:
- `AcceptInput` in `base_rd3_entity` sends input names via `umsg.Start("RD_AddInputToMenu")`.
- `SendColorAndBloom` / `SendSunBeam` in the Spacebuild server addon send planet and
  star data to joining players via `umsg.Start("AddPlanet")` / `umsg.Start("AddStar")`.

The `net` library offers proper message pooling, size safety, and is maintained.

**Fix pattern**
```lua
-- Before (umsg)
umsg.Start("RD_AddInputToMenu", caller)
    umsg.Bool(last)
    umsg.String(v.Name)
    umsg.Short(self:EntIndex())
umsg.End()

-- After (net)
util.AddNetworkString("RD_AddInputToMenu")  -- once, at file top
net.Start("RD_AddInputToMenu")
    net.WriteBit(last and 1 or 0)
    net.WriteString(v.Name)
    net.WriteInt(self:EntIndex(), 16)
net.Send(caller)
```
Add matching `net.Receive(...)` receivers on the client side.  
The large commented-out `umsg` block in `resourcedistribution.lua` (~lines 24–202) is
already dead code and can simply be deleted (see item 7).

**Expected gain**: future-proof messaging; no reliance on deprecated API.

---

## 12  Replace deprecated `SetNetworkedInt` / `GetNetworkedInt` with NW2 vars

**Files**  
`lua/entities/base_rd3_entity/init.lua`, `lua/entities/rd_pump/init.lua`,
`lua/entities/rd_one_way_valve/init.lua`, `lua/entities/rd_node_valve/init.lua`,
`lua/entities/rd_ent_valve/init.lua`, `lua/entities/resource_node/init.lua`,
`lua/entities/other_probe/init.lua`, `lua/entities/nature_plant/init.lua`
(and their matching `cl_init.lua` files)

**Problem**  
`Entity:SetNetworkedInt()` / `Entity:GetNetworkedInt()` are marked deprecated in the
GLua reference.  The replacement is the NW2 API (`SetNWInt` / `GetNWInt`), which has
lower overhead and explicit type safety.  The codebase uses `SetNetworkedInt` in
approximately 40 call-sites across 8 entity pairs.

**Fix**
```lua
-- Before
self:SetNetworkedInt("overlaymode", 1)
local mode = self:GetNetworkedInt("overlaymode")

-- After
self:SetNWInt("overlaymode", 1)
local mode = self:GetNWInt("overlaymode")
```
Repeat for all variant types used (`SetNetworkedBool` → `SetNWBool`, etc.).

**Expected gain**: API correctness; NW2 vars are batched and more efficient.

---

## 13  Replace `ents.Create` monkey-patch with `OnEntityCreated` hook

**File**: `lua/autorun/server/sv_caf_autostart.lua` (lines ~428–434)

**Problem**  
The bootstrap file overwrites the engine global `ents.Create`:
```lua
local oldcreate = ents.Create
ents.Create = function(class)
    local ent = oldcreate(class)
    timer.Simple(0.1, function() OnEntitySpawn(ent, "SENT") end)
    return ent
end
```
Monkey-patching engine globals is fragile: other addons that do the same will silently
break each other, and the 0.1 s timer fires even when the entity is already invalid.

The GLua reference provides the proper `OnEntityCreated(entity)` game hook, which fires
immediately on every entity creation without the need for overriding globals.

**Fix**
```lua
hook.Add("OnEntityCreated", "CAF_OnEntityCreated", function(ent)
    if not IsValid(ent) then return end
    timer.Simple(0, function()   -- defer one tick so Initialize() has run
        if IsValid(ent) then OnEntitySpawn(ent, "SENT") end
    end)
end)
```
Remove the `ents.Create` override entirely.

**Expected gain**: compatibility with other addons; no silent clobbering of engine
functions.

---

## 14  Add `EntityRemoved` hook for RD and LS table cleanup

**Files**  
`lua/caf/addons/server/resourcedistribution.lua`,
`lua/caf/addons/server/lifesupport.lua`

**Problem**  
RD entity cleanup currently relies solely on per-entity `ENT:OnRemove()` defined in
`base_rd3_entity`.  Entities that do not inherit from this base (map entities,
non-RD props destroyed by explosions, etc.) are never cleaned out of `ent_table` /
`nettable`.  Similarly, the LS `CheckRegulators()` function polls for invalid entries
inside the think loop instead of reacting to removal events.

The GLua reference provides `EntityRemoved(entity)` – a global server-side hook that
fires for every entity removal, regardless of class or base.

**Fix**
```lua
hook.Add("EntityRemoved", "CAF_RD_EntityRemoved", function(ent)
    local RD = CAF.GetAddon("Resource Distribution")
    if RD and RD.GetStatus() then
        RD.Unlink(ent)
        RD.RemoveRDEntity(ent)
    end
    local LS = CAF.GetAddon("Life Support")
    if LS and LS.GetStatus() then
        LS.RemoveAirRegulator(ent)
        LS.RemoveTemperatureRegulator(ent)
    end
end)
```
This also eliminates the need for the `CheckRegulators()` polling loop in
`lifesupport.lua`.

**Expected gain**: no stale table entries; removal of a polling loop.

---

## 15  Add `PlayerDisconnected` hook to prevent player-data leaks

**File**: `lua/caf/addons/server/lifesupport.lua`

**Problem**  
When a player disconnects, their Lua `Player` object is garbage-collected but the suit
data tables (`ply.suit`, `ply.caf.custom.ls`) are never explicitly cleared.  On servers
with many connects/disconnects this creates a small accumulation.  More importantly, if
any LS think function iterates `player.GetAll()` and a disconnect happens mid-tick, a
stale reference can cause errors.

The GLua reference provides `PlayerDisconnected(player)` – called right before the
player object is removed.

**Fix**
```lua
local function LSPlayerDisconnected(ply)
    if ply.suit then ply.suit = nil end
    if ply.caf and ply.caf.custom then ply.caf.custom.ls = nil end
end
-- Register in LS.__Construct, remove in LS.__Destruct:
hook.Add("PlayerDisconnected", "LS_PlayerDisconnected", LSPlayerDisconnected)
```

**Expected gain**: explicit cleanup; prevents errors from stale references.

---

## 16  Add `PlayerDeath` hook for consistent LS death effects

**File**: `lua/caf/addons/server/lifesupport.lua`

**Problem**  
The `PlayerKilled` hook that would trigger ragdoll / death-in-space effects is entirely
commented out in `spacebuild.lua`.  Life-support death currently has no hook reaction –
players who die from LS effects (suffocation, temperature, pressure) are handled only
through the `LS.DamageLS` path without any server-side death event.

The GLua reference provides `PlayerDeath(player, inflictor, attacker)` which fires
reliably for every player death, regardless of cause.

**Fix**
```lua
local function LSPlayerDeath(ply, inflictor, attacker)
    -- Reset suit so respawn begins from a clean state;
    -- do not call LsResetSuit here because PlayerSpawn handles the respawn reset.
    if ply.caf and ply.caf.custom and ply.caf.custom.ls then
        ply.caf.custom.ls.inspace = false
    end
end
hook.Add("PlayerDeath", "LS_PlayerDeath", LSPlayerDeath)
```
Register in `LS.__Construct`, remove in `LS.__Destruct`.

**Expected gain**: correct LS state reset on death; foundation for future death
effects (ragdolls in zero-g, etc.).

---

## 17  Add `PostCleanupMap` hook to reset SB/RD/LS state

**Files**  
`lua/caf/addons/server/spacebuild.lua`,
`lua/caf/addons/server/resourcedistribution.lua`

**Problem**  
When an admin calls `game.CleanUpMap()` (or a player uses the sandbox "clean" button),
the Spacebuild planet/star lists (`Planets`, `Stars`), the RD network tables
(`nettable`, `ent_table`), and the LS generator lists are not cleared.  After a cleanup
the server still references destroyed entities.  `game.CleanUpMap` is already monkey-
patched in `spacebuild.lua` to preserve map entities, but there is no post-cleanup
reset.

The GLua reference provides `PostCleanupMap()` – called after all entities are removed
during a map cleanup.

**Fix**
```lua
hook.Add("PostCleanupMap", "SB_PostCleanupMap", function()
    -- Re-run environment registration
    Register_Environments()
    Register_Sun()
    -- RD: reset all transient network tables
    RD.ResetAll()   -- expose a reset function in resourcedistribution.lua
    -- LS: rebuild generator lists
    LS.generators.air = {}
    LS.generators.temperature = {}
    -- Re-init all currently connected players
    for _, ply in ipairs(player.GetAll()) do
        LSSpawnFunc(ply)
    end
end)
```

**Expected gain**: correct state after map cleanup; no stale entity references.

---

## 18  Add `PlayerLeaveVehicle` hook to deregister vehicle from RD

**File**: `lua/caf/addons/server/lifesupport.lua`

**Problem**  
`PlayerSpawnedVehicle` registers vehicles as non-storage RD devices.  There is no
complementary hook that removes the RD registration when the vehicle is despawned or
the player disconnects mid-ride.  The `EntityRemoved` hook (item 14) would partially
cover this, but `PlayerLeaveVehicle` offers an earlier, cleaner deregistration
opportunity and can update the player's environment assignment when they exit a vehicle
that had its own atmosphere.

The GLua reference provides `PlayerLeaveVehicle(player, vehicle)`.

**Fix**
```lua
local function LS_LeaveVeh(ply, veh)
    -- If vehicle provided its own RD-linked atmosphere, reassign player environment
    if IsValid(ply) and ply.environment then
        local SB = CAF.GetAddon("Spacebuild")
        if SB then SB.UpdatePlayerEnvironment(ply) end
    end
end
hook.Add("PlayerLeaveVehicle", "LS_vehicle_leave", LS_LeaveVeh)
```
Register in `LS.__Construct`, remove in `LS.__Destruct`.

**Expected gain**: correct environment re-assignment when exiting a vehicle;
cleaner lifecycle management.

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
| 11 | Migrate active `umsg` → `net` | Medium | Correctness / API |
| 12 | Replace deprecated `SetNetworkedInt` → `SetNWInt` | Medium | API correctness |
| 13 | Replace `ents.Create` patch → `OnEntityCreated` hook | Low | Stability |
| 14 | Add `EntityRemoved` hook | Low | Stability / Perf |
| 15 | Add `PlayerDisconnected` hook | Low | Stability |
| 16 | Add `PlayerDeath` hook | Low | Correctness |
| 17 | Add `PostCleanupMap` hook | Medium | Correctness |
| 18 | Add `PlayerLeaveVehicle` hook | Low | Correctness |
