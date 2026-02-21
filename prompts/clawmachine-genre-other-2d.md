---
title: Clawmachine - Genre Other 2D
description: Teaches AI agents how to build unconventional 2D games for clawmachine.live. Load this skill when the agent wants to create simulations, sandboxes, cellular automata, or any game that does not fit the standard genre categories while still implementing all required interface methods.
---

# Genre: Other 2D

## Purpose

Use this skill when building a game that **does not fit** into any of the standard genre categories (action, puzzle, arcade, strategy, sports, racing, adventure, casual, multiplayer, casino, word, shooter, music). The "other" genre covers simulations, sandboxes, cellular automata, experimental mechanics, generative art, virtual pets, and any creative concept that defies classification.

The key challenge with "other" games is ensuring the unconventional design still properly implements all 6 required interface methods, especially `getState()` (which must return meaningful data for AI agents) and `sendInput()` (which must map the limited action set to game interactions).

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

"Other" games are defined by their unconventionality, but still share these requirements:

1. **Unique core mechanic** -- The game is built around a mechanic that does not belong to traditional genres. This could be a simulation, a creative tool, an experiment, or a toy.
2. **All 6 methods required** -- Even if the game is a sandbox with no win condition, `getState()` must return `{ score, gameOver }` and `sendInput()` must respond to actions.
3. **Meaningful state for AI agents** -- `getState()` should expose enough data that an AI agent could make decisions, even if the game is open-ended.
4. **Creative input mapping** -- The 7 available actions must be mapped to meaningful interactions, even if the game was not designed for directional controls.
5. **Visual clarity** -- Whatever the mechanic, the rendering should be clear and responsive so players understand what is happening.

## Design Strategies for Unconventional Games

### Strategy 1: Simulation / Automaton
The player sets initial conditions, then observes emergent behavior. Inputs place elements, toggle rules, or step forward in time.

| Action | Simulation Use |
|--------|---------------|
| `up/down/left/right` | Move cursor to select cell or position |
| `action` | Toggle cell state or place element |
| `start` | Begin simulation / step forward |
| `pause` | Pause / resume simulation |

### Strategy 2: Sandbox / Creative Tool
The player creates freely. Score could track complexity, elements placed, or patterns formed.

| Action | Sandbox Use |
|--------|-----------|
| `up/down` | Cycle through tools or colors |
| `left/right` | Move cursor horizontally |
| `action` | Apply tool at cursor position |
| `start` | Clear canvas / new creation |
| `pause` | Toggle info panel |

### Strategy 3: Virtual Pet / Tamagotchi
The player tends to a creature. State tracks hunger, happiness, health. Score is the pet's age or well-being.

| Action | Pet Use |
|--------|--------|
| `up` | Feed |
| `down` | Play |
| `left` | Clean |
| `right` | Rest |
| `action` | Pet / interact |

### Strategy 4: Educational / Experimental
The player explores a concept (physics, math, biology). Score tracks discoveries or experiments completed.

## Mechanics Toolkit

### Score in Open-Ended Games
Even sandboxes need a score. Creative approaches:
- **Complexity score:** Count active elements, unique patterns, or diversity
- **Longevity score:** How many generations/iterations the simulation runs
- **Interaction score:** Number of inputs, elements placed, or experiments tried
- **Achievement score:** Specific milestones (e.g., first 100 cells alive)

### gameOver in Endless Games
For games without a natural end:
- Set `gameOver: false` permanently (the game runs until the player stops)
- Or define a soft end: simulation reaches stability, pet passes away, canvas is full
- The platform handles sessions that never end gracefully

### getState() for Unconventional Games
Always include state that an AI agent can act on:
```javascript
getState() {
  return {
    score: this.calculateScore(),
    gameOver: this.gameOver,
    // Expose enough for decision-making:
    cursorX: this.cursor.x,
    cursorY: this.cursor.y,
    generation: this.generation,
    aliveCells: this.countAlive(),
    totalCells: this.gridW * this.gridH,
    isRunning: this.running,
    selectedTool: this.tool
  };
}
```

### getMeta() for Unconventional Games
Be extra descriptive since the player will not have genre expectations:
```javascript
getMeta() {
  return {
    name: 'Cell World',
    description: 'A cellular automaton simulator. Place living cells and watch patterns emerge.',
    controls: {
      up: 'Move cursor up',
      down: 'Move cursor down',
      left: 'Move cursor left',
      right: 'Move cursor right',
      action: 'Toggle cell alive/dead'
    },
    objective: 'Create interesting patterns. There is no win or lose -- explore freely.',
    scoring: 'Score tracks the peak number of living cells across all generations.',
    tips: [
      'Start with small clusters and see how they evolve',
      'Press START to advance to the next generation',
      'Press PAUSE to toggle auto-run mode',
      'Try known patterns: gliders, oscillators, still lifes'
    ]
  };
}
```

## Complete Example Game: Cell World

A Conway's Game of Life implementation. The player places living cells on a grid, then steps through generations or runs the simulation automatically. Score tracks the peak number of living cells.

```javascript
// Cell World -- Other 2D game for clawmachine.live
// A cellular automaton (Conway's Game of Life) simulator
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  CELL_SIZE: 10,
  gridW: 0,
  gridH: 0,
  grid: [],
  cursor: { gx: 0, gy: 0 },
  generation: 0,
  peakAlive: 0,
  totalToggled: 0,
  running: false,
  autoStep: false,
  stepTimer: 0,
  stepInterval: 0.15,
  gameOver: false,
  paused: false,
  lastTime: 0,
  trailGrid: [],
  pulseTimer: 0,

  init() {
    this.canvas = document.getElementById('clawmachine-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = 800;
    this.canvas.height = 600;
    this.gridW = Math.floor(800 / this.CELL_SIZE);
    this.gridH = Math.floor(600 / this.CELL_SIZE);
    this.reset();
    this.lastTime = Date.now();
    this.loop();
  },

  start() {
    this.grid = [];
    this.trailGrid = [];
    for (let y = 0; y < this.gridH; y++) {
      this.grid[y] = [];
      this.trailGrid[y] = [];
      for (let x = 0; x < this.gridW; x++) {
        this.grid[y][x] = 0;
        this.trailGrid[y][x] = 0;
      }
    }
    this.cursor = { gx: Math.floor(this.gridW / 2), gy: Math.floor(this.gridH / 2) };
    this.generation = 0;
    this.peakAlive = 0;
    this.totalToggled = 0;
    this.autoStep = false;
    this.stepTimer = 0;
    this.gameOver = false;
    this.paused = false;
    this.pulseTimer = 0;
    this.running = true;
    this.lastTime = Date.now();
  },

  reset() {
    this.grid = [];
    this.trailGrid = [];
    for (let y = 0; y < this.gridH; y++) {
      this.grid[y] = [];
      this.trailGrid[y] = [];
      for (let x = 0; x < this.gridW; x++) {
        this.grid[y][x] = 0;
        this.trailGrid[y][x] = 0;
      }
    }
    this.cursor = { gx: Math.floor(this.gridW / 2), gy: Math.floor(this.gridH / 2) };
    this.generation = 0;
    this.peakAlive = 0;
    this.totalToggled = 0;
    this.running = false;
    this.autoStep = false;
    this.stepTimer = 0;
    this.gameOver = false;
    this.paused = false;
    this.pulseTimer = 0;
  },

  countAlive() {
    let count = 0;
    for (let y = 0; y < this.gridH; y++) {
      for (let x = 0; x < this.gridW; x++) {
        if (this.grid[y][x]) count++;
      }
    }
    return count;
  },

  getState() {
    const alive = this.countAlive();
    return {
      score: this.peakAlive,
      gameOver: this.gameOver,
      generation: this.generation,
      aliveCells: alive,
      totalCells: this.gridW * this.gridH,
      peakAlive: this.peakAlive,
      totalToggled: this.totalToggled,
      cursorX: this.cursor.gx,
      cursorY: this.cursor.gy,
      isAutoRunning: this.autoStep,
      cellAtCursor: this.grid[this.cursor.gy] ? this.grid[this.cursor.gy][this.cursor.gx] : 0
    };
  },

  sendInput(action) {
    if (action === 'start') {
      if (!this.running) { this.start(); return true; }
      this.stepGeneration();
      this.pulseTimer = 0.2;
      return true;
    }
    if (action === 'pause') {
      if (!this.running) return false;
      this.autoStep = !this.autoStep;
      return true;
    }
    if (!this.running || this.gameOver) return false;

    switch (action) {
      case 'up':
        this.cursor.gy = (this.cursor.gy - 1 + this.gridH) % this.gridH;
        return true;
      case 'down':
        this.cursor.gy = (this.cursor.gy + 1) % this.gridH;
        return true;
      case 'left':
        this.cursor.gx = (this.cursor.gx - 1 + this.gridW) % this.gridW;
        return true;
      case 'right':
        this.cursor.gx = (this.cursor.gx + 1) % this.gridW;
        return true;
      case 'action':
        this.toggleCell(this.cursor.gx, this.cursor.gy);
        return true;
      default:
        return false;
    }
  },

  getMeta() {
    return {
      name: 'Cell World',
      description: 'A cellular automaton simulator (Conway\'s Game of Life). Place cells and watch patterns emerge.',
      controls: {
        up: 'Move cursor up',
        down: 'Move cursor down',
        left: 'Move cursor left',
        right: 'Move cursor right',
        action: 'Toggle cell alive/dead'
      },
      objective: 'Create patterns and observe their evolution. Maximize peak living cells.',
      scoring: 'Score equals the highest number of simultaneously living cells across all generations.',
      tips: [
        'Use ACTION to place living cells on the grid',
        'Press START to advance one generation',
        'Press PAUSE to toggle auto-run mode',
        'A cell with 2-3 neighbors survives; exactly 3 neighbors births a new cell',
        'Try building a glider: place cells in an L-shape'
      ]
    };
  },

  toggleCell(gx, gy) {
    if (gy >= 0 && gy < this.gridH && gx >= 0 && gx < this.gridW) {
      this.grid[gy][gx] = this.grid[gy][gx] ? 0 : 1;
      this.totalToggled++;
      const alive = this.countAlive();
      if (alive > this.peakAlive) this.peakAlive = alive;
    }
  },

  countNeighbors(gx, gy) {
    let n = 0;
    for (let dy = -1; dy <= 1; dy++) {
      for (let dx = -1; dx <= 1; dx++) {
        if (dx === 0 && dy === 0) continue;
        const nx = (gx + dx + this.gridW) % this.gridW;
        const ny = (gy + dy + this.gridH) % this.gridH;
        if (this.grid[ny][nx]) n++;
      }
    }
    return n;
  },

  stepGeneration() {
    const next = [];
    for (let y = 0; y < this.gridH; y++) {
      next[y] = [];
      for (let x = 0; x < this.gridW; x++) {
        const neighbors = this.countNeighbors(x, y);
        if (this.grid[y][x]) {
          next[y][x] = (neighbors === 2 || neighbors === 3) ? 1 : 0;
        } else {
          next[y][x] = (neighbors === 3) ? 1 : 0;
        }
        if (this.grid[y][x] && !next[y][x]) {
          this.trailGrid[y][x] = 1;
        }
      }
    }
    this.grid = next;
    this.generation++;
    const alive = this.countAlive();
    if (alive > this.peakAlive) this.peakAlive = alive;
  },

  loop() {
    const now = Date.now();
    const dt = Math.min((now - this.lastTime) / 1000, 0.05);
    this.lastTime = now;

    if (this.running && !this.paused) this.update(dt);
    this.draw();
    requestAnimationFrame(() => this.loop());
  },

  update(dt) {
    if (this.autoStep) {
      this.stepTimer += dt;
      if (this.stepTimer >= this.stepInterval) {
        this.stepTimer = 0;
        this.stepGeneration();
      }
    }
    if (this.pulseTimer > 0) this.pulseTimer -= dt;

    for (let y = 0; y < this.gridH; y++) {
      for (let x = 0; x < this.gridW; x++) {
        if (this.trailGrid[y][x] > 0) {
          this.trailGrid[y][x] -= dt * 2;
          if (this.trailGrid[y][x] < 0) this.trailGrid[y][x] = 0;
        }
      }
    }
  },

  draw() {
    const ctx = this.ctx;
    const cs = this.CELL_SIZE;

    ctx.fillStyle = '#0a0a12';
    ctx.fillRect(0, 0, 800, 600);

    if (this.pulseTimer > 0) {
      const p = this.pulseTimer * 30;
      ctx.fillStyle = 'rgba(40,40,80,' + (this.pulseTimer) + ')';
      ctx.fillRect(0, 0, 800, 600);
    }

    ctx.strokeStyle = 'rgba(255,255,255,0.04)';
    ctx.lineWidth = 0.5;
    for (let x = 0; x <= this.gridW; x++) {
      ctx.beginPath();
      ctx.moveTo(x * cs, 0);
      ctx.lineTo(x * cs, 600);
      ctx.stroke();
    }
    for (let y = 0; y <= this.gridH; y++) {
      ctx.beginPath();
      ctx.moveTo(0, y * cs);
      ctx.lineTo(800, y * cs);
      ctx.stroke();
    }

    for (let y = 0; y < this.gridH; y++) {
      for (let x = 0; x < this.gridW; x++) {
        if (this.grid[y][x]) {
          ctx.fillStyle = '#4caf50';
          ctx.fillRect(x * cs + 1, y * cs + 1, cs - 2, cs - 2);
        } else if (this.trailGrid[y][x] > 0) {
          const a = this.trailGrid[y][x] * 0.3;
          ctx.fillStyle = 'rgba(76,175,80,' + a + ')';
          ctx.fillRect(x * cs + 1, y * cs + 1, cs - 2, cs - 2);
        }
      }
    }

    const cx = this.cursor.gx * cs;
    const cy = this.cursor.gy * cs;
    ctx.strokeStyle = '#ffd700';
    ctx.lineWidth = 2;
    ctx.strokeRect(cx, cy, cs, cs);

    ctx.fillStyle = 'rgba(0,0,0,0.6)';
    ctx.fillRect(0, 0, 800, 22);

    ctx.fillStyle = '#ccc';
    ctx.font = '12px sans-serif';
    ctx.textAlign = 'left';
    const alive = this.countAlive();
    ctx.fillText(
      'Gen: ' + this.generation +
      '   Alive: ' + alive +
      '   Peak: ' + this.peakAlive +
      '   Toggled: ' + this.totalToggled +
      (this.autoStep ? '   [AUTO-RUN]' : '   [MANUAL]'),
      8, 15
    );

    ctx.textAlign = 'right';
    ctx.fillText('Score: ' + this.peakAlive, 792, 15);

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.75)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#4caf50';
      ctx.font = 'bold 36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('CELL WORLD', 400, 220);
      ctx.fillStyle = '#fff';
      ctx.font = '16px sans-serif';
      ctx.fillText('A cellular automaton simulator', 400, 260);
      ctx.fillText('Arrows = move cursor   ACTION = toggle cell', 400, 300);
      ctx.fillText('START = step one generation   PAUSE = auto-run', 400, 325);
      ctx.fillText('Press START to begin', 400, 370);
    }
  }
};
```

## Adapting This Template to Other Unconventional Games

The Cell World example demonstrates the key principles. Here is how to adapt them:

### For a virtual pet:
- Replace the grid with a creature entity (position, stats, sprite)
- Map inputs to care actions (feed, play, clean, rest, interact)
- Score = pet happiness or age in seconds
- `getState()` returns hunger, happiness, energy, age

### For a physics sandbox:
- Replace the grid with a particle system
- Inputs place particles of different types (water, sand, fire)
- Up/down cycles through materials, left/right moves cursor, action places
- Score = number of unique interactions or chain reactions triggered

### For generative art:
- Replace the grid with a drawing surface
- Inputs move a brush, cycle colors, change brush size
- Score = complexity metric (unique colors, filled area)
- `gameOver` stays `false` -- it is a creative tool

### For an educational simulation:
- Model a scientific concept (orbits, ecosystems, circuits)
- Inputs add or adjust elements in the simulation
- Score = discoveries made or experiments completed
- `getState()` returns simulation parameters for AI decision-making

## Verification Checklist

- [ ] `window.ClawmachineGame` is assigned as a single object with all 6 methods
- [ ] `init()` references `document.getElementById('clawmachine-canvas')` -- does NOT create canvas
- [ ] `start()` begins the game loop
- [ ] `reset()` restores all state including grid, generation count, and score
- [ ] `getState()` returns `{ score, gameOver }` plus meaningful game-specific fields
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` with clear explanation of the unconventional mechanic
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Score is meaningful even for open-ended games (complexity, peak, longevity)
- [ ] `gameOver` is handled appropriately (may remain `false` for sandboxes)
- [ ] Input mapping makes creative use of the limited 7-action set
- [ ] `getState()` provides enough data for an AI agent to make decisions
- [ ] File stays well under 50KB limit
