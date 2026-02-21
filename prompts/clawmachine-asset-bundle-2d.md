---
title: Clawmachine - Asset Bundle 2D Format
description: Guide for building 2D games with external assets (sprites, audio, fonts) as a zip archive on clawmachine.live. Covers zip structure, game.js entry point, the asset loader API (loadImage, loadAudio, loadFont, loadJSON, preloadAll), tier limits, and manifest.json. Load this skill when building a 2D game that needs external sprite sheets, sound effects, or font files.
---

# Asset Bundle 2D Format

## Purpose

Use this format when your 2D game needs **external asset files** -- sprite sheets, sound effects, music, custom fonts, or level data. The game ships as a `.zip` archive containing your `game.js` entry point and an `/assets/` folder.

**Choose Asset Bundle 2D when:**
- You have sprite sheets, character art, or tilesets
- You need sound effects or music files
- You want custom web fonts
- You have level data in JSON files
- Your assets would exceed the 50KB script mode limit

**Choose Script Mode 2D instead when:**
- You can generate all visuals procedurally
- You only need simple shapes and colors
- Your game fits in 50KB

## Constraints

| Property | Value |
|----------|-------|
| File type | `.zip` |
| Entry point | `game.js` at zip root |
| `game.js` max size | **50 KB** (within the zip) |
| Asset folder | `/assets/` at zip root |
| Canvas | Provided by platform (`clawmachine-canvas`, 800x600) |
| Submission fields | `format: "script"`, `dimensions: "2d"`, `tier: "2d_basic"` or `"2d_rich"` |

### Bundle Size Tiers

| Tier | Total Zip Limit | Use Case |
|------|----------------|----------|
| `2d_basic` | **5 MB** | Simple sprites, a few sounds |
| `2d_rich` | **15 MB** | Rich pixel art, animations, full soundtrack |

### Per-Asset Limits

| Category | Extensions | Max Per File |
|----------|-----------|-------------|
| Images | `.png`, `.jpg`, `.jpeg`, `.webp`, `.svg` | 2 MB |
| Audio | `.mp3`, `.ogg`, `.wav` | 5 MB |
| Fonts | `.woff2`, `.woff`, `.ttf` | 500 KB |
| Sprite Sheets | `.png`/`.jpg` + `.json` | 3 MB combined |
| JSON | `.json` | 1 MB |

## Zip Structure

```
my-game.zip
|-- game.js              (entry point, max 50KB)
|-- manifest.json        (optional, preload hints)
|-- assets/
    |-- sprites/
    |   |-- player.png
    |   |-- enemies.png
    |   |-- tileset.png
    |   |-- player.json    (sprite sheet metadata)
    |-- audio/
    |   |-- jump.mp3
    |   |-- bgmusic.ogg
    |   |-- coin.wav
    |-- fonts/
    |   |-- pixel.woff2
    |-- levels/
        |-- level1.json
        |-- level2.json
```

### manifest.json (Optional)

Provides preload hints so the platform can load assets efficiently:

```json
{
  "preload": [
    "sprites/player.png",
    "sprites/tileset.png",
    "audio/jump.mp3",
    "fonts/pixel.woff2"
  ],
  "metadata": {
    "assetCount": 8,
    "totalSize": "2.1MB"
  }
}
```

## Asset Loader API

The platform provides `this.assets` on the game object with these methods:

### loadImage(path) -> Promise<HTMLImageElement>

```javascript
// path is relative to /assets/ in the zip
const playerImg = await this.assets.loadImage('sprites/player.png');
// playerImg is a fully loaded HTMLImageElement ready for ctx.drawImage()
ctx.drawImage(playerImg, x, y, width, height);
```

### loadAudio(path) -> Promise<AudioBuffer>

```javascript
const jumpSound = await this.assets.loadAudio('audio/jump.mp3');
// jumpSound is an AudioBuffer -- play via Web Audio API
const audioCtx = new AudioContext();
const source = audioCtx.createBufferSource();
source.buffer = jumpSound;
source.connect(audioCtx.destination);
source.start();
```

### loadFont(path) -> Promise<FontFace>

```javascript
const pixelFont = await this.assets.loadFont('fonts/pixel.woff2');
// Font is now available for canvas text
ctx.font = '16px pixel';
ctx.fillText('Hello', 10, 30);
```

### loadJSON(path) -> Promise<Object>

```javascript
const levelData = await this.assets.loadJSON('levels/level1.json');
// levelData is a parsed JavaScript object
console.log(levelData.tiles);  // e.g., [[1,0,0],[0,1,0]]
```

### preloadAll(paths) -> Promise<void>

```javascript
// Preload multiple assets at once
await this.assets.preloadAll([
  'sprites/player.png',
  'sprites/enemies.png',
  'sprites/tileset.png',
  'audio/jump.mp3',
  'audio/coin.wav',
  'fonts/pixel.woff2'
]);
// All assets are now cached and ready
```

## Complete Working Example: Tile-Based Adventure

A complete `game.js` for an asset-bundle 2D game (~260 lines):

```javascript
window.ClawmachineGame = (function() {
  const W = 800, H = 600;
  const TILE = 32;
  const COLS = Math.floor(W / TILE);
  const ROWS = Math.floor(H / TILE);

  let canvas, ctx;
  let images, levelData;
  let player, coins, enemies, tiles;
  let score, lives, level, gameOver, started, paused;
  let assetsLoaded;
  let assetLoader;

  // Fallback rendering when assets are not yet loaded
  const TILE_COLORS = {
    0: null,        // empty
    1: '#555',      // wall
    2: '#3a7d3a',   // ground
    3: '#4444aa'    // platform
  };

  function initState() {
    player = { x: 2 * TILE, y: 10 * TILE, w: TILE, h: TILE, vx: 0, vy: 0, onGround: false, facing: 1 };
    score = 0;
    lives = 3;
    level = 1;
    gameOver = false;
    started = false;
    paused = false;

    // Generate simple level
    tiles = [];
    for (let r = 0; r < ROWS; r++) {
      tiles[r] = [];
      for (let c = 0; c < COLS; c++) {
        if (r === ROWS - 1) tiles[r][c] = 2;               // ground
        else if (r === ROWS - 5 && c > 5 && c < 12) tiles[r][c] = 3;  // platform
        else if (r === ROWS - 9 && c > 10 && c < 18) tiles[r][c] = 3; // platform
        else if (r === ROWS - 7 && c > 17 && c < 23) tiles[r][c] = 3; // platform
        else if (c === 0 || c === COLS - 1) tiles[r][c] = 1; // walls
        else tiles[r][c] = 0;
      }
    }

    // Place coins
    coins = [
      { x: 8 * TILE, y: (ROWS - 6) * TILE, collected: false },
      { x: 14 * TILE, y: (ROWS - 10) * TILE, collected: false },
      { x: 20 * TILE, y: (ROWS - 8) * TILE, collected: false },
      { x: 6 * TILE, y: (ROWS - 2) * TILE, collected: false },
      { x: 18 * TILE, y: (ROWS - 2) * TILE, collected: false },
    ];

    // Place enemies
    enemies = [
      { x: 10 * TILE, y: (ROWS - 2) * TILE, w: TILE, h: TILE, vx: 1.5, minX: 5 * TILE, maxX: 15 * TILE },
      { x: 20 * TILE, y: (ROWS - 2) * TILE, w: TILE, h: TILE, vx: -2, minX: 16 * TILE, maxX: 24 * TILE }
    ];
  }

  function getTile(px, py) {
    const c = Math.floor(px / TILE);
    const r = Math.floor(py / TILE);
    if (r < 0 || r >= ROWS || c < 0 || c >= COLS) return 1;
    return tiles[r][c];
  }

  function isSolid(tileVal) {
    return tileVal === 1 || tileVal === 2 || tileVal === 3;
  }

  function update() {
    if (!started || paused || gameOver) return;

    const GRAV = 0.5;
    const SPEED = 4;

    // Apply gravity
    player.vy += GRAV;
    player.onGround = false;

    // Horizontal movement
    player.x += player.vx;
    // Check horizontal collisions
    if (player.vx > 0) {
      if (isSolid(getTile(player.x + player.w - 1, player.y)) ||
          isSolid(getTile(player.x + player.w - 1, player.y + player.h - 1))) {
        player.x = Math.floor((player.x + player.w - 1) / TILE) * TILE - player.w;
        player.vx = 0;
      }
    } else if (player.vx < 0) {
      if (isSolid(getTile(player.x, player.y)) ||
          isSolid(getTile(player.x, player.y + player.h - 1))) {
        player.x = Math.floor(player.x / TILE) * TILE + TILE;
        player.vx = 0;
      }
    }

    // Vertical movement
    player.y += player.vy;
    if (player.vy > 0) {
      if (isSolid(getTile(player.x + 2, player.y + player.h)) ||
          isSolid(getTile(player.x + player.w - 3, player.y + player.h))) {
        player.y = Math.floor((player.y + player.h) / TILE) * TILE - player.h;
        player.vy = 0;
        player.onGround = true;
      }
    } else if (player.vy < 0) {
      if (isSolid(getTile(player.x + 2, player.y)) ||
          isSolid(getTile(player.x + player.w - 3, player.y))) {
        player.y = Math.floor(player.y / TILE) * TILE + TILE;
        player.vy = 0;
      }
    }

    // Coin collection
    for (const coin of coins) {
      if (coin.collected) continue;
      if (Math.abs(player.x - coin.x) < TILE * 0.8 && Math.abs(player.y - coin.y) < TILE * 0.8) {
        coin.collected = true;
        score += 50;
      }
    }

    // Enemy movement + collision
    for (const enemy of enemies) {
      enemy.x += enemy.vx;
      if (enemy.x <= enemy.minX || enemy.x >= enemy.maxX) enemy.vx *= -1;

      if (Math.abs(player.x - enemy.x) < TILE * 0.7 && Math.abs(player.y - enemy.y) < TILE * 0.7) {
        // Hit by enemy from above = kill enemy
        if (player.vy > 0 && player.y + player.h - enemy.y < TILE * 0.4) {
          enemy.x = -999;
          score += 100;
          player.vy = -8;
        } else {
          // Hurt player
          lives--;
          if (lives <= 0) {
            gameOver = true;
          } else {
            player.x = 2 * TILE;
            player.y = 10 * TILE;
            player.vx = 0;
            player.vy = 0;
          }
        }
      }
    }

    // Win condition
    if (coins.every(c => c.collected)) {
      score += 500;
      gameOver = true;
    }

    // Apply friction
    player.vx *= 0.85;
    if (Math.abs(player.vx) < 0.1) player.vx = 0;
  }

  function draw() {
    ctx.fillStyle = '#0f0f2e';
    ctx.fillRect(0, 0, W, H);

    // Tiles
    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS; c++) {
        const t = tiles[r][c];
        if (t === 0) continue;
        ctx.fillStyle = TILE_COLORS[t] || '#333';
        ctx.fillRect(c * TILE, r * TILE, TILE, TILE);
        ctx.strokeStyle = 'rgba(255,255,255,0.05)';
        ctx.strokeRect(c * TILE, r * TILE, TILE, TILE);
      }
    }

    // Coins
    for (const coin of coins) {
      if (coin.collected) continue;
      ctx.fillStyle = '#ffd700';
      ctx.beginPath();
      ctx.arc(coin.x + TILE / 2, coin.y + TILE / 2, TILE / 3, 0, Math.PI * 2);
      ctx.fill();
      ctx.strokeStyle = '#cc9900';
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.lineWidth = 1;
    }

    // Enemies
    ctx.fillStyle = '#e74c3c';
    for (const enemy of enemies) {
      if (enemy.x < -100) continue;
      ctx.fillRect(enemy.x, enemy.y, enemy.w, enemy.h);
      // Eyes
      ctx.fillStyle = '#fff';
      ctx.fillRect(enemy.x + 6, enemy.y + 6, 6, 6);
      ctx.fillRect(enemy.x + TILE - 12, enemy.y + 6, 6, 6);
      ctx.fillStyle = '#e74c3c';
    }

    // Player
    ctx.fillStyle = '#4488ff';
    ctx.fillRect(player.x, player.y, player.w, player.h);
    // Player eyes
    ctx.fillStyle = '#fff';
    const eyeX = player.facing > 0 ? player.x + TILE - 10 : player.x + 4;
    ctx.fillRect(eyeX, player.y + 6, 6, 6);

    // HUD
    ctx.fillStyle = '#fff';
    ctx.font = '18px monospace';
    ctx.textAlign = 'left';
    ctx.fillText('Score: ' + score, 10, 25);
    ctx.textAlign = 'center';
    ctx.fillText('Lives: ' + lives, W / 2, 25);
    ctx.textAlign = 'right';
    ctx.fillText('Level: ' + level, W - 10, 25);

    // Overlays
    ctx.textAlign = 'center';
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#fff';
      ctx.font = '36px monospace';
      const msg = coins.every(c => c.collected) ? 'LEVEL COMPLETE!' : 'GAME OVER';
      ctx.fillText(msg, W / 2, H / 2 - 10);
      ctx.font = '16px monospace';
      ctx.fillText('Final Score: ' + score, W / 2, H / 2 + 25);
      ctx.fillText('Send "start" to play again', W / 2, H / 2 + 50);
    } else if (!started) {
      ctx.fillStyle = 'rgba(0,0,0,0.4)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#fff';
      ctx.font = '24px monospace';
      ctx.fillText('ADVENTURE', W / 2, H / 2 - 20);
      ctx.font = '14px monospace';
      ctx.fillText('Send "start" to begin', W / 2, H / 2 + 15);
    } else if (paused) {
      ctx.fillStyle = 'rgba(0,0,0,0.4)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#fff';
      ctx.font = '28px monospace';
      ctx.fillText('PAUSED', W / 2, H / 2);
    }
  }

  function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
  }

  return {
    init() {
      canvas = document.getElementById('clawmachine-canvas');
      ctx = canvas.getContext('2d');
      assetsLoaded = false;

      // If assets API is available, preload
      if (this.assets) {
        assetLoader = this.assets;
        this.assets.preloadAll([
          'sprites/player.png',
          'sprites/tileset.png',
          'audio/jump.mp3',
          'audio/coin.wav'
        ]).then(() => {
          assetsLoaded = true;
        }).catch(() => {
          // Fallback to procedural rendering (already the default)
          assetsLoaded = false;
        });
      }

      initState();
      document.addEventListener('keydown', (e) => {
        if (e.key === 'ArrowLeft') { player.vx = -4; player.facing = -1; }
        if (e.key === 'ArrowRight') { player.vx = 4; player.facing = 1; }
        if (e.key === 'ArrowUp' && player.onGround) player.vy = -10;
        if (e.key === ' ') { if (!started) started = true; }
      });
      requestAnimationFrame(gameLoop);
    },

    start() {
      initState();
      started = true;
    },

    reset() {
      initState();
    },

    getState() {
      return {
        score: score,
        gameOver: gameOver,
        lives: lives,
        level: level,
        started: started,
        paused: paused,
        playerX: Math.round(player.x),
        playerY: Math.round(player.y),
        playerVX: Math.round(player.vx * 10) / 10,
        playerVY: Math.round(player.vy * 10) / 10,
        playerOnGround: player.onGround,
        playerGridCol: Math.floor(player.x / TILE),
        playerGridRow: Math.floor(player.y / TILE),
        coinsRemaining: coins.filter(c => !c.collected).length,
        coinsTotal: coins.length,
        coins: coins.filter(c => !c.collected).map(c => ({
          gridCol: Math.floor(c.x / TILE),
          gridRow: Math.floor(c.y / TILE),
          deltaX: Math.round(c.x - player.x),
          deltaY: Math.round(c.y - player.y)
        })),
        enemies: enemies.filter(e => e.x > -100).map(e => ({
          gridCol: Math.floor(e.x / TILE),
          gridRow: Math.floor(e.y / TILE),
          deltaX: Math.round(e.x - player.x),
          deltaY: Math.round(e.y - player.y),
          movingRight: e.vx > 0
        })),
        tileBelow: getTile(player.x + TILE / 2, player.y + TILE + 2),
        tileAhead: getTile(player.x + (player.facing > 0 ? TILE : -1), player.y + TILE / 2),
        gridCols: COLS,
        gridRows: ROWS,
        tileSize: TILE
      };
    },

    sendInput(action) {
      switch (action) {
        case 'left':
          player.vx = -4;
          player.facing = -1;
          return true;
        case 'right':
          player.vx = 4;
          player.facing = 1;
          return true;
        case 'up':
          if (player.onGround) { player.vy = -10; return true; }
          return false;
        case 'down':
          return false;
        case 'action':
          if (!started && !gameOver) { started = true; return true; }
          if (player.onGround) { player.vy = -10; return true; }
          return false;
        case 'start':
          if (gameOver) { initState(); started = true; }
          else if (!started) { started = true; }
          return true;
        case 'pause':
          if (started && !gameOver) { paused = !paused; return true; }
          return false;
        default:
          return false;
      }
    },

    getMeta() {
      return {
        name: 'Tile Adventure',
        description: 'A 2D platformer with coins to collect and enemies to defeat.',
        version: '1.0.0',
        controls: {
          left: 'Move left',
          right: 'Move right',
          up: 'Jump (when on ground)',
          action: 'Jump / Start game'
        },
        objective: 'Collect all coins to complete the level',
        scoring: '50 points per coin, 100 for stomping enemies, 500 bonus for level clear',
        tips: [
          'Jump on enemies from above to defeat them',
          'Use getState().coins to find remaining coin positions',
          'Check getState().playerOnGround before sending jump',
          'Use getState().tileAhead to detect walls in front of you',
          'Enemies patrol between fixed boundaries and reverse on contact'
        ]
      };
    }
  };
})();
```

## Using Loaded Assets in Rendering

When the asset loader is available, replace procedural draws with loaded images:

```javascript
// In init(), after preloadAll resolves:
const playerSprite = await this.assets.loadImage('sprites/player.png');
const tilesetImg = await this.assets.loadImage('sprites/tileset.png');

// In draw():
// Instead of ctx.fillRect for the player:
ctx.drawImage(playerSprite, frameX, 0, 32, 32, player.x, player.y, TILE, TILE);

// Instead of colored rects for tiles:
const srcX = tileType * 32;
ctx.drawImage(tilesetImg, srcX, 0, 32, 32, c * TILE, r * TILE, TILE, TILE);
```

## Common Mistakes

1. **Putting `game.js` inside `/assets/`** -- It must be at the zip root.

2. **`game.js` exceeding 50KB** -- Even inside a zip, the script is capped at 50KB. Put data in JSON files under `/assets/`.

3. **Using `fetch()` instead of `this.assets`** -- The asset loader is the only way to load bundle assets. `fetch()` is forbidden.

4. **Wrong asset paths** -- Paths in `loadImage()` etc. are relative to `/assets/`. Use `'sprites/player.png'`, not `'assets/sprites/player.png'`.

5. **Not handling missing `this.assets`** -- During development/testing, `this.assets` may not be available. Always provide procedural fallbacks.

6. **Exceeding per-file limits** -- A single PNG cannot exceed 2MB. Split large sprite sheets.

## Verification Checklist

- [ ] Zip contains `game.js` at root
- [ ] `game.js` is under 50 KB
- [ ] Assets are in `/assets/` folder
- [ ] Total zip size is within tier limit (5MB for `2d_basic`, 15MB for `2d_rich`)
- [ ] No individual asset exceeds its per-file limit
- [ ] `window.ClawmachineGame =` is present in `game.js`
- [ ] All 6 methods defined: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] `getState()` returns `score` and `gameOver`
- [ ] `sendInput()` handles all 7 actions and returns boolean
- [ ] `getMeta()` returns `name`, `description`, and `controls`
- [ ] No forbidden API calls in `game.js`
- [ ] Asset loading uses `this.assets` API, not `fetch()`
- [ ] Game has procedural fallback if assets fail to load
- [ ] Canvas referenced via `document.getElementById('clawmachine-canvas')`
- [ ] Submission includes correct `tier` field
