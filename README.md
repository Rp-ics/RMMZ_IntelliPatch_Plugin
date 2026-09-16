# IntelliPatch — Dynamic Reactive Pathfinding for RPG Maker MZ

Tile-based A* (binary heap + Map closed set), programmable navigation patches, per-frame round-robin recalc scheduler, path smoothing, closest-reachable fallback, composable Goal System, 12 expert extension hooks and a visual debug overlay. By **Rpx & Just Dev**.

- **IntelliPatch Lite** (free): the production-ready core — A*, cost/block patches, recalc + giveup policies, 10 Plugin Commands, 4-color debug with `LITE` watermark. See [`Lite/README.md`](Lite/README.md).
- **IntelliPatch Pro** (paid, v2.0.1): everything in Lite plus the full Goal System (20 modes), force/portal/conditional patches, origin/back_retry/random/ghost policies, 13 extra commands, `register*` expert hooks, custom steering, purple/cyan debug. This guide covers both; Pro-only features are marked **[Pro]**.

> **Note:** this repository contains **guides and documentation only** (`.md` files). The plugin files (`IntelliPatch.js`, `IntelliPatchLite.js`) are distributed separately — see *Terms of Use* below.

## Contents

1. [Lite vs Pro](#1-lite-vs-pro)
2. [Installation](#2-installation)
3. [Quick start](#3-quick-start)
4. [Configuration](#4-configuration)
5. [Core concepts](#5-core-concepts)
6. [Movement API](#6-movement-api)
7. [Touch triggers & Allow Touch](#7-touch-triggers--allow-touch)
8. [Goal System **[Pro]**](#8-goal-system-pro)
9. [Patch types](#9-patch-types)
10. [Stuck policies](#10-stuck-policies)
11. [Plugin Commands](#11-plugin-commands)
12. [Debug](#12-debug)
13. [Expert extension API **[Pro]**](#13-expert-extension-api-pro)
14. [Performance tuning](#14-performance-tuning)
15. [Migration from VisuStella pathfinding](#15-migration-from-visustella-pathfinding)
16. [Troubleshooting](#16-troubleshooting)
17. [Compatibility](#17-compatibility)
18. [File structure](#18-file-structure)
19. [Terms of Use / Credits](#19-terms-of-use--credits)

## 1. Lite vs Pro

| Feature | Lite (free) | Pro (paid) |
|---|---|---|
| A* with binary heap, smoothing, fallback, portal loop guard | ✅ | ✅ |
| Revision-keyed cache + round-robin recalc scheduler | ✅ | ✅ |
| `moveTo` · `moveToPath` · `follow` · `wander` · `cancel` · `cancelAll` · `getPath` · `hasPath` | ✅ | ✅ |
| `allowTouch` destination contact (Player/Event Touch) | ✅ | ✅ |
| Patches: `cost` (mud, wind), `block` (walls, doors) | ✅ | ✅ |
| Patches: `force` (currents), `portal` (teleports), `conditional` (JS callbacks) | ❌ | ✅ |
| Stuck policies: `recalc`, `giveup` | ✅ | ✅ |
| Stuck policies: `origin`, `back_retry`, `random`, `ghost` | ❌ | ✅ |
| Goal System (20 modes, nesting, avoid, interrupts, waits, barriers, timeouts) | ❌ | ✅ |
| 10 core Plugin Commands | ✅ | ✅ |
| 13 extra commands (goals, portals, raw JSON patches) | ❌ | ✅ |
| 12 `register*` expert hooks + custom steering | ❌ | ✅ |
| Debug overlay (green/red/yellow/gray) + `debug()` + `debugTile()` | ✅ | ✅ |
| Purple goal points + cyan target line, `debugGoal()`, `debugRegistries()` | ❌ | ✅ |

Do NOT install Lite and Pro in the same project (both use one global).

## 2. Installation

1. Copy `IntelliPatch.js` (Pro) **or** `Lite/IntelliPatchLite.js` (Lite) into your project's `js/plugins/` folder.
2. Open Plugin Manager (F10), add it, configure the params (14 Pro / 12 Lite).
3. Load order (top → bottom): movement plugins (Altimit/QMovement, VisuStella) → **IntelliPatch** → anything overriding `Game_CharacterBase.update`.
4. Start with defaults (`balanced`, `DiagonalMode off`), enable `DebugMode` while building, disable before release.

No dependencies. No setup script calls required.

## 3. Quick start

From an event Script call or Move Route → Script (where `this` is the event):

```js
IntelliPatch.moveTo(this, 15, 20);
```

Or from the editor with Plugin Commands (`MoveTo`, `FollowTarget`, `WanderArea`, …). Native move-route commands placed BEFORE the Script line run first; after it, the Navigator takes over (only while the character is not already moving).

## 4. Configuration

Pro has 14 params, Lite has 12 (no `GhostMaxPass` / `EscapeRadius`).

| Param | Type | Default | Notes |
|---|---|---|---|
| MaxIterations | number | 2000 | A* node expansion cap per search. |
| RecalcThreshold | number | 2 | Tiles the target must move before auto-recalc. 0 is treated as 1. |
| DiagonalMode | select | off | `off` = cardinal only. `all` = lenient. `auto` = strict, BOTH adjacent orthogonals passable (no corner-cutting, recommended). |
| DefaultHeuristic | select | manhattan | `manhattan` (cardinal) / `octile` (diagonal) / `euclidean`. |
| PerformanceMode | select | balanced | `safe` = 5 paths/sec. `balanced` = PathsPerSecond. `aggressive` = unthrottled, iters capped at 500. |
| PathsPerSecond | number | 10 | Global token bucket for path computations per second. |
| MaxRecalcQueue | number | 5 | Pending recalcs per navigator; excess merges, latest target wins. |
| PathSmoothing | boolean | true | String-pulling via Bresenham line-of-sight (lookahead 8). |
| AllowUnreachableFallback | boolean | true | Walk to closest-reachable tile, set `fallbackUsed`. |
| DebugMode | boolean | false | Overlay + console diagnostics. |
| DefaultBlockedPolicy | select | recalc | Pro: `recalc origin back_retry random ghost giveup`. Lite: `recalc giveup`. |
| BlockedRetries | number | 3 | Alternative-recalc attempts before the stuck policy runs. |
| GhostMaxPass **[Pro]** | number | 2 | Max blocked tiles to ghost through (collision OFF, patches still apply). |
| EscapeRadius **[Pro]** | number | 4 | `random` / `back_retry` wander radius in tiles. |

Runtime override: `IntelliPatch.setGlobalConfig("debug", true)` (keys mirror the params, `debug` = DebugMode).

## 5. Core concepts

- **Navigator** — one per character, states `idle | following | wandering | paused | stuck | failed`. You order it (`moveTo`, goals…), it paths, steps with `moveStraight` + a custom diagonal (never native `moveDiagonally`), and reports via `onArrive / onFail / onRecalc / onBlocked`.
- **Recalc** — a new road is computed when the target moves beyond `RecalcThreshold`, when the event is stuck, when a patch changes, or when the next tile becomes occupied. Throttled by the token bucket; bursts are allowed for empty paths so new orders never look frozen.
- **Cache** — paths are keyed by map + endpoints + `patchRevision` + flags and validated against live events on every hit, so moved events can never loop a stale road.
- **Patches** — named zones that change tile cost, block tiles, push movement, teleport or run JS per tile. Every mutation bumps `patchRevision`.
- **Never destructive** — the Navigator never clears your native move route (`cancel` keeps it, `cancelAll` clears it) and only drives while `isMoving() === false`. Passability is read-only (`isPassable`, `eventsXy`), so third-party `canPass` overrides keep working.

## 6. Movement API

```js
IntelliPatch.moveTo(character, x, y, options)
IntelliPatch.moveToPath(character, [[x,y],[x,y]], options)
IntelliPatch.follow(character, targetCharacter, distance, options)
IntelliPatch.wander(character, area, options)
IntelliPatch.moveToGoal(character, goal, options)   // [Pro]
IntelliPatch.cancel(character)      // stops Navigator only, native route survives
IntelliPatch.cancelAll(character)   // stops Navigator AND clears native route
IntelliPatch.addPatch(config)       // Lite: type cost|block only
IntelliPatch.removePatch(id)
IntelliPatch.setPatchEnabled(id, enabled)           // [Pro] (Lite: script API only via add/remove)
IntelliPatch.getPatch(id)                           // [Pro]
IntelliPatch.getPath(character)     // copy with .fallbackUsed, for debug
IntelliPatch.hasPath(character)
IntelliPatch.setGlobalConfig(key, value)
IntelliPatch.debug()                // dump every navigator
IntelliPatch.debugTile(x, y)        // list every patch on a tile
IntelliPatch.debugGoal(eventId)     // [Pro] dump one goal tree
IntelliPatch.debugRegistries()      // [Pro] list every registry entry
```

`options`:

```js
{
  recalcThreshold: 2,   // 0 treated as 1
  maxIterations: 2000,
  diagonal: false,      // true forces lenient diagonals even when DiagonalMode is off
  avoidEvents: true,    // other events are avoided en route...
  allowTouch: true,     // ...destination contact ON by default (set false to stop adjacent)
  avoidPlayer: false,   // (fires Player Touch / Event Touch triggers on overlap)
  priority: "normal",   // low | normal | high (scheduler; FIFO within level)
  heuristic: "manhattan",
  stuckPolicy: "recalc",// Pro: recalc|origin|back_retry|random|ghost|giveup. Lite: recalc|giveup
  blockedRetries: 3,
  ghostPass: 2,         // [Pro] ghost only
  escapeRadius: 4,      // [Pro] random/back_retry only
  steering: "",         // [Pro] registered steering behavior name
  steeringSmooth: false,// [Pro] smooth vector mode (see expert API)
  onArrive: function(){},
  onFail: function(){},
  onRecalc: function(newPath){},
  onBlocked: function(info){},   // {policy, reason, attempts, target}
  onInterrupt: function(o,n){}   // [Pro] goals only
}
```

- `follow()` keeps approximately N tiles: stops when closer, resumes when farther. Not an exact orbit. `distance: 0` + `allowTouch` walks onto the target's tile.
- `moveToPath()` waypoint rule: per waypoint try `findPath`, else closest-reachable fallback when allowed, else abort everything and call `onFail()`.
- `addPatch()` on ID collision silently overwrites (idempotent, no error).
- Characters: pass `this` inside Move Route → Script, `$gamePlayer`, or `$gameMap.event(n)`.

## 7. Touch triggers & Allow Touch

The problem: with avoidance on, a chaser stops **adjacent** to its target — so Player Touch / Event Touch triggers never fire, because MZ fires them on tile **overlap**.

The fix: `allowTouch` (option or the `Allow Touch` command arg, **ON by default**; pass `false` to stop adjacent).

- En-route tiles are still avoided; only the **destination** tile may be occupied.
- Patch blocks still apply, even on the target (a shut door stays shut).
- The executor takes the final step with collision bypassed for **one step only** (`through` is restored immediately after), then arrival fires normally.

Chaser setup (monster with Event Touch chases the player into contact):

```js
// Move Route → Script,runs every few frames or on a loop:
IntelliPatch.moveTo(this, $gamePlayer.x, $gamePlayer.y, {
  avoidEvents: true, allowTouch: true,
  recalcThreshold: 1,
  onArrive: function(){ /* overlap: Event Touch fires via the engine */ }
});
```

Player walking onto a Player-Touch trap/switch with avoidance on:

```js
IntelliPatch.moveTo($gamePlayer, 12, 8, { avoidEvents: true, allowTouch: true });
```

With `allowTouch: false`, the mover stops adjacent (fallback arrival) — correct for Action-Button talks, wrong for touch triggers. **Important:** make sure the project runs plugin v2.0.1+ / Lite v1.0.1+ (check with F12: `IntelliPatch._version`) — older files ignore the option entirely.

## 8. Goal System **[Pro]**

`IntelliPatch.moveToGoal(character, goal, options)` drives a **declarative** goal instead of a raw tile. Plain configs are normalized (`new Goal()`); `Goal` instances are shared as-is so interleave counters and barrier groups stay common across navigators.

```js
IntelliPatch.moveToGoal(this, {
  mode: "sequence",
  points: [{ x: 5, y: 5 }, { x: 10, y: 10, wait: { frames: 60 } }],
  onPointReached: function(point, index){},
  onGoalComplete: function(){},
  onGoalFailed: function(reason){}
});
```

Goal shape: `{ mode, points: [PointGoal|Goal], events: [EventGoal], avoid: [...], weights, tolerance, loop, timeout, interrupt, onPointReached, onGoalComplete, onGoalFailed, id }`. Points nest other Goals (recursive). `PointGoal = {x, y, weight?, wait?, while?, condition?}`. `EventGoal = {id, at: {x,y}|[...], adjacency?}`. `WaitCondition = {frames?, switch?+value?, signal?, playerNear?, condition?}` (all present keys must pass).

| Mode | Behavior |
|---|---|
| `any` / `oneOf` | Nearest reachable point (A* verified). One-shot. |
| `all` | Every point, nearest-first. Done when all visited. |
| `sequence` | Points in order. |
| `sequenceThenLoop` | Sequence, wraps to 0 (`loop` equivalent inline). |
| `sequenceThenReverse` | Ping-pong 0→N→0 (patrols). |
| `closest` / `farthest` | Min / max Manhattan (closest is A* verified). One-shot. |
| `random` | Uniform pick. One-shot. |
| `weighted` | Weighted pick via `weights`. One-shot. |
| `conditional` | First point whose `condition()` passes. |
| `conditionalSequence` | Sequence where each step's `while` gate must pass before advancing. |
| `paired` | Two navigators (roles A/B via `_barrierGroup`), both must arrive. |
| `swap` | A walks to B's start and vice versa (`swapPeer` wiring). |
| `interleave` | N navigators share one goal, points dealt round-robin. |
| `simultaneous` | Each navigator gets its point (`_simulIndex`); arrivals barriered within `tolerance` frames. |
| `eventAt` | An event must reach its tile(s). Done when in place. |
| `eventAtAny` | An event must reach ONE of the tiles (nearest first). |
| `twoEventsMeet` | Both events end within `adjacency` (minimax midpoint). |
| `guardPost` | `{id, at}` list — each event resolves its own post. |
| `custom` | Your resolver via `registerGoalMode`. |

More behaviors: `avoid` soft-blocks tiles (all-avoided → `done` with reason), `interrupt: [{when, then, priority}]` swaps the goal mid-flight (checked every 15 frames, deep-cloned `then`, `onInterrupt` fires), `timeout` frames fails the journey, `loop: true` restarts on completion. Arrival at a short fallback (target unreached) counts as blocked, never as success.

Patrol that waits at each post and restarts:

```js
IntelliPatch.moveToGoal(this, {
  mode: "sequenceThenLoop",
  points: [{ x: 4, y: 4, wait: { frames: 90 } }, { x: 12, y: 4, wait: { frames: 90 } }]
});
```

Flee to whichever exit switch is ON:

```js
IntelliPatch.moveToGoal(this, {
  mode: "conditional",
  points: [
    { x: 2, y: 2, condition: function(){ return $gameSwitches.value(11); } },
    { x: 18, y: 14 }
  ]
});
```

## 9. Patch types

```js
IntelliPatch.addPatch({ id: "mud", type: "cost",
  area: { x: 3, y: 3, w: 5, h: 5 }, data: { multiplier: 3.0 } });
// data: { multiplier, add, direction } — direction 2/4/6/8 limits `add` to that travel dir (wind).

IntelliPatch.addPatch({ id: "door", type: "block",
  area: { x: 12, y: 8, w: 1, h: 1 },
  condition: function(){ return !$gameSwitches.value(5); } });
// Blocked while condition is true (no condition = always blocked).

IntelliPatch.addPatch({ id: "current", type: "force",   // [Pro]
  area: { x: 0, y: 10, w: 20, h: 2 }, data: { direction: 6 } });
// Execution override (cheap with the flow, costly against it in planning).

IntelliPatch.addPatch({ id: "p1", type: "portal",       // [Pro]
  area: { x: 2, y: 2, w: 1, h: 1 }, data: { toX: 18, toY: 15 } });
// Portal loop guard skips re-entered portalId:x,y branches per search.

IntelliPatch.addPatch({ id: "lava", type: "conditional", // [Pro]
  area: { x: 0, y: 0, w: 30, h: 30 },
  data: { callback: function(x, y, ch){ return $gameSwitches.value(1) ? 10 : 1; } } });
// Return false = blocked, number = cost multiplier.
```

`area` may also be a predicate: `area: function(x, y){ return $gameMap.regionId(x, y) === 2; }`. `priority` (default 0) sorts evaluation, higher first. Every mutation bumps `patchRevision` and lazily invalidates the cache.

## 10. Stuck policies

An alternative road is ALWAYS tried first (up to `blockedRetries` recalcs, with live cache validation). Only then does `stuckPolicy` run:

| Policy | Behavior |
|---|---|
| `recalc` (default) | Keep hunting (backs off, never gives up while the map can change). Guards/patrols. |
| `giveup` | Stay still (`failed` + `onFail()`). When standing still is correct. |
| `origin` **[Pro]** | Walk back to the order start, then `onFail()`. Fetch quests that must reset. |
| `back_retry` **[Pro]** | Step back along the walked trail, retry (up to `blockedRetries` rounds). Corridors. |
| `random` **[Pro]** | Escape-wander within `escapeRadius`, retry. Crowds. |
| `ghost` **[Pro]** | Collision OFF for up to `ghostPass` tiles, then back ON. Patches STILL apply. Cutscenes that must arrive. |

```js
// Cutscene NPC that MUST arrive, phasing through at most 3 blocked tiles:
IntelliPatch.moveTo(this, 15, 20, { stuckPolicy: "ghost", ghostPass: 3, blockedRetries: 2 });
```

## 11. Plugin Commands

All take `eventId` with `0 = player, -1 = this event, N = map event N`. Movement commands share `blockedPolicy` (`default` = DefaultBlockedPolicy) + `blockedRetries`, plus `allowTouch` and (Pro) `ghostPass`/`escapeRadius`.

Core (Lite + Pro): **MoveTo** (eventId, x, y, diagonal, avoidEvents, allowTouch) · **FollowTarget** (followerId, targetId, distance, allowTouch) · **WanderArea** (eventId, x1, y1, x2, y2, avoidEvents, allowTouch) · **ClearPath** (eventId) · **MoveToPath** (eventId, pathString `"5,5;10,10"`, diagonal, allowTouch — per-waypoint fallback rule applies) · **WaitForArrival** (eventId, timeoutFrames — blocks the interpreter until idle/failed or timeout) · **AddCostPatch** (patchId, x, y, w, h, multiplier) · **AddBlockPatch** (patchId, x, y, w, h) · **RemovePatch** (patchId) · **DebugToggle** (enabled).

Pro extra: **AddPatch** (patchId, config JSON) · **SetPatchEnabled** (patchId, enabled) · **AddPortalPatch** (patchId, x1, y1, x2, y2) · **MoveToGoal** (eventId, mode, pointsString, weightsString, loop, timeout, …) · **MoveToAny / MoveToSequence / MoveToClosest** (shortcuts) · **EventAt** (eventId, targetEventId, atString, adjacency) · **TwoEventsMeet** (eventA, eventB, adjacency, tolerance) · **GuardPost** (postString `"5@3,3;6@8,8"`) · **Simultaneous** (actorString `"5@2,2;6@10,2"`, tolerance) · **Swap** (eventA, eventB) · **ClearGoal** (eventId).

All commands validate input, warn in playtest, and never throw.

## 12. Debug

- Overlay colors: green = path, red = blocked, yellow = patched tile, gray = visited. Lite stamps a small `LITE` watermark in the corner.
- Per-navigator label: `len | recalcs | state | stuck | fallbackUsed | policy:attempts` (+ Pro goal suffix `goal=<mode>:<idx>/<total>`).
- Pro goal layer: purple point markers + cyan char→target line + `H/C/P/G/W/S` registry footer.
- Click any tile (DebugMode on) or call `IntelliPatch.debugTile(x, y)` to log every patch on it.
- `IntelliPatch.debug()` dumps all navigators; `IntelliPatch.debugGoal(eventId)` **[Pro]** dumps one goal tree (target, progress, wait, interrupts, barrier); `IntelliPatch.debugRegistries()` **[Pro]** lists every extension entry.

## 13. Expert extension API **[Pro]**

Everything is opt-in, validated (warn-and-`false`, never throws), idempotent (silent overwrite), documented in `@help → EXTENDING INTELLIPATCH`.

```js
IntelliPatch.registerHeuristic(name, fn)        // fn(x0,y0,x1,y1) -> number
IntelliPatch.registerCostFunction(name, fn)     // fn(character,x,y,d) -> multiplier (order kept)
IntelliPatch.registerPatchType(name, handlers)  // {applyCost?,isBlocking?,portalTarget?,forceDir?}
IntelliPatch.registerGoalMode(name, resolver)   // {resolve(goal,nav), validate(goal)?}
IntelliPatch.registerWaitCondition(name, fn)    // fn(value, navigator) -> bool
IntelliPatch.registerSteeringBehavior(name, fn) // fn(navigator, dt) -> {dx,dy} | direction
IntelliPatch.registerPatch(config)              // alias to addPatch (custom types ok)
IntelliPatch.registerCallback(hook, fn)         // onArrive|onFail|onRecalc|onBlocked|onInterrupt|onGoalComplete
IntelliPatch.registerPreRecalc(fn)              // fn(navigator) before every recalc
IntelliPatch.registerPostRecalc(fn)             // fn(navigator, path|null) after every recalc
IntelliPatch.registerOnMapChange(fn)            // fn(mapId) in Game_Map.setup
IntelliPatch.registerOnArrival(fn)              // fn(navigator, point) per waypoint arrival
```

Custom heuristic (rush toward the player's facing):

```js
IntelliPatch.registerHeuristic("biased", function(x0,y0,x1,y1) {
  var base = Math.abs(x0-x1) + Math.abs(y0-y1);
  if ($gamePlayer.direction() === 6 && x1 > x0) base *= 0.7;
  if ($gamePlayer.direction() === 4 && x1 < x0) base *= 0.7;
  return base;
});
IntelliPatch.setGlobalConfig("defaultHeuristic", "biased");
```

Custom wait + interrupt logging (e.g. with a trigger-area plugin):

```js
IntelliPatch.registerWaitCondition("inArea", function(areaId, nav) {
  return PolyTrigger2D.isInside(areaId, nav.character.eventId
    ? nav.character.eventId() : 0);
});
IntelliPatch.registerCallback("onInterrupt", function(nav, oldGoal, newGoal) {
  console.log("[IntelliPatch] goal change", oldGoal && oldGoal.id, "->", newGoal && newGoal.id);
});
```

Steering override for one order (grid-locked by default, `steeringSmooth: true` for vector mode):

```js
IntelliPatch.registerSteeringBehavior("east", function(nav, dt) { return { dx: 1, dy: 0 }; });
IntelliPatch.moveTo(this, 15, 5, { steering: "east" });
```

## 14. Performance tuning

- `safe` for 200+ active navigators on old hardware (5 paths/sec global, expect visible recalc delay).
- `balanced` as default for typical projects (token bucket from `PathsPerSecond`, 2000 iterations).
- `aggressive` for small maps with few navigators that need low-latency recalc (unthrottled, iterations capped at 500 to avoid freezes).

Further tips: shrink `MaxIterations` on huge open maps; raise `RecalcThreshold` for crowds; prefer `AddCostPatch` rects over per-tile conditional callbacks; keep `PathSmoothing` on (fewer steps = less movement overhead).

## 15. Migration from VisuStella pathfinding

1. Replace VisuMZ move-route path commands with `IntelliPatch.moveTo(this, x, y)` Script lines (native commands before the line still run).
2. Replace "move toward player" routes with `IntelliPatch.follow(this, $gamePlayer, 1)` — add `allowTouch: true` if the event must make contact (touch triggers).
3. Replace region-based slow tiles with `AddCostPatch` rects (same rectangles you used for VisuStella regions).
4. Keep VisuStella enabled and above IntelliPatch; remove only duplicate smart-path plugins (Shaz/Solar_Flare) to avoid double pathing.
5. Test with `DebugMode` on: green should track your old routes; yellow should cover your old slow regions.

## 16. Troubleshooting

- **Chaser never touches the player / switch never steps on**: `allowTouch` is ON by default — if the mover still stops adjacent, check the order doesn't pass `allowTouch: false`, and confirm the plugin version (`IntelliPatch._version` ≥ 2.0.1 Pro / 1.0.1 Lite). Without it the mover correctly stops adjacent.
- **Event walks through walls**: check tileset passability (O/X) and `DiagonalMode`; use `auto` to stop corner-cutting.
- **Event never arrives**: call `IntelliPatch.debug()`; `failed` + red tile = unreachable (check `AllowUnreachableFallback` and `debugTile` output).
- **Event stops for no reason**: steps advance strictly on arrival, empty paths retry every 20 frames, cached roads are validated against live events. Read the overlay `pol:attempts` label and pick a fitting policy (`ghost` for must-arrive, `origin` for reset, `giveup` for stay). A short fallback arrival (`fb=1`) without reaching the target counts as blocked, not success.
- **Jitter / constant recalc**: raise `RecalcThreshold`, lower `PathsPerSecond`, set wander priority `low`.
- **Portal loop / teleport spam [Pro]**: one portal per tile pair, distinct ids; the guard skips re-entry per search by design.
- **Goal never completes [Pro]**: `IntelliPatch.debugGoal(eventId)` shows mode, sub-index, armed waits and barrier state; check `timeout` and unsatisfied `while`/`wait` gates.
- **WaitForArrival never resumes**: timeout expires and releases the interpreter; check `hasPath()` and target validity.
- **No overlay**: `DebugMode` on, on Scene_Map (not menu/battle), spriteset present; click a tile to log patches.
- **Conflict**: confirm load order (VisuStella → IntelliPatch → CharacterBase-overriders); disable other pathfinders first; never install Lite + Pro together.

## 17. Compatibility

- VisuStella Events and Movement Core: detected via `Imported.VisuMZ_*`; the Navigator ticks on `Game_CharacterBase.update` (never `Game_Event.update`) so VisuStella overrides cannot miss or double ticks.
- Any plugin overriding `Game_Character.prototype.canPass`: respected (Pathfinder prefers `character.canPass`, falls back to `$gameMap.isPassable`).
- Read-only map access: `$gameMap.isPassable`, `$gameMap.eventsXy`, `$gameMap.isValid`; no core files modified.
- Save/load: navigators and compiled patch functions live in memory only and are never serialized; `Game_Map.setup` cancels all navigators, clears the cache, and bumps the revision. Re-issue paths via autorun after load/transfer if needed.
- NW.js (MZ Chromium): no top-level await, no `??=` / `||=` / `&&=`; all engine aliases guarded.

## 18. File structure

```
IntelliPatch/
├── README.md              ← this guide (Pro + Lite)
├── IntelliPatch.js           (distributed separately, not in this repo)
└── Lite/
    ├── README.md          ← Lite-specific guide
    └── IntelliPatchLite.js   (distributed separately, not in this repo)
```

Plugin internals — Pro `IntelliPatch.js` (single IIFE):

```
CONSTANTS / UTIL / EXTENSION REGISTRIES / BINARY HEAP / PATCH
GOAL / PATHFINDER / GOAL RESOLVER / WAIT CONDITION / RECALC SCHEDULER
NAVIGATOR / GLOBAL API / HOOKS ENGINE / PLUGIN COMMANDS / DEBUG
```

Lite `IntelliPatchLite.js` keeps the same skeleton minus Goal/Resolver/Wait/Registries and Pro-only branches.

## 19. Terms of Use / Credits

Free for commercial and non-commercial use. Redistribution of the plugin file is not allowed. Credit not required but appreciated. Copyright notice must remain intact.

Authors: Rpx & Just Dev. https://github.com/Rp-ics/RMMZ_IntelliPatch_Plugin
