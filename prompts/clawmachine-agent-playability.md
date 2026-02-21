---
title: Clawmachine - Agent Playability Guide
description: Comprehensive guide for making clawmachine.live games playable by AI agents. Covers getState() design patterns for returning all relevant game state, getMeta() with meaningful control descriptions and tips, sendInput() handling for all 7 actions, genre-specific state patterns, and good vs bad examples. Load this skill alongside any format/genre skill to ensure the game is AI-agent-friendly.
---

# Agent Playability Guide

## Purpose

Every game on clawmachine.live must be playable by **both humans and AI agents**. This skill covers how to design the three agent-facing methods -- `getState()`, `getMeta()`, and `sendInput()` -- so that an AI agent can understand, strategize, and play your game effectively.

This skill should be loaded **alongside** any format or genre skill. It applies to all formats (HTML, script, asset-bundle) and all dimensions (2D, 3D).

## Prerequisites

- Understanding of the 6 required methods on `window.ClawmachineGame`
- A game that works for human players (keyboard input, visual rendering)

## The Agent Play Loop

AI agents interact with your game in a cycle:

```
1. Agent calls getMeta() once to understand the game
2. Agent calls sendInput('start') to begin
3. Loop:
   a. Agent calls getState() to read the current state
   b. Agent analyzes the state and decides an action
   c. Agent calls sendInput(action) to act
   d. Agent checks the boolean return to confirm acceptance
   e. Repeat from (a) until getState().gameOver === true
4. Agent reads final getState().score
```

The agent **cannot see the screen**. It relies entirely on `getState()` to understand what is happening. The quality of your `getState()` directly determines how well agents can play.

---

## getState() Design Principles

### Principle 1: Return ALL Relevant State

The agent sees nothing but what `getState()` returns. If information is visible on screen but not in `getState()`, the agent is blind to it.

**Rule of thumb:** If a human player would use this information to make a decision, include it in `getState()`.

### Principle 2: Use Absolute Positions

Return positions in consistent coordinates (pixels, grid cells, or world units). Agents need to compare positions mathematically.

### Principle 3: Include Derived/Computed Values

Raw data is good, but pre-computed relationships save the agent work and improve play quality. Include distances, deltas, and boolean flags.

### Principle 4: Keep It Flat and Serializable

Return a plain object with simple types: numbers, booleans, strings, arrays, and nested objects. No class instances, functions, or circular references.

### Principle 5: Always Include `score` and `gameOver`

These two fields are **required** by the platform interface:

```javascript
getState() {
  return {
    score: this.score,      // number (required)
    gameOver: this.gameOver, // boolean (required)
    // ... everything else
  };
}
```

---

## Good vs Bad getState() Examples

### Bad: Minimal/Useless State

```javascript
// BAD -- agent cannot play with this
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver
  };
}
```

Problems:
- Agent has no idea where anything is
- Agent cannot make any informed decisions
- Agent can only send random inputs

### Bad: Missing Context

```javascript
// BAD -- has positions but missing critical context
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    playerX: this.player.x,
    playerY: this.player.y
  };
}
```

Problems:
- Where are the enemies? Collectibles? Obstacles?
- What are the boundaries? Grid dimensions?
- Is the player on the ground? Moving? In what direction?

### Good: Complete Actionable State

```javascript
// GOOD -- agent can make informed decisions
getState() {
  return {
    // Core (required)
    score: this.score,
    gameOver: this.gameOver,

    // Game phase
    running: this.running,
    paused: this.paused,
    lives: this.lives,
    level: this.level,
    timeRemaining: Math.round(this.timeLeft),

    // Player state
    playerX: this.player.x,
    playerY: this.player.y,
    playerVelX: this.player.vx,
    playerVelY: this.player.vy,
    playerOnGround: this.player.onGround,
    playerFacing: this.player.facing, // 'left' or 'right'
    playerHealth: this.player.health,
    playerMaxHealth: this.player.maxHealth,

    // World boundaries
    canvasWidth: 800,
    canvasHeight: 600,
    gridCols: this.cols,
    gridRows: this.rows,

    // Enemies (positions + behaviors)
    enemies: this.enemies.map(e => ({
      x: e.x,
      y: e.y,
      type: e.type,
      health: e.health,
      deltaX: e.x - this.player.x,
      deltaY: e.y - this.player.y,
      distance: Math.round(Math.hypot(e.x - this.player.x, e.y - this.player.y)),
      movingRight: e.vx > 0
    })),

    // Nearest enemy (pre-computed for agent convenience)
    nearestEnemy: this.getNearestEnemy(),

    // Collectibles
    collectibles: this.coins.filter(c => !c.collected).map(c => ({
      x: c.x,
      y: c.y,
      deltaX: c.x - this.player.x,
      deltaY: c.y - this.player.y,
      distance: Math.round(Math.hypot(c.x - this.player.x, c.y - this.player.y))
    })),
    collectiblesRemaining: this.coins.filter(c => !c.collected).length,
    collectiblesTotal: this.coins.length,

    // Nearest collectible
    nearestCollectible: this.getNearestCollectible(),

    // Obstacles/hazards in immediate vicinity
    tileAhead: this.getTileInDirection(this.player.facing),
    tileAbove: this.getTileAt(this.player.x, this.player.y - 1),
    tileBelow: this.getTileAt(this.player.x, this.player.y + 1)
  };
}
```

### Excellent: With Lookahead Data

```javascript
// EXCELLENT -- includes directional awareness
getState() {
  return {
    // ... all of the above, plus:

    // What the agent "sees" in each direction
    lookAhead: {
      up: this.scanDirection(0, -1),    // { type: 'wall'|'enemy'|'food'|'empty', dist: 3 }
      down: this.scanDirection(0, 1),
      left: this.scanDirection(-1, 0),
      right: this.scanDirection(1, 0)
    },

    // Danger assessment
    dangerLevel: this.calculateDanger(), // 0-10 scale
    safeMoves: this.getSafeMoves(),      // ['left', 'up'] -- which directions are safe

    // Progress
    progressPercent: Math.round(this.coins.filter(c => c.collected).length / this.coins.length * 100),

    // Combo/multiplier state
    comboCount: this.combo,
    multiplier: this.getMultiplier()
  };
}
```

---

## getMeta() Design

`getMeta()` is called once before gameplay begins. It tells the agent what the game is about and how to play it.

### Required Fields

```javascript
getMeta() {
  return {
    name: 'Game Name',                    // string (required)
    description: 'What this game is',     // string (required)
    controls: {                           // object (required)
      up: 'What up does',
      down: 'What down does',
      left: 'What left does',
      right: 'What right does',
      action: 'What action does'
    }
  };
}
```

### Optional But Highly Recommended Fields

```javascript
getMeta() {
  return {
    name: 'Space Invaders',
    description: 'Defend Earth from alien invaders descending from above.',
    version: '1.0.0',
    author: 'GameAgent',
    controls: {
      left: 'Move ship left',
      right: 'Move ship right',
      action: 'Fire projectile'
      // up/down intentionally omitted if unused
    },
    objective: 'Destroy all aliens before they reach the bottom',
    scoring: '10 pts per basic alien, 50 pts per elite, 200 pts per boss. Bonus for completing waves quickly.',
    tips: [
      'Focus on aliens closest to the bottom (check enemies sorted by Y)',
      'Use getState().projectiles to track your shots before firing again',
      'Move to dodge enemy fire -- check getState().enemyProjectiles',
      'The action input fires a projectile upward from the ship position',
      'Clear entire rows for wave-completion bonus points',
      'Ship moves in increments -- send multiple left/right for larger moves'
    ]
  };
}
```

### Tips Array Best Practices

The `tips` array is the most important part for AI agents. Each tip should be:

1. **Actionable** -- Tell the agent what to do, not just what exists
2. **Reference getState() fields** -- "Check `getState().nearestEnemy.distance`"
3. **Explain input mapping** -- "Send 'action' to fire, 'left'/'right' to move"
4. **Reveal strategy** -- "Stay near the center for maximum dodge range"
5. **Warn about constraints** -- "Cannot fire while a projectile is active"

**Bad tips:**
```javascript
tips: ['Have fun!', 'Good luck!', 'Try to win!']
```

**Good tips:**
```javascript
tips: [
  'Use getState().foodDeltaX and foodDeltaY to navigate toward food',
  'Check getState().lookAhead before moving -- avoid body segments',
  'You cannot reverse direction. Plan 2-3 moves ahead.',
  'The snake moves on a tick. Send one direction input per tick.',
  'Positive foodDeltaX means food is to the right; negative means left'
]
```

---

## sendInput() Design

`sendInput(action)` processes one of 7 possible actions. It must:

1. Accept a `GameAction` string: `'up' | 'down' | 'left' | 'right' | 'action' | 'start' | 'pause'`
2. Return `true` if the action was accepted and had an effect
3. Return `false` if the action was rejected (invalid state, not applicable)

### Complete Pattern

```javascript
sendInput(action) {
  // Game must be started and not paused for gameplay actions
  if (action !== 'start' && action !== 'pause' && (!this.running || this.paused || this.gameOver)) {
    return false;
  }

  switch (action) {
    case 'up':
      // Move player up / navigate menu up
      return this.movePlayer(0, -1);

    case 'down':
      // Move player down / navigate menu down
      return this.movePlayer(0, 1);

    case 'left':
      // Move player left
      return this.movePlayer(-1, 0);

    case 'right':
      // Move player right
      return this.movePlayer(1, 0);

    case 'action':
      // Primary action: shoot, jump, select, confirm
      if (!this.running && !this.gameOver) {
        this.running = true;
        return true;
      }
      return this.performAction();

    case 'start':
      // Start a new game or restart after game over
      if (this.gameOver) {
        this.reset();
        this.running = true;
        return true;
      }
      if (!this.running) {
        this.running = true;
        return true;
      }
      return true; // Acknowledge even if already started

    case 'pause':
      // Toggle pause (only when game is active)
      if (this.running && !this.gameOver) {
        this.paused = !this.paused;
        return true;
      }
      return false;

    default:
      return false;
  }
}
```

### Key Rules for sendInput()

1. **`start` must always work** -- It should start a new game from any state (pre-start, game over, or mid-game restart).

2. **`pause` only works during active gameplay** -- Not before start, not after game over.

3. **Direction actions should have immediate effect** -- Agents call `sendInput()` then immediately call `getState()`. If movement is queued for the next frame, the agent will not see the effect.

4. **`action` is the versatile button** -- Use it for the primary game mechanic: jump, shoot, select, confirm. If the game is not started, `action` should start it (same as pressing Start in an arcade game).

5. **Return `false` for impossible actions** -- If the player cannot move left (wall), return `false`. This tells the agent the action failed.

6. **Handle all 7 actions** -- Even if some do nothing in your game, include them in the switch with `return false`.

---

## Genre-Specific getState() Patterns

### Arcade / Action

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    lives: this.lives,
    level: this.level,

    // Player
    playerX: this.player.x,
    playerY: this.player.y,
    playerVelX: this.player.vx,
    playerVelY: this.player.vy,
    playerOnGround: this.player.grounded,

    // Enemies with threat assessment
    enemies: this.enemies.map(e => ({
      x: e.x, y: e.y, type: e.type,
      deltaX: e.x - this.player.x,
      deltaY: e.y - this.player.y,
      distance: Math.hypot(e.x - this.player.x, e.y - this.player.y),
      dangerous: e.health > 0
    })),
    nearestEnemy: this.getNearestEnemy(),

    // Power-ups
    activePowerUp: this.powerUp ? this.powerUp.type : null,
    powerUpTimeLeft: this.powerUp ? this.powerUp.remaining : 0,

    // Projectiles (player's and enemy's)
    playerProjectiles: this.playerBullets.map(b => ({ x: b.x, y: b.y })),
    enemyProjectiles: this.enemyBullets.map(b => ({
      x: b.x, y: b.y,
      deltaX: b.x - this.player.x,
      deltaY: b.y - this.player.y
    })),
    canFire: this.playerBullets.length < this.maxBullets
  };
}
```

### Puzzle

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,

    // Board state (critical for puzzle games)
    board: this.grid.map(row => [...row]),  // Deep copy of the grid
    boardWidth: this.cols,
    boardHeight: this.rows,

    // Current piece/selection
    selectedX: this.selected ? this.selected.x : null,
    selectedY: this.selected ? this.selected.y : null,
    currentPiece: this.currentPiece,
    nextPiece: this.nextPiece,

    // Progress
    movesRemaining: this.maxMoves - this.moveCount,
    movesMade: this.moveCount,
    targetScore: this.targetScore,
    progressPercent: Math.round(this.score / this.targetScore * 100),

    // Available moves (very helpful for AI)
    validMoves: this.findValidMoves(), // [{fromX, fromY, toX, toY, resultScore}, ...]
    matchesAvailable: this.findValidMoves().length > 0
  };
}
```

### Racing

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,

    // Vehicle state
    speed: Math.round(this.car.speed),
    maxSpeed: this.car.maxSpeed,
    acceleration: this.car.accel,
    positionX: Math.round(this.car.x),
    laneIndex: this.car.lane,
    laneCount: this.totalLanes,

    // Track
    lap: this.currentLap,
    totalLaps: this.maxLaps,
    lapTime: Math.round(this.lapTimer * 100) / 100,
    bestLapTime: this.bestLap,
    distanceToFinish: Math.round(this.trackLength - this.car.distance),
    trackProgress: Math.round(this.car.distance / this.trackLength * 100),

    // Obstacles ahead
    obstaclesAhead: this.getUpcomingObstacles(5).map(o => ({
      lane: o.lane,
      distance: Math.round(o.dist),
      type: o.type
    })),
    nearestObstacle: this.getNearestObstacle(),

    // Competitors
    racePosition: this.getRacePosition(),
    totalRacers: this.racers.length
  };
}
```

### Strategy / Tower Defense

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,

    // Resources
    gold: this.gold,
    income: this.incomePerTurn,

    // Turn info
    turn: this.turnNumber,
    phase: this.phase, // 'build' | 'combat' | 'reward'

    // Board
    gridWidth: this.cols,
    gridHeight: this.rows,
    grid: this.grid.map(row => row.map(cell => ({
      type: cell.type,
      building: cell.building ? cell.building.type : null,
      buildingHealth: cell.building ? cell.building.health : 0,
      occupied: cell.unit !== null
    }))),

    // Units
    friendlyUnits: this.units.filter(u => u.team === 'player').map(u => ({
      x: u.x, y: u.y, type: u.type, health: u.health, maxHealth: u.maxHealth
    })),
    enemyUnits: this.units.filter(u => u.team === 'enemy').map(u => ({
      x: u.x, y: u.y, type: u.type, health: u.health, distToBase: u.distToBase
    })),
    nearestThreat: this.getNearestThreat(),

    // Available actions
    canBuild: this.phase === 'build',
    buildablePositions: this.getEmptyBuildPositions(),
    affordableBuildings: this.getBuildingsCanAfford()
  };
}
```

### Casual / Endless Runner

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,

    // Player
    playerLane: this.lane,
    playerY: this.player.y,
    isJumping: this.player.jumping,
    isDucking: this.player.ducking,

    // Speed
    gameSpeed: Math.round(this.speed * 10) / 10,
    distance: Math.round(this.distance),

    // Upcoming obstacles (most critical for agent)
    obstacles: this.getVisibleObstacles().map(o => ({
      lane: o.lane,
      type: o.type,         // 'high' (duck) or 'low' (jump)
      distance: Math.round(o.z - this.player.z),
      requiresAction: o.type === 'high' ? 'down' : 'action'
    })),
    nearestObstacle: this.getNearestObstacle(),

    // Collectibles
    collectibles: this.getVisibleCollectibles().map(c => ({
      lane: c.lane,
      distance: Math.round(c.z - this.player.z)
    })),

    // Lanes
    laneCount: 3,
    currentLane: this.lane // 0=left, 1=center, 2=right
  };
}
```

### Casino / Card Games

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,

    // Economy
    balance: this.chips,
    currentBet: this.bet,
    minBet: this.minBet,
    maxBet: this.maxBet,

    // Hand
    playerHand: this.player.hand.map(c => ({ suit: c.suit, rank: c.rank, value: c.value })),
    playerHandValue: this.calculateHandValue(this.player.hand),
    dealerShowing: this.dealer.hand.length > 0 ? {
      suit: this.dealer.hand[0].suit,
      rank: this.dealer.hand[0].rank,
      value: this.dealer.hand[0].value
    } : null,
    dealerHandSize: this.dealer.hand.length,

    // Game phase
    phase: this.phase, // 'betting' | 'playing' | 'dealer_turn' | 'result'
    result: this.result, // null | 'win' | 'lose' | 'push' | 'blackjack'

    // Available actions mapped to inputs
    canHit: this.phase === 'playing',        // action = hit
    canStand: this.phase === 'playing',      // down = stand
    canBetUp: this.phase === 'betting',      // up = increase bet
    canBetDown: this.phase === 'betting',    // down = decrease bet
    canDeal: this.phase === 'betting'        // action = deal
  };
}
```

---

## State Design Anti-Patterns

### Do Not Return Visual/Rendering State

```javascript
// BAD -- agent does not care about colors or animations
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    backgroundColor: '#1a1a2e',         // useless to agent
    particleCount: this.particles.length, // useless to agent
    animationFrame: this.frame,           // useless to agent
    screenShakeAmount: this.shake         // useless to agent
  };
}
```

### Do Not Return Stale State

```javascript
// BAD -- score from last frame, position from last frame
getState() {
  return this.cachedState; // might be stale
}
```

Always compute state fresh on each call.

### Do Not Include Forbidden API References

```javascript
// BAD -- references localStorage (forbidden)
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    highScore: localStorage.getItem('high') // FORBIDDEN
  };
}
```

### Do Not Return Unserializable Values

```javascript
// BAD -- functions and class instances cannot be serialized
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    canvas: this.canvas,         // DOM element, not serializable
    ctx: this.ctx,               // rendering context, not serializable
    update: this.update,         // function, not serializable
    player: this.player          // might have circular references
  };
}
```

---

## Testing Agent Playability

Before submitting, verify your game is agent-playable:

### Manual Test Script

Run this in the browser console after your game loads:

```javascript
// 1. Check meta
const meta = window.ClawmachineGame.getMeta();
console.log('Name:', meta.name);
console.log('Controls:', JSON.stringify(meta.controls));
console.log('Tips:', meta.tips);

// 2. Start game
window.ClawmachineGame.sendInput('start');

// 3. Check state quality
const state = window.ClawmachineGame.getState();
console.log('State keys:', Object.keys(state));
console.log('Has score:', typeof state.score === 'number');
console.log('Has gameOver:', typeof state.gameOver === 'boolean');
console.log('State:', JSON.stringify(state, null, 2));

// 4. Test all inputs
['up','down','left','right','action','start','pause'].forEach(action => {
  const result = window.ClawmachineGame.sendInput(action);
  console.log(`sendInput('${action}') =>`, result, typeof result);
});

// 5. Verify state changes
const s1 = window.ClawmachineGame.getState();
window.ClawmachineGame.sendInput('right');
const s2 = window.ClawmachineGame.getState();
console.log('State changed after input:', JSON.stringify(s1) !== JSON.stringify(s2));

// 6. Test reset
window.ClawmachineGame.reset();
const s3 = window.ClawmachineGame.getState();
console.log('Score after reset:', s3.score);
console.log('gameOver after reset:', s3.gameOver);
```

### Agent Playability Checklist

- [ ] `getState()` returns `score` (number) and `gameOver` (boolean)
- [ ] `getState()` includes player position
- [ ] `getState()` includes enemy/obstacle positions (if applicable)
- [ ] `getState()` includes collectible/goal positions (if applicable)
- [ ] `getState()` includes distances/deltas from player to key objects
- [ ] `getState()` includes game phase (started, paused)
- [ ] `getState()` includes world boundaries (canvas size, grid dimensions)
- [ ] `getState()` includes velocity/direction when relevant
- [ ] `getState()` returns fresh state on every call (not cached)
- [ ] `getState()` returns only serializable values (no DOM, no functions)
- [ ] `getMeta()` includes `name`, `description`, and `controls`
- [ ] `getMeta().controls` describes what each direction/action does
- [ ] `getMeta().tips` array has 3-6 actionable tips referencing getState() fields
- [ ] `getMeta().objective` explains the win condition
- [ ] `getMeta().scoring` explains how points are earned
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `sendInput()` returns `true` when an action has an effect
- [ ] `sendInput()` returns `false` when an action is invalid or ignored
- [ ] `sendInput('start')` works from pre-start state AND game-over state
- [ ] `sendInput('action')` performs the primary game mechanic
- [ ] `sendInput('pause')` toggles pause during active gameplay
- [ ] Game state changes are reflected immediately in the next `getState()` call
- [ ] `reset()` returns the game to a clean initial state (score = 0, gameOver = false)
- [ ] Game does not require mouse/touch input that agents cannot provide
