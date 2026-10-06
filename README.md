# ISO SNAKE

**v3.10** — a neon isometric 3D snake game in a single HTML file.

Built with [Three.js](https://threejs.org): no build step, no assets, no dependencies to install. Everything (game, shaders, audio, UI) lives in one `index.html`.

**Default look — all four post-FX toggles off:**

![ISO SNAKE default look](docs/fx-off.png)

**All four FX toggles on (grade · grain · scanlines · Amiga look):**

![ISO SNAKE with every post-FX enabled](docs/fx-all.png)

## Features

- 🐍 Iridescent snake rendered with a custom shader injected via `onBeforeCompile` (head = one Mesh, body = one `InstancedMesh` of rounded cubes with a tapered tail: 2 draw calls)
- 🌆 Isometric neon world: animated floor grid, breathing border, purple obstacles, wrap-around borders
- 🌕 Space dome sky: gradient horizon glow, static star field, drifting nebula bands — stars ring the table and never appear behind the glass
- 🪞 True floor reflection of every table element (snake, food, power-ups, obstacles), clipped exactly to the board silhouette from any camera angle + a glowing ribbon trail behind the snake
- ✨ Arcade post-processing: neon bloom (half-res), chromatic aberration, vignette, CRT scanlines, glitch — all merged into a single FX pass
- 🎛 Opt-in retro FX toggles, off by default and enabled one by one (pill buttons bottom-left or keys `1`–`4`): color grade, film grain, CRT scanlines, and an authentic Amiga-era look (640×400 hi-res interlace + HAM 4096-color posterize)
- 🔊 Fully synthetic audio (WebAudio, no sound files)
- 📱 Touch support: swipe to steer, tap to start; desktop: arrow keys / WASD + orbit camera
- ⚡ Low input latency: smooth interpolated head turning, zero-allocation hot paths, three-level quality tiers that downgrade under strain and gently recover after ~10 s of healthy frames
- 🎊 Power-ups: 2× points, slow-mo, ghost, shrink — with HUD ring countdown (now an animated sweep arc)
- 🏆 Faster pacing: level up every 300 points (+100 each next level), local high-score saved on your device

## Controls

| Action | Desktop | Touch |
|---|---|---|
| Move | `↑↓←→` / `WASD` | swipe |
| Pause | `P` / `Esc` / ⏸ | — |
| Help | `H` / ? | — |
| Audio | `M` | — |
| FX toggles | `1` grade · `2` grain · `3` scanlines · `4` Amiga | pill buttons bottom-left |
| Start / confirm | `Enter` / click | tap |
| Orbit camera | drag | two fingers |
| Zoom | scroll wheel | pinch |

The in-game help screen (press `H`):

![ISO SNAKE help](docs/help.png)

## Download the game

The whole game is a single file — pick whatever suits you:

- **Direct download** (single file, ~73 KB):
  [`index.html`](https://github.com/ciskje/ISOSnake/raw/main/index.html) → save it and double-click it in any modern browser.
- **Whole repository as ZIP**:
  [`archive/refs/heads/main.zip`](https://github.com/ciskje/ISOSnake/archive/refs/heads/main.zip)
- **Git clone**:
  ```bash
  git clone https://github.com/ciskje/ISOSnake.git
  ```
- **Play instantly in the browser** (no download): [https://ciskje.github.io/ISOSnake/](https://ciskje.github.io/ISOSnake/)

> The file loads Three.js from a CDN, so you need an internet connection the first time you open it (everything else is self-contained).

## Run it locally

```bash
# just open the file
start index.html            # Windows
open index.html             # macOS
xdg-open index.html         # Linux

# or serve it
python -m http.server 8000
```

## Repository layout

```
index.html   ← the entire game
AGENTS.md    ← notes/instructions for AI agents working on this repo
README.md    ← this file
LICENSE      ← MIT
docs/        ← screenshots
```

## Licence

ISO SNAKE is released under the **MIT Licence**. See [`LICENSE`](LICENSE) for details.

Three.js is licensed under its own MIT licence and is loaded from its official CDN.
