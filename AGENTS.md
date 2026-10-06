# AGENTS.md — ISO SNAKE

Instructions for agents working on this project. Learned during the optimization/translation sessions (v2.0 → v2.1).

## The project

- Single-file Three.js game: `index.html` (~1270 lines). No build step, no dependencies beyond the three.js CDN (importmap + `type="module"`).
- Grid snake (20×20), isometric 3D, neon arcade look, post-processing (bloom + custom FXPass), synthetic WebAudio, touch + keyboard + OrbitControls.
- Git repo at the project root; commit messages in the repo's style (short, descriptive).

## Version convention (IMPORTANT)

Version `X.Y` must stay in sync in **three places**:
1. `<title>ISO SNAKE v2.1</title>` (line ~6)
2. menu subtitle: `neon isometric snake · 3D · v2.1` (line ~105)
3. script comment: `/* version: 2.1 — bump X (major) ... */` (line ~189)

- **X** = major: structure, refactor, gameplay (e.g. the whole optimization + InstancedMesh work: 1.x → 2.0).
- **Y** = minor: tweaks (pulse direction/speed, translation to English: 2.0 → 2.1).
- Bump on every change, then commit.

## File map (line anchors drift; grep the section banners)

Sections are delimited by `/* ===== name ===== */` banners:
- `config` (~192): `CFG` — `baseSpeed: 7` (→ ~143 ms tick), `levelPoints: 500`, `grid: 20`, `powerupChance: 0.035` (per tick).
- `state` (~211): `G` object (includes `quality`, `fpsAcc/fpsN/slowWindows`, `particleScale`).
- `three` (~228): renderer (`antialias:false`, pixelRatio clamp `isTouch ? 1 : 1.5`, `shadowMap.enabled = false`), camera, lights (no shadow config), composer + bloom (half res) + FXPass (tonemapping/colorspace folded in, no OutputPass), resize handler (~433, calls `bloom.setSize(w/2, h/2)`).
- `mesh & materials` (~445): iridescent snake via `onBeforeCompile`; `makeSnakeMat` (head, `uIdx` uniform, cache key `isosnake-iris-v1`) and `makeSnakeMatInst` (body, `aIdx` instanced attribute, cache key `isosnake-iris-inst-v1` — keys MUST differ). Snake = `headMesh` (Mesh + eyes) + `bodyMesh` (InstancedMesh, capacity `MAX_SNAKE = grid*grid`, `frustumCulled=false`).
- `regles` (~730): `tick()` (~796) — grid step; `key(x,y) = (x<<5)|y` numeric; `prevSnake` reuses the old array (no per-point copies).
- `input` (~1012): `queueDir` (~1016) uses shared `DIR` objects + instant head rotation; `keydown` uses `e.code` (`ArrowUp`/`KeyW`…), `if (e.repeat) return`.
- `loop` (~1101): `updateVisuals` (interpolation + instanced matrices via scratch `_m4`), `frame()` (~1177) with `dt = Math.min(0.05, …)` catch-up cap, powerup spawn roll inside the tick loop, adaptive quality block.
- `avvio` (~1238): `sfx.ensure()` once at startup (not in hot paths); `tone()` resumes a suspended ctx.

## Semantics to preserve (do NOT "optimize away")

- Tick-based grid movement: no mid-cell turning; input only queues direction (max 2 queued).
- `dt = Math.min(0.05, dt)` catch-up cap.
- `G.prevSnake[i]` = previous position of segment i; grow case → `prevSnake` shorter, `updateVisuals` falls back with `|| cur`.
- Power-up chance is **per tick**, not per frame.

## Critical pitfall: accented characters

The `edit` tool often FAILS on strings containing accented/special chars (à è ì · — → ⏸ ×) — the file bytes don't match what the read/grep output appears to show. Workaround that works reliably:

1. Dump the exact strings from the file with a node script (`readFileSync(p,'utf8')`, regex to collect comments, write to a temp txt).
2. Read the dumped strings, then replace **by index** with `s.split(exact).join(replacement)` in a node script.
3. Pure-ASCII strings are fine for the `edit` tool.

Temp scripts: put them in `C:\Users\Principale\AppData\Local\Temp\opencode\` (approved). Use `.mjs` with `import { readFileSync, writeFileSync } from 'fs'` (ESM — `require` fails).

## Verification recipe

Syntax-check the module script after every edit:

```powershell
$src = Get-Content -Raw "Z:\Lavorazioni\2026\AI\Isosnake\index.html"
if ($src -match '(?s)<script type="module">(.*?)</script>') {
  Set-Content -Path "$env:TEMP\opencode\isosnake-check.mjs" -Value $1 -Encoding UTF8
  node --check "$env:TEMP\opencode\isosnake-check.mjs"
}
```

Also grep for leftover Italian words/accents after translation work.

## Shell notes

- PowerShell: `&&` is NOT a valid separator — use `;`.
- `git status --short; git log -3 --oneline` works; LF→CRLF warnings are harmless.

## History of decisions (why the code looks like it does)

- v2.0 latency/frame-time work: `e.repeat` filter, `e.code` matching, instant head rotation toward queued dir, shadows fully off, bloom half-res, OutputPass merged into FXPass, pixelRatio clamp, adaptive quality (`composer.setPixelRatio(0.75)` on sustained slow windows), numeric `key()`, zero-alloc `tick()`/`freeCells()` (reused `_occ` Set + `_freeOut` array), scratch `_v`/`_v2`/`_m4`, swipe detected in `touchmove` (>24 px, `touchDirDone` flag) with `touchend` fallback.
- v2.0 also: luminance pulse reversed tail→head and sped to 1.6 Hz; InstancedMesh body (2 draw calls for the snake).
- v2.1: everything translated to English (UI + comments).
- v3.0 (P0): faster level change — threshold is now `CFG.levelPoints + (level-1)*100` with `levelPoints: 300` (was `level * 500`); obstacle ramp softened to `min(6, level+1)`. Speed values untouched.
- v3.1 (P1): snake silhouette — `RoundedBoxGeometry` (radius 0.12, 2 segments) for head/body, per-instance tail taper `1 - 0.25*(i/n)` composed into the instance matrix (zero alloc; diagonal written directly into `_m4.elements` because r160 `Matrix4.scale` takes a Vector3, not x/y/z — passing numbers produced NaN matrices and an invisible body), eyes moved to z 0.42 with bright emissive (they were buried inside the head box). **Head yaw override**: the deliberate v2.0 instant snap is replaced by a smooth shortest-arc lerp (`1 - exp(-dt/0.04)`, ~120 ms) in `updateVisuals`; `queueDir` no longer touches `rotation.y`. Shader cache keys unchanged (`-v1`).
- v3.2 (P2): ribbon trail + fake reflection — ribbon = one indexed `BufferGeometry` (2 verts x MAX_SNAKE, additive, `depthWrite:false`, `frustumCulled:false`, vertex-color fade to tail, zero-width pair breaks the strip at wrap jumps, drawRange = (n-1)*6); reflection = TRUE mirror: floor is now semi-transparent (`transparent:true, opacity:0.82`) and the flipped copy lives BELOW it in a group (`scale.y=-1, y=-0.1`, additive opacity 0.3, `DoubleSide` because negative scale flips winding, `depthWrite:false`, `fog:false`); mirror body reuses the exact body instance matrices (group transform does the flip). Natural transparent sort draws mirror (farther) before the floor, so the floor's alpha composites over it — but that sort is position-dependent, so `renderOrder = -1` is FORCED on both mirror meshes: near-side instances (e.g. the head on the camera side) would otherwise sort after the floor and be depth-culled against it (the disappearing head reflection). No coplanar faces possible (mirror is below the floor). `setSnakeLength` updates both body counts. **All table elements reflect**: food (shared octahedron geo, synced pos/scale/rot in `frame()`), power icon (same geo, color-matched additive mirror mat, created/removed in `setPowerType`), obstacles (per-cell mirror box in an `obstMirrors` Map, static, removed in `setObstacles`). Every mirror mesh carries `renderOrder = -1`. Draw calls up to ~28 with 6 obstacles (guideline exceeded, meshes are tiny).
- v3.3 (P3): space dome — inverted `SphereGeometry` r=140 (`side:BackSide`, `depthWrite:false`, `frustumCulled:false`, unlit `ShaderMaterial`): vertical gradient, `floor(d*170)` hash star twinkle (time-driven), 2 nebula bands with a slow swirl, horizon glow hue ~240 matched to the fog. **Decision: dome radius 140 must stay > `maxDistance` 90 and < `far` 200** or the camera clips inside it. `scene.background` kept as fallback. `uTime` fed from `updateVisuals`. Draw calls +1. **Tune**: a top-down camera only sees sky near the horizon, so nebula bands sit LOW (h 0.10/0.26), stars are round dots (`d*300` cells, `fract` disc, threshold 0.986, brightness 1.4 so they catch bloom) with **no horizon mask** — the top-down play camera sees the LOWER dome around the table, so stars must cover the whole sphere, gradient/horizon glow brightened — the first values were invisible after ACES tonemapping. **Later tune**: stars are STATIC (twinkle removed, user disliked pulsing) and masked `smoothstep(-0.35,-0.15,h)` so they stop under the table (steep under-glass view is h < -0.4); mirror materials are now OPAQUE dim colors (additive washed out against the bloom) — all mirror meshes need `DoubleSide` because the flip group inverts winding. **Clip to board**: `renderer.localClippingEnabled = true` + 4 `clippingPlanes` on every mirror material. First attempt was a static x/z box at +/-grid/2 — NOT enough: from a low camera the copy under the near edge projects OUTSIDE the table silhouette even inside the footprint (the floating power-up ghost). Final fix: `_mirrorDefs` (edge point + direction per side) and `updateMirrorClip()` rebuilds each plane through the board edge AND the current camera position every frame (`frame()`, after `controls.update()`), so mirrors are hard-clipped exactly to the floor quad's screen silhouette. **Later guard (v3.4 tune)**: `updateMirrorClip()` skips a plane when the cross product degenerates (camera nearly coplanar with an edge plane -> unstable orientation sign) and biases each plane constant inward by 0.06 so grazing/ultra-low cameras cannot leak borderline spill slivers past the table rim. **Stars use the same geometry trick but in the dome shader**: `uCamPos` uniform + vertex varying `vWP = position` (dome centered at origin). For fragments with `vWP.y < -0.05`: march along cam→P to the y = 0 crossing (`s = -camY/(P.y-camY)`), hide the star if that hit is inside the board footprint (`vis = smoothstep(half-0.6, half+0.6, max(|q.x|,|q.z|))`). Result: stars ring the table at ground level AND fill the sky, none behind the glass (`vWP` now uses `modelMatrix * position` = true world coords). **Sky follows camera**: `dome.position.copy(camera.position)` right before `composer.render()` — eating/power-up bumps are pure camera translations, so pinning the dome to the eye keeps stars/nebula perfectly still during shake while the table still jumps (and no parallax drift when orbiting). Dome r=140 > maxDistance 90 + board diagonal ⇒ a camera-centered dome can never clip scene geometry. **Event FX mask**: FXPass gained `uTable` = padded uv-AABB of the four projected board corners (`_fxCN`, recomputed every frame in `frame()` right before `composer.render()`, after shake) and glitch + chromatic aberration are multiplied by a soft inside-rect mask — eating/power-up bumps fringe and tear only near the table, sky/stars stay pristine. **Pitfall**: first attempt did this with `vDir = normalize(position)` — WRONG BASIS (direction from dome center, not eye ray) → killed all stars; must interpolate world fragment position and intersect explicitly. **GLSL gotchas that silently blacked the whole dome** (three.js logs Shader Error to console, render just shows nothing): never read `.z` off a `vec2` (`q.x`/`q.y` only) and never write `${half} - 0.6` into GLSL — bare integer literal minus float is illegal in GLSL ES (precompute in JS: `${half - 0.6}` emits `9.4`). Verified empirically by rendering headless Chrome screenshots of an instrumented copy (banner captures window errors + console.error).
- **P4 (floor v2) REJECTED**: scan sweep / circuit traces / reticle / distance fade implemented, user disliked it on first look - reverted same day (`git revert`), do not reintroduce without explicit request; version numbering skips P4 so P5 lands on v3.4.
- v3.4 (P5): entity juice. Food: additive halo billboard = `THREE.Sprite` + 64px canvas radial-gradient `CanvasTexture` (auto-faces camera, no quaternion sync needed), pulse-synced with octahedron scale, plus throttled idle sparkles reusing the existing particle system (`_sparkAcc > 0.3 s`, count scaled by `G.particleScale`). Power countdown: `powerRingMat` is now a custom ShaderMaterial with `uArc` uniform (angle fraction from top CW via `mod(atan(vP.x,vP.y),2pi)/2pi`, AA via `fwidth(fracA)`), faint ghost track outside the arc — replaces the old opacity hack; `setPowerType` sets `uniforms.uColor`. Obstacles -> InstancedMesh trio capacity `MAX_OBST=64`: rounded box body + shared matrix to flipped mirror IM (same pattern as bodyMesh/mirrorBody, opaque dim mat + `_boardClip` shared planes + `renderOrder=-1`) + flat base ring IM above floor (no mirror for glow FX); emissive pulse via `matObst.emissiveIntensity` per frame (uniform update, no recompile). Draw calls DOWN ~9 net (was 6 boxes + 6 mirrors -> 3 instanced draws + 1 sprite). Harness after edit: shaderErrors [], jsErrors [] (bannerRed metric inflated by the new magenta halo, not an error).
