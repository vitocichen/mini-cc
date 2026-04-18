# MiniCC 3.9.0 — Local Patched Build (duel + instance-refresh fixes)

This folder is a **snapshot of the locally-installed MiniCC 3.9.0** with two
bug fixes applied. It is checked in on branch `fix-duel-and-instance-refresh`
as a standalone backup, separate from the repository's `src/` layout.

## What was fixed

All changes are contained in `Modules/NameplatesModule.lua`.

### Fix 1 — Buff/CC icons missing on friendly duel opponents

**Symptom**: Accepting a duel from a friendly player; the opponent starts
fighting you but the CC / Important icons on their nameplate never appear,
even though the aura is active. The user had to **manually toggle the
`敌人-控制` / `敌人-重要` (Enemy.CC / Enemy.Important) checkboxes off and on
to force the icons to show up**.

**Root cause**: `OnNamePlateAdded` used to early-return (before creating
`nameplateAnchors[unit]` and `watchers[unit]`) whenever the current faction's
options had no mode enabled. For a friendly player with `Friendly.*` all
disabled, no watcher was ever attached to that nameplate. When the duel
started and `GetUnitOptions()` began returning the Enemy table, no watcher
was listening for `UNIT_AURA` and `OnAuraDataChanged` never fired, so
containers were never materialized.

(The original author confirmed: *"It's because the duel unit is still
considered friendly instead of an enemy unit, it's been a bug for a while
I'm just not sure how to fix it."*)

**Fix**:
1. `OnNamePlateAdded` now keeps `data + watcher` alive whenever **either**
   the current faction OR the opposite faction has any mode enabled. This
   ensures a watcher is always listening across duel start/end transitions.
2. `OnAuraDataChanged` now self-heals: when it detects that the currently
   active `unitOptions` requires a container that is `nil`, it calls
   `EnsureContainersForNameplate` inline to materialize it before dispatching
   to `ApplyCcToNameplate` / `ApplyImportantSpellsToNameplate`.
3. Extra event registrations in `M:Init()`: `UNIT_FACTION`,
   `PLAYER_FLAGS_CHANGED` — defensive in case a future Blizzard change makes
   `UnitIsEnemy` actually flip during duels, or when similar faction edge
   cases occur.

### Fix 2 — Icons missing after zoning into instance/BG/arena

**Symptom**: On login (e.g. in a capital city with `Always=false` and no
other gate matched), the module initialized with `IsModuleEnabled()==false`
and therefore never ran `ApplyBlizzardNameplateSettings()`. After zoning
into a battleground / dungeon / arena, the module should now be active, but
nothing re-evaluated — users had to toggle the `Enabled` checkbox to force a
`M:Refresh()` which then called `ApplyBlizzardNameplateSettings` and set the
Blizzard nameplate aura CVars, at which point icons finally appeared.

**Fix**: `M:Init()` now registers `PLAYER_ENTERING_WORLD` and
`ZONE_CHANGED_NEW_AREA`. On those events, the handler tracks a
`previousModuleActive` flag and, on a `disabled -> enabled` context
transition, directly calls:

```
ApplyBlizzardNameplateSettings()
EnableWatchers()
RebuildContainers()
```

This bypasses `HaveModesChanged()`, which only tracks *config* changes and
is blind to *context* changes like entering an instance.

## Layout

```
backup-3.9.0-with-fixes/
├── Modules/
│   └── NameplatesModule.lua   <-- contains both fixes
├── ... (rest of the 3.9.0 addon, unmodified)
└── README-FIXES.md            <-- this file
```

## How to apply manually

Drop `Modules/NameplatesModule.lua` into your installed MiniCC 3.9.0 and
`/reload`. No SavedVariables or DB schema changes.
