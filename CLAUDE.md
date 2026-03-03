# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the games

No build step. Open any HTML file directly in the browser:

```bash
open shooter.html
open tic-tac-toe.html
```

There is no package manager, no bundler, and no dev server.

## Git workflow

Every meaningful change must be committed and pushed to GitHub (`eran1205/claude-code-intro`). Always push after committing — the user wants a saved version on GitHub at all times.

```bash
git add <files>
git commit -m "..."
git push
```

Never use `git add -A` or `git add .` — stage files explicitly by name.

## Architecture: shooter.html

Everything lives in one file (~1100 lines). Sections are delimited by `// ─── NAME ───` banners and appear in this order:

| Section | Purpose |
|---|---|
| `CONSTANTS` | Canvas size (`W=800, H=600`), `STATE` enum, `SCALE=2` |
| `CONTROLS CONFIG` | `DEFAULT_CONTROLS`, `ACTION_KEYS/LABELS` arrays, `keyLabel()` display helper |
| `LEVELS DATA` | `LEVELS[]` array — each entry has `name`, `hpBonus`, and `waves[][]` |
| `ENEMY CONFIG` | `ENEMY_CONFIG` object keyed by type (`basic/fast/tank/shooter`) |
| `HELPERS` | `aabbOverlap`, `circleOverlap`, `getEdgeSpawnPos`, `drawText` |
| `AUDIO MANAGER` | `AudioManager` — Web Audio API, lazy `AudioContext` init on first user gesture |
| `INPUT MANAGER` | `InputManager` — `keys` dict + `_pressed` set; `captureNextKey(cb)` for rebinding |
| `PARTICLE` | `Particle` class + `spawnExplosion()` factory |
| `BULLET` | `Bullet` class with tracer trail |
| `DRAW HELPERS` | `withTransform()`, `drawHealthPip()`, per-entity draw functions |
| `PLAYER` | `Player` class — movement, shooting, walk animation, iFrames |
| `ENEMY` | `Enemy` class — config-driven, moves toward player, shoots if `shootInterval > 0` |
| `GAME` | `Game` class — main loop, state machine, collision, wave/level management, all render methods |

### Game loop

Fixed ~60 fps via `requestAnimationFrame` + a `< 14ms` guard. No delta time — physics uses integer frame counts. Order each frame: `update()` → shake transform → `render()` → `input.flush()`.

### State machine

`MENU → PLAYING ⇄ PAUSED`, `PLAYING → LEVEL_COMPLETE → PLAYING`, `PLAYING → GAME_OVER → MENU`, `PLAYING → VICTORY → MENU`, `MENU ⇄ CONTROLS`.

State transitions always go through `setState(s)` which resets `stateTimer`. Each state has a matching `updateX()` and `renderX()` method on `Game`.

### Rendering pattern

All sprites are drawn with canvas 2D primitives at `SCALE=2` (no image assets). The standard pattern:

```js
withTransform(ctx, entity.x, entity.y, entity.angle, ctx => {
  ctx.scale(SCALE, SCALE);
  // draw in local space centred at (0,0)
});
```

`ctx.imageSmoothingEnabled = false` is set once at construction for crisp pixels.

### Adding a new enemy type

1. Add an entry to `ENEMY_CONFIG` with `hp`, `speed`, `size`, `color`, `scoreValue`, `shootInterval`, `contactDamage`, `bulletDamage`.
2. Add a `drawEnemyX(ctx, hp, maxHp)` function following the existing pattern (save → scale(SCALE,SCALE) → draw → drawHealthPip → restore).
3. Add a `case 'x':` in `Enemy.draw()`.
4. Reference the type string in the `LEVELS` wave definitions.

### Adding a new level

Append an entry to `LEVELS[]`. Each wave entry is `{type, count}`. Enemy types unlock implicitly — just include them in a wave.

### Controls system

`DEFAULT_CONTROLS` and `ACTION_KEYS[]`/`ACTION_LABELS[]` are kept in sync (same order). To add a new bindable action: add it to all three, pass it through `Player.update(input, controls)` or check it in `Game.updatePlaying()`. Bindings are saved to `localStorage` key `topShooterControls`.

### localStorage keys

| Key | Contents |
|---|---|
| `topShooterHS` | High score (integer) |
| `topShooterControls` | JSON object of current key bindings |
