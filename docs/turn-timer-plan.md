# Turn Timer & Win Celebration Plan

## Overview
Two new features for the WebGL tic-tac-toe game:

1. **Turn Timer** — Each player has 5 seconds to move. On timeout, the turn is skipped (no piece placed) and the other player goes next.
2. **Win Celebration** — Confetti particle burst using WebGL point sprites when a player wins. Particles spawn from the winning line center, colored by player (cyan for X, purple for O). Draw triggers a neutral gold burst from board center.

---

# Turn Timer

## HTML Changes

1. Add a timer bar element below `#status`:
   ```html
   <div id="timer-container"><div id="timer-bar"></div></div>
   ```

## CSS Changes

1. Style `#timer-container` as a dark rounded track (width matches game board, ~420px, height ~8px, dark background, rounded corners)
2. Style `#timer-bar` as the fill inside the track:
   - Rounded corners to match container
   - Starts at 100% width, shrinks to 0% as time depletes
   - Color transitions via HSL interpolation: green (hsl 120) → yellow (hsl 60) → red (hsl 0) based on remaining fraction

## JS State Changes

1. Add constant `TURN_TIME = 5` (seconds)
2. Add `let turnStart = 0` — timestamp when the current turn began. Initialized to `now` on first frame (see edge cases below).
3. Add `const timerBar = document.getElementById('timer-bar')` reference

## JS Logic — Render Loop (`render()`)

Each frame:
1. If `!gameOver`:
   - Calculate `elapsed = now - turnStart`
   - Calculate `fraction = Math.max(0, 1 - elapsed / TURN_TIME)`
   - Set `timerBar.style.width = fraction * 100 + '%'`
   - Set `timerBar.style.backgroundColor = 'hsl(' + (fraction * 120) + ', 100%, 50%)'`
   - If `elapsed >= TURN_TIME`:
     - Skip the current player's turn: flip `currentPlayer`
     - Reset `turnStart = now`
     - Update status text to show new player's turn
2. If `gameOver`: set `timerBar.style.width = '100%'` and `timerBar.style.backgroundColor` to a neutral color (e.g. `#333`)

## JS Logic — `handleClick()`

After switching to the next player (the `else` branch), add:
- `turnStart = now`

Use `now` from the render loop (already available; up to ~16ms accuracy is acceptable for a 5s timer).

## JS Logic — `restart()`

Add:
- `turnStart = now` (reset timer for fresh game)
- Reset timer bar display
- Remove `.winner-text` class from `statusEl`

## Edge Cases — Timer

- **First frame**: `turnStart` is initialized to `0`. Since `now` also starts near `0` on page load, `elapsed` will be near-0 on the very first frame, giving a full timer. To be safe, add an initialization guard: `if (turnStart === 0) turnStart = now;` at the top of the timer logic in `render()`.
- **Timer should freeze** visually when game is over.
- **Skip turn** should not place any piece — just flip whose turn it is and reset the timer.
- **Tab background**: If a tab is backgrounded and returns, `elapsed` could be very large, triggering one skip per frame until caught up. This is acceptable behavior — at most one skip per frame renders.

---

# Win Celebration

## JS State Changes

1. Add `let particles = []` — array of active particle objects
2. Add `let lastFrame = 0` — for computing `dt` in the render loop (needed for particle physics)

Each particle object:
```
{
  x, y,          // position in game-pixel coords (0–420)
  vx, vy,        // velocity in game-pixels/sec
  color: [r,g,b], // RGB 0–1
  life: 1.0,      // 1.0 → 0.0, removed when <= 0
  size,           // radius in CSS pixels (DPR handled in shader)
  rotation,       // for visual variety (future use)
  rotSpeed        // angular velocity (future use)
}
```

## JS Logic — Delta Time (`render()`)

At the top of `render()`, before any drawing:
```js
const dt = lastFrame === 0 ? 0 : Math.min(now - lastFrame, 0.1);
lastFrame = now;
```
The `Math.min(..., 0.1)` caps `dt` to 100ms to prevent huge jumps if the tab was backgrounded.

## JS Logic — Spawning (`handleClick()`)

When a win or draw is detected (where `gameOver = true` is set):
1. Determine spawn origin:
   - If win: average position of the three winning cells (`cellCenter(a) + cellCenter(b) + cellCenter(c)) / 3`). Note: current code destructures `winLine` as `[a, b]` — need to destructure all three as `[a, b, c]` for the spawn position.
   - If draw: board center (`SIZE/2, SIZE/2`)
2. Determine particle colors (pick randomly per particle from variations):
   - X wins: cyan `[0, 0.85, 1.0]`, `[0, 0.6, 1.0]`, `[0.2, 1.0, 1.0]`
   - O wins: purple `[0.45, 0.15, 1.0]`, `[0.6, 0.3, 1.0]`, `[0.3, 0.1, 0.8]`
   - Draw: gold `[1.0, 0.85, 0.3]`, `[1.0, 0.7, 0.2]`, `[1.0, 1.0, 0.6]`
3. Spawn ~80–120 particles in a burst:
   - Random angle (full 360°)
   - Random speed (30–150 game-pixels/sec) — decompose into `vx = speed * cos(angle)`, `vy = speed * sin(angle)`
   - Random size (3–8 CSS pixels)
   - `life = 1.0`
4. Set status text color to match winner:
   - X wins: `statusEl.style.color = '#00d9ff'`
   - O wins: `statusEl.style.color = '#7b2fff'`
   - Draw: `statusEl.style.color = '#ffd966'`

## WebGL — Particle Shader

Add two new shader programs using the existing `makeProgram()` helper and `getUniforms()`.

### Vertex Shader (`particleVS`)
```glsl
attribute vec2 a_pos;
attribute float a_size;
attribute float a_life;
attribute vec3 a_particleColor;
uniform vec2 u_res;
uniform float u_dpr;
varying float v_life;
varying vec3 v_color;
void main() {
  vec2 clip = vec2((a_pos.x / u_res.x) * 2.0 - 1.0, 1.0 - (a_pos.y / u_res.y) * 2.0);
  gl_Position = vec4(clip, 0.0, 1.0);
  gl_PointSize = a_size * a_life * u_dpr;
  v_life = a_life;
  v_color = a_particleColor;
}
```

The `u_dpr` uniform multiplies `gl_PointSize` by `devicePixelRatio` so particles render at the correct visual size on high-DPI screens (matching how the canvas is already scaled by DPR).

The `u_res` uniform should be set to `[SIZE, SIZE]` (i.e. `[420, 420]`) since particle positions are in game-pixel coords, consistent with the existing `toClip()` convention.

### Fragment Shader (`particleFS`)
```glsl
precision mediump float;
varying float v_life;
varying vec3 v_color;
void main() {
  vec2 coord = gl_PointCoord - vec2(0.5);
  float dist = length(coord);
  if (dist > 0.5) discard;
  float alpha = smoothstep(0.5, 0.2, dist) * v_life;
  gl_FragColor = vec4(v_color, alpha);
}
```

This renders each particle as a soft glowing circle that fades as life decreases.

### Buffers

Create 4 dynamic buffers (one per attribute) with `gl.DYNAMIC_DRAW`. Each frame, build `Float32Array`s from the `particles` array and upload via `gl.bufferData`. Use interleaved or separate buffers — separate is simpler given the existing codebase pattern.

Attributes and their locations:
- `a_pos` — vec2, particle x/y position
- `a_size` — float, particle size
- `a_life` — float, current life (0–1)
- `a_particleColor` — vec3, RGB

Uniforms:
- `u_res` — vec2, `[SIZE, SIZE]` = `[420, 420]`
- `u_dpr` — float, `devicePixelRatio`

## JS Logic — Particle Update (`render()`)

After drawing pieces and win line, before `requestAnimationFrame`:
1. Update each particle:
   - `x += vx * dt`
   - `y += vy * dt`
   - `vy += 200 * dt` (gravity pulling down at 200 px/s²)
   - `life -= dt * 0.4` (particles fade over ~2.5 seconds)
   - `rotation += rotSpeed * dt`
2. Remove particles where `life <= 0`: `particles = particles.filter(p => p.life > 0)`
3. If `particles.length > 0`:
   - Build typed arrays: positions (Float32Array, 2 per particle), sizes (Float32Array, 1 per particle), lives (Float32Array, 1 per particle), colors (Float32Array, 3 per particle)
   - Upload to their respective buffers
   - Bind the particle program, set uniforms, enable/point all four attributes, draw with `gl.drawArrays(gl.POINTS, 0, particles.length)`

## JS Logic — `restart()`

Add:
- `particles = []`
- `statusEl.style.color = ''` (reset to CSS default)

## CSS Changes — Status Text Animation

1. Add a CSS class `.winner-text` with a pulsing glow animation:
   ```css
   .winner-text {
     animation: winPulse 0.6s ease-in-out infinite alternate;
   }
   @keyframes winPulse {
     from { text-shadow: 0 0 5px currentColor; transform: scale(1); }
     to   { text-shadow: 0 0 20px currentColor, 0 0 40px currentColor; transform: scale(1.08); }
   }
   ```
2. In `handleClick()`, when setting `gameOver = true`, add `statusEl.classList.add('winner-text')`
3. In `restart()`, remove: `statusEl.classList.remove('winner-text')`

Since we also set `statusEl.style.color` dynamically on win (cyan/purple/gold), the `currentColor` in the text-shadow will inherit that color, making the glow match the winner.

## Edge Cases — Celebration

- **Restart mid-celebration**: `particles = []` clears everything immediately.
- **Draw celebration** uses the same particle system with different colors and spawn point (board center).
- `gl.enable(gl.BLEND)` is already set in the existing code, so particle alpha blending works without changes.
- Particles are drawn on `game-canvas` (on top of pieces, same layer) — this is fine since they should overlay the board.
- **Multiple spawn prevention**: The `if (gameOver) return` guard at the top of `handleClick()` prevents re-triggering a celebration on additional clicks.
- **`dt` first frame**: `dt = 0` on the first frame (since `lastFrame = 0`), so particles won't jump on initialization.
- **gl_PointSize limit**: Most GPUs support at least 64px. At DPR 2, max particle size would be 8 * 1.0 * 2 = 16 device pixels — well within limits.