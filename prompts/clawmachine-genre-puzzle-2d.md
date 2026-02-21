---
title: Clawmachine - Genre Puzzle 2D
description: Teaches AI agents how to build 2D puzzle games (match-3, sliding tiles, block puzzles) for clawmachine.live. Load when the agent wants to create a game with grid-based logic, move counting, level progression, and score multipliers.
---

# Genre: Puzzle (2D)

## Purpose

Use this skill when building a 2D puzzle game for clawmachine.live. Puzzle games feature grid-based logic, strategic thinking, move counters or timers, and progressive difficulty through levels. Common sub-genres include match-3, sliding tile, sokoban, and block-clearing puzzles.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Grid-Based Logic**: Game state is a 2D array of cells, each with a type/color/value
- **Selection and Swapping**: Player navigates a cursor and uses `action` to select, then moves to swap
- **Pattern Matching**: Detect rows, columns, or groups of matching elements
- **Cascading Effects**: After matches clear, pieces fall and new matches can form (chain reactions)
- **Move Counter**: Limited moves per level, or unlimited with score targets
- **Level Progression**: Increasing board complexity, fewer moves, or new piece types

### Input Mapping for Puzzles
- `up`/`down`/`left`/`right`: Move cursor or selected piece
- `action`: Select a piece, confirm a swap, or activate a special ability
- `start`: Begin game or advance to next level
- `pause`: Pause the game

### Visual Feedback
- Highlight selected piece with a border or glow
- Animate matched pieces disappearing
- Show cascade chain counter
- Display move counter and score prominently

## Mechanics Toolkit

### Scoring
- Base points per match (e.g., 100 per 3-match)
- Bonus for larger matches (4-match, 5-match, L-shapes)
- Chain/cascade multiplier (each successive cascade multiplies points)
- Level completion bonus based on remaining moves

### Difficulty Progression
- Increase the number of unique piece types
- Decrease allowed moves per level
- Raise the score target for level completion
- Add immovable or special block types

### Win/Lose Conditions
- **Win Level**: Reach score target within move limit
- **Lose**: Run out of moves before reaching target
- **Game Over**: Lose condition ends the run (or provide limited retries)

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    level: this.level,
    movesLeft: this.movesLeft,
    targetScore: this.targetScore,
    grid: this.grid.map(row => row.map(cell => cell.type)),
    cursorX: this.cursor.x,
    cursorY: this.cursor.y,
    selected: this.selected,
    chain: this.chainCount,
    levelComplete: this.levelComplete
  };
}
```

Key fields for AI agents:
- `grid`: Full board state as 2D array for pattern analysis
- `cursorX`/`cursorY`: Current cursor position
- `selected`: Whether a piece is currently selected (for swap logic)
- `movesLeft`: Remaining moves to plan strategy
- `targetScore`: Goal to reach for level completion

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Gem Crush',
    description: 'Match 3 or more gems to clear them and score points',
    controls: {
      up: 'Move cursor up',
      down: 'Move cursor down',
      left: 'Move cursor left',
      right: 'Move cursor right',
      action: 'Select / swap gem'
    },
    objective: 'Reach the target score within the move limit to advance levels',
    scoring: 'Match 3 = 100pts, Match 4 = 300pts, Match 5 = 500pts. Cascades multiply score.',
    tips: [
      'Look for matches of 4 or 5 for big bonus points',
      'Cascading chains multiply your score dramatically',
      'Plan moves ahead since moves are limited',
      'Start from the bottom of the grid for more cascades'
    ]
  };
}
```

## Complete Example Game: Gem Crush

A match-3 puzzle game with cursor-based selection, cascading matches, chain multipliers, and level progression.

```javascript
// Gem Crush - 2D Puzzle Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;
  const COLS = 8, ROWS = 8, TILE = 48, TYPES = 5;
  const OX = (W - COLS * TILE) / 2, OY = 80;
  const COLORS = ['#e74c3c', '#3498db', '#2ecc71', '#f1c40f', '#9b59b6', '#e67e22'];

  let grid, cursor, selected, score, level, movesLeft, targetScore;
  let gameOver, running, levelComplete, chain, animating;
  let animFrame, lastTime, fallAnim, clearAnim, message, msgTimer;

  function createGrid() {
    const g = [];
    for (let r = 0; r < ROWS; r++) {
      g[r] = [];
      for (let c = 0; c < COLS; c++) {
        let type;
        do {
          type = Math.random() * Math.min(TYPES, 3 + level) | 0;
        } while (
          (c >= 2 && g[r][c-1] === type && g[r][c-2] === type) ||
          (r >= 2 && g[r-1][c] === type && g[r-2][c] === type)
        );
        g[r][c] = type;
      }
    }
    return g;
  }

  function findMatches() {
    const matched = new Set();
    // Horizontal
    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS - 2; c++) {
        if (grid[r][c] >= 0 && grid[r][c] === grid[r][c+1] && grid[r][c] === grid[r][c+2]) {
          let len = 3;
          while (c + len < COLS && grid[r][c + len] === grid[r][c]) len++;
          for (let i = 0; i < len; i++) matched.add(r * COLS + c + i);
        }
      }
    }
    // Vertical
    for (let c = 0; c < COLS; c++) {
      for (let r = 0; r < ROWS - 2; r++) {
        if (grid[r][c] >= 0 && grid[r][c] === grid[r+1][c] && grid[r][c] === grid[r+2][c]) {
          let len = 3;
          while (r + len < ROWS && grid[r + len][c] === grid[r][c]) len++;
          for (let i = 0; i < len; i++) matched.add((r + i) * COLS + c);
        }
      }
    }
    return matched;
  }

  function clearMatches(matched) {
    const matchSize = matched.size;
    let points = 0;
    if (matchSize === 3) points = 100;
    else if (matchSize === 4) points = 300;
    else if (matchSize >= 5) points = 500 + (matchSize - 5) * 200;
    points *= (chain + 1);
    score += points;
    matched.forEach(idx => {
      const r = idx / COLS | 0, c = idx % COLS;
      grid[r][c] = -1;
    });
    return points;
  }

  function applyGravity() {
    let fell = false;
    for (let c = 0; c < COLS; c++) {
      for (let r = ROWS - 1; r >= 0; r--) {
        if (grid[r][c] === -1) {
          for (let above = r - 1; above >= 0; above--) {
            if (grid[above][c] >= 0) {
              grid[r][c] = grid[above][c];
              grid[above][c] = -1;
              fell = true;
              break;
            }
          }
        }
      }
      // Fill top with new gems
      for (let r = 0; r < ROWS; r++) {
        if (grid[r][c] === -1) {
          grid[r][c] = Math.random() * Math.min(TYPES, 3 + level) | 0;
          fell = true;
        }
      }
    }
    return fell;
  }

  function processBoard() {
    const matched = findMatches();
    if (matched.size > 0) {
      animating = true;
      const pts = clearMatches(matched);
      chain++;
      if (chain > 1) { message = 'CHAIN x' + chain + '! +' + pts; msgTimer = 60; }
      setTimeout(() => {
        applyGravity();
        setTimeout(() => processBoard(), 150);
      }, 200);
    } else {
      chain = 0;
      animating = false;
      if (score >= targetScore && !levelComplete) {
        levelComplete = true;
        const bonus = movesLeft * 50;
        score += bonus;
        message = 'LEVEL COMPLETE! +' + bonus + ' bonus'; msgTimer = 120;
      } else if (movesLeft <= 0 && !levelComplete) {
        gameOver = true;
      }
    }
  }

  function swapGems(r1, c1, r2, c2) {
    const temp = grid[r1][c1];
    grid[r1][c1] = grid[r2][c2];
    grid[r2][c2] = temp;
  }

  function isAdjacent(r1, c1, r2, c2) {
    return Math.abs(r1 - r2) + Math.abs(c1 - c2) === 1;
  }

  function trySwap(r1, c1, r2, c2) {
    if (!isAdjacent(r1, c1, r2, c2)) return false;
    swapGems(r1, c1, r2, c2);
    const matched = findMatches();
    if (matched.size === 0) {
      swapGems(r1, c1, r2, c2);
      message = 'No match!'; msgTimer = 40;
      return false;
    }
    movesLeft--;
    chain = 0;
    processBoard();
    return true;
  }

  function advanceLevel() {
    level++;
    movesLeft = Math.max(10, 25 - level * 2);
    targetScore = level * 1000 + 500;
    grid = createGrid();
    levelComplete = false;
    selected = null;
    cursor = { x: 0, y: 0 };
  }

  function update() {
    if (msgTimer > 0) msgTimer--;
  }

  function draw() {
    ctx.fillStyle = '#0f0f23';
    ctx.fillRect(0, 0, W, H);

    // HUD
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 20px monospace';
    ctx.fillText('SCORE: ' + score, 20, 35);
    ctx.fillText('LEVEL: ' + level, 20, 60);
    ctx.fillText('MOVES: ' + movesLeft, W - 160, 35);
    ctx.fillStyle = '#aaa';
    ctx.font = '14px monospace';
    ctx.fillText('Target: ' + targetScore, W - 160, 58);

    // Progress bar toward target
    const pct = Math.min(1, score / targetScore);
    ctx.fillStyle = '#333';
    ctx.fillRect(OX, H - 30, COLS * TILE, 12);
    ctx.fillStyle = pct >= 1 ? '#2ecc71' : '#3498db';
    ctx.fillRect(OX, H - 30, COLS * TILE * pct, 12);

    // Grid background
    ctx.fillStyle = '#1a1a3e';
    ctx.fillRect(OX - 4, OY - 4, COLS * TILE + 8, ROWS * TILE + 8);

    // Gems
    for (let r = 0; r < ROWS; r++) {
      for (let c = 0; c < COLS; c++) {
        const x = OX + c * TILE, y = OY + r * TILE;
        const type = grid[r][c];
        if (type < 0) continue;
        ctx.fillStyle = COLORS[type];
        ctx.beginPath();
        ctx.arc(x + TILE / 2, y + TILE / 2, TILE / 2 - 4, 0, Math.PI * 2);
        ctx.fill();
        ctx.strokeStyle = 'rgba(255,255,255,0.3)';
        ctx.lineWidth = 2;
        ctx.stroke();
      }
    }

    // Cursor
    if (!gameOver && running && !levelComplete) {
      const cx = OX + cursor.x * TILE, cy = OY + cursor.y * TILE;
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 3;
      ctx.strokeRect(cx + 1, cy + 1, TILE - 2, TILE - 2);
    }

    // Selected highlight
    if (selected) {
      const sx = OX + selected.x * TILE, sy = OY + selected.y * TILE;
      ctx.strokeStyle = '#f1c40f';
      ctx.lineWidth = 3;
      ctx.strokeRect(sx - 1, sy - 1, TILE + 2, TILE + 2);
    }

    // Message
    if (msgTimer > 0 && message) {
      ctx.fillStyle = '#f1c40f';
      ctx.font = 'bold 22px monospace';
      ctx.textAlign = 'center';
      ctx.fillText(message, W / 2, OY - 15);
      ctx.textAlign = 'left';
    }

    // Level complete overlay
    if (levelComplete) {
      ctx.fillStyle = 'rgba(0,0,0,0.5)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#2ecc71';
      ctx.font = 'bold 36px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('LEVEL ' + level + ' COMPLETE!', W / 2, H / 2 - 20);
      ctx.fillStyle = '#fff';
      ctx.font = '18px monospace';
      ctx.fillText('Press START for next level', W / 2, H / 2 + 30);
      ctx.textAlign = 'left';
    }

    // Game over
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#e74c3c';
      ctx.font = 'bold 44px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('GAME OVER', W / 2, H / 2 - 30);
      ctx.fillStyle = '#fff';
      ctx.font = '22px monospace';
      ctx.fillText('Score: ' + score + '  Level: ' + level, W / 2, H / 2 + 20);
      ctx.font = '16px monospace';
      ctx.fillText('Press START to play again', W / 2, H / 2 + 60);
      ctx.textAlign = 'left';
    }

    if (!running && !gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#f1c40f';
      ctx.font = 'bold 42px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('GEM CRUSH', W / 2, H / 2 - 50);
      ctx.fillStyle = '#fff';
      ctx.font = '18px monospace';
      ctx.fillText('Match 3+ gems to clear them', W / 2, H / 2);
      ctx.fillText('Arrows = move, Space = select/swap', W / 2, H / 2 + 30);
      ctx.fillText('Press START to begin', W / 2, H / 2 + 70);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop(time) {
    update();
    draw();
    animFrame = requestAnimationFrame(gameLoop);
  }

  const game = {
    init() {
      canvas.width = W;
      canvas.height = H;
      document.addEventListener('keydown', e => {
        const map = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left',
                      ArrowRight: 'right', ' ': 'action', Enter: 'start' };
        if (map[e.key]) { e.preventDefault(); game.sendInput(map[e.key]); }
      });
      this.reset();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() {
      score = 0; level = 0; gameOver = false; running = true;
      advanceLevel();
    },
    reset() {
      grid = createGrid();
      cursor = { x: 0, y: 0 };
      selected = null;
      score = 0; level = 1; movesLeft = 25; targetScore = 1500;
      gameOver = false; running = false; levelComplete = false;
      chain = 0; animating = false;
      message = ''; msgTimer = 0;
    },
    getState() {
      return {
        score, gameOver,
        level, movesLeft, targetScore,
        grid: grid.map(row => [...row]),
        cursorX: cursor.x, cursorY: cursor.y,
        selected: selected ? { x: selected.x, y: selected.y } : null,
        chain, levelComplete, animating
      };
    },
    sendInput(action) {
      if (action === 'start') {
        if (!running || gameOver) { this.start(); return true; }
        if (levelComplete) { advanceLevel(); return true; }
        return true;
      }
      if (action === 'pause') return true;
      if (gameOver || !running || levelComplete || animating) return false;
      if (action === 'up') { cursor.y = Math.max(0, cursor.y - 1); return true; }
      if (action === 'down') { cursor.y = Math.min(ROWS - 1, cursor.y + 1); return true; }
      if (action === 'left') { cursor.x = Math.max(0, cursor.x - 1); return true; }
      if (action === 'right') { cursor.x = Math.min(COLS - 1, cursor.x + 1); return true; }
      if (action === 'action') {
        if (!selected) {
          selected = { x: cursor.x, y: cursor.y };
        } else {
          if (selected.x === cursor.x && selected.y === cursor.y) {
            selected = null;
          } else {
            trySwap(selected.y, selected.x, cursor.y, cursor.x);
            selected = null;
          }
        }
        return true;
      }
      return false;
    },
    getMeta() {
      return {
        name: 'Gem Crush',
        description: 'Match 3 or more gems in a row to clear them, score points, and advance through levels.',
        controls: {
          up: 'Move cursor up',
          down: 'Move cursor down',
          left: 'Move cursor left',
          right: 'Move cursor right',
          action: 'Select gem / confirm swap'
        },
        objective: 'Reach the target score within the move limit to complete each level',
        scoring: '3-match = 100pts, 4-match = 300pts, 5+ match = 500+ pts. Chain cascades multiply points.',
        tips: [
          'Select a gem with action, then move cursor to an adjacent gem and press action to swap',
          'Cascading chain reactions multiply your score significantly',
          'Plan your moves carefully since each level has a move limit',
          'Matching from the bottom of the board tends to create more cascades',
          'Leftover moves at level end give a 50-point bonus each'
        ]
      };
    }
  };

  window.ClawmachineGame = game;
})();
```

## Verification Checklist

- [ ] `window.ClawmachineGame` is assigned with all 6 methods: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] Canvas referenced via `document.getElementById('clawmachine-canvas')` (not created)
- [ ] `getState()` returns `score` (number) and `gameOver` (boolean)
- [ ] `getState()` returns puzzle-specific state: `grid`, `cursorX`, `cursorY`, `selected`, `movesLeft`, `level`
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `puzzle` on submission
- [ ] Grid-based match-3 logic detects horizontal and vertical matches
- [ ] Gravity fills empty spaces and spawns new gems
- [ ] Cascading chain reactions are detected and scored
- [ ] Move counter decrements on valid swaps
- [ ] Level progression works when target score is reached
- [ ] Game over triggers when moves run out without reaching target
