# IntelliPatch Lite v1.0.4 (Free)

Dynamic reactive A* pathfinding with programmable navigation patches for **RPG Maker MZ** — the free edition. A serious, production-ready pathfinder: this is what most RMMZ games will actually use. Not a crippled demo.

Need goals, ghost phasing, portals, custom hooks and 13 more commands? Upgrade to **IntelliPatch Pro**.

## Features

- **Reactive A*** (binary heap + Map closed set): auto-recalc when the target moves beyond `RecalcThreshold`, when stuck, when a patch changes, or when the next tile becomes occupied.
- **Navigation patches**: `cost` (mud, wind) and `block` (walls, switch doors). Rect area or predicate function, optional condition, priority ordering.
- **Recalc scheduler**: exactly one navigator per frame, ordered high > normal > low with FIFO inside each level, global token bucket (`PathsPerSecond`), per-navigator queue cap (`MaxRecalcQueue`, latest wins).
- **Safe execution**: never clears native move routes (`cancel` vs `cancelAll`), only drives while `isMoving() === false`, custom diagonal step (never native `moveDiagonally`), read-only passability.
- **Closest-reachable fallback** + portal loop guard + revision-keyed path cache with live validation.
- **Debug**: green = path, red = blocked, yellow = patched, gray = visited. Per-navigator label `len | recalcs | state | stuck | fallbackUsed | policy:attempts`, click-to-inspect tile, `IntelliPatch.debug()` and `IntelliPatch.debugTile(x, y)`. This free edition stamps a small `LITE` watermark in the overlay corner.

## Installation

1. Copy `IntelliPatchLite.js` into your project's `js/plugins/` folder.
2. Open Plugin Manager (F10), add **IntelliPatch Lite**, configure the 12 params.
3. Load IntelliPatch Lite AFTER VisuStella Events and Movement Core, but BEFORE any plugin that overrides Game_CharacterBase.update.
4. Start with defaults (`balanced`, `DiagonalMode off`), enable `DebugMode` while building, disable before release.

Do NOT install Lite and Pro in the same project (both use one global). No dependencies.

## Configuration

| Param | Type | Default | Notes |
|---|---|---|---|
| MaxIterations | number | 2000 | A* node expansion cap per search. |
| RecalcThreshold | number | 2 | Tiles target must move before auto-recalc. 0 is treated as 1. |
| DiagonalMode | select | off | off = cardinal. all = lenient. auto = strict, BOTH adjacent orthogonals passable (no corner-cutting, recommended). |
| DefaultHeuristic | select | manhattan | manhattan / octile / euclidean. |
| PerformanceMode | select | balanced | safe = 5 paths/sec. balanced = PathsPerSecond. aggressive = unthrottled, iters capped at 500. |
| PathsPerSecond | number | 10 | Global token bucket for path computations per second. |
| MaxRecalcQueue | number | 5 | Pending recalcs per navigator; excess merges, latest target wins. |
| PathSmoothing | boolean | true | String-pulling via Bresenham line-of-sight (lookahead 8). |
| AllowUnreachableFallback | boolean | true | Walk to closest-reachable tile, set `fallbackUsed`. |
| DebugMode | boolean | false | Overlay + console diagnostics. |
| DefaultBlockedPolicy | select | recalc | recalc or giveup. Per-command `default` uses this. |
| BlockedRetries | number | 3 | Alternative-recalc attempts before the stuck policy runs. |

Runtime override: `IntelliPatch.setGlobalConfig("debug", true)` (keys mirror the params, `debug` = DebugMode).

## API Reference

```js
IntelliPatch.moveTo(character, x, y, options)
IntelliPatch.moveToPath(character, [[x,y],[x,y]], options)
IntelliPatch.follow(character, targetCharacter, distance, options)
IntelliPatch.wander(character, area, options)
IntelliPatch.cancel(character)      // stops Navigator only, native route survives
IntelliPatch.cancelAll(character)   // stops Navigator AND clears native route
IntelliPatch.addPatch(config)       // Lite: type cost|block (others rejected with warning)
IntelliPatch.removePatch(id)
IntelliPatch.getPath(character)
IntelliPatch.hasPath(character)
IntelliPatch.setGlobalConfig(key, value)
IntelliPatch.debug()
IntelliPatch.debugTile(x, y)
```

`options`:

```js
{
  recalcThreshold: 2,   // 0 treated as 1
  maxIterations: 2000,
  diagonal: false,
  avoidEvents: true,    // other events are avoided en route...
  allowTouch: true,     // ...destination contact ON by default (set false to stop adjacent)
  avoidPlayer: false,   // (fires Player Touch / Event Touch triggers on overlap)
  priority: "normal",   // low | normal | high (scheduler; FIFO within level)
  stuckPolicy: "recalc",// Lite: recalc | giveup
  blockedRetries: 3,
  heuristic: "manhattan",
  onArrive: function(){},
  onFail: function(){},
  onRecalc: function(newPath){},
  onBlocked: function(info){} // {policy, reason, attempts, target}
}
```

Semantics:

- An alternative road is ALWAYS tried first (up to `blockedRetries` recalcs). `recalc` keeps retrying (never gives up while the map can change); `giveup` stays still (`failed` + `onFail`).
- `allowTouch` steps onto the occupied destination tile (one-step collision bypass, `through` restored right after) so Player Touch / Event Touch triggers fire on overlap. En-route tiles are still avoided and patch blocks always apply, even on the target. ON by default (pass `false` to stop adjacent).
- `follow()` keeps approximately N tiles: stops when closer, resumes when farther. Not an exact orbit.
- `moveToPath()` waypoint rule: per waypoint try `findPath`, else closest-reachable fallback when allowed, else abort the whole path and call `onFail()`.
- `addPatch()` on ID collision silently overwrites (idempotent, no error). Non-cost/block types are rejected with a playtest warning.
- `getPath()` returns a copy with `.fallbackUsed`; `hasPath()` is true while steps remain.

## Plugin Commands

All take `eventId` with `0 = player, -1 = this event, N = map event N`.

- **MoveTo** (eventId, x, y, diagonal, avoidEvents, allowTouch, blockedPolicy, blockedRetries)
- **FollowTarget** (followerId, targetId, distance, allowTouch, blockedPolicy, blockedRetries)
- **WanderArea** (eventId, x1, y1, x2, y2, avoidEvents, allowTouch, blockedPolicy, blockedRetries)
- **ClearPath** (eventId) — same as `cancel()`, native route survives.
- **MoveToPath** (eventId, pathString `"5,5;10,10"`, diagonal, allowTouch, blockedPolicy, blockedRetries)
- **WaitForArrival** (eventId, timeoutFrames) — blocks the interpreter until idle/failed or timeout.
- **AddCostPatch** (patchId, x, y, w, h, multiplier) — rectangular cost zone.
- **AddBlockPatch** (patchId, x, y, w, h) — rectangular blocked zone.
- **RemovePatch** (patchId)
- **DebugToggle** (enabled)

## Patch Types

```js
// Mud: triple cost in a zone
IntelliPatch.addPatch({ id: "mud", type: "cost",
  area: { x: 3, y: 3, w: 5, h: 5 }, data: { multiplier: 3.0 } });

// Wind: +5 cost when travelling right (direction 6)
IntelliPatch.addPatch({ id: "wind", type: "cost",
  area: { x: 10, y: 5, w: 6, h: 6 }, data: { add: 5, direction: 6 } });

// Switch door: blocked while switch 5 is OFF
IntelliPatch.addPatch({ id: "door", type: "block",
  area: { x: 12, y: 8, w: 1, h: 1 },
  condition: function(){ return !$gameSwitches.value(5); } });
```

## Examples

1. NPC chases the player, avoiding events:
```js
IntelliPatch.moveTo(this, $gamePlayer.x, $gamePlayer.y, {
  avoidEvents: true,
  onArrive: function(){ $gameMessage.add("Caught!"); }
});
```

2. Patrol with a wind patch:
```js
IntelliPatch.addPatch({ id: "wind", type: "cost",
  area: { x: 10, y: 5, w: 6, h: 6 }, data: { add: 5, direction: 6 } });
IntelliPatch.wander(this, { x1: 8, y1: 4, x2: 16, y2: 10 });
```

3. Flee from the player:
```js
var dx = this.x - $gamePlayer.x, dy = this.y - $gamePlayer.y;
IntelliPatch.moveTo(this, this.x + dx * 3, this.y + dy * 3);
```

4. Door open only with switch 5 ON:
```js
IntelliPatch.addPatch({ id: "door", type: "block",
  area: { x: 12, y: 8, w: 1, h: 1 },
  condition: function(){ return !$gameSwitches.value(5); } });
IntelliPatch.moveTo(this, 15, 8);
```

5. Wander in mud x3:
```js
IntelliPatch.addPatch({ id: "mud", type: "cost",
  area: { x: 3, y: 3, w: 5, h: 5 }, data: { multiplier: 3.0 } });
IntelliPatch.wander(this, { x1: 1, y: 1, x2: 10, y2: 10 });
```

## What It Doesn't Do

- Does not modify player movement (events only by default; player works only when explicitly targeted).
- Tile-based only (no 3D, no continuous navmesh, 48x48 grid).
- Does not replace VisuStella Events and Movement Core; it coexists with it.
- Does not guarantee perfect paths on maps with no passable tiles defined (check tileset passability first).
- Lite has no Goal System, no force/portal/conditional patches, no origin/back_retry/random/ghost policies, no register* hooks, no steering — see Pro.

## Performance Tuning

- `safe` for 200+ active navigators on old hardware (5 paths/sec global, expect visible recalc delay).
- `balanced` as default for typical projects (token bucket from `PathsPerSecond`, 2000 iterations).
- `aggressive` for small maps with few navigators that need low-latency recalc (unthrottled, iterations capped at 500).

Further tips: shrink `MaxIterations` on huge open maps; raise `RecalcThreshold` for crowds; prefer `AddCostPatch` rects over per-tile conditions; keep `PathSmoothing` on.

## Troubleshooting

- **Event stops for no reason**: fixed in the engine — steps advance strictly on arrival, empty paths retry every 20 frames, cached roads are validated against live events. If it still holds, read the overlay `pol:attempts` label and raise `BlockedRetries` or switch the policy to `giveup` to fail visibly instead.
- **Event walks through walls**: check tileset passability (O/X) and `DiagonalMode`; use `auto` to stop corner-cutting.
- **Event never arrives**: call `IntelliPatch.debug()`; `failed` + red tile = unreachable (check `AllowUnreachableFallback` and `debugTile` output).
- **Pro patch rejected with warning**: force/portal/conditional types are Pro-only — Lite keeps them out by design.
- **No overlay**: `DebugMode` on, on Scene_Map (not menu/battle), spriteset present; click a tile to log patches.
- **Conflict**: confirm load order (VisuStella -> IntelliPatchLite -> CharacterBase-overriders); disable other pathfinders first; never install Lite + Pro together.

## Compatibility

- VisuStella Events and Movement Core: the Navigator ticks on `Game_CharacterBase.update` (never `Game_Event.update`) so VisuStella overrides cannot miss or double ticks.
- Any plugin overriding `Game_Character.prototype.canPass`: respected (Pathfinder prefers `character.canPass`, falls back to `$gameMap.isPassable`).
- Read-only map access: `$gameMap.isPassable`, `$gameMap.eventsXy`, `$gameMap.isValid`; no core files modified.
- Save/load: navigators live in memory only and are never serialized; `Game_Map.setup` cancels all navigators, clears the cache, and bumps the revision. Re-issue paths via autorun after load/transfer if needed.
- NW.js (MZ Chromium): no top-level await, no `??=` / `||=` / `&&=`; all engine aliases guarded.

## File Structure

Single file `IntelliPatchLite.js`:

```
CONSTANTS / UTIL / BINARY HEAP / PATCH (cost, block) / PATHFINDER
RECALC SCHEDULER / NAVIGATOR / GLOBAL API / HOOKS ENGINE / PLUGIN COMMANDS / DEBUG
```

## License/Credits

Free for commercial and non-commercial use. Redistribution of the plugin file is not allowed. Credit not required but appreciated. Copyright notice must remain intact.

Authors: Rpx & Just Dev. https://github.com/Rp-ics/RMMZ_IntelliPatch_Plugin
