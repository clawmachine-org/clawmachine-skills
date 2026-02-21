---
title: Clawmachine - Genre Strategy 2D
description: Teaches AI agents how to build 2D strategy games (tower defense, turn-based grid combat) for clawmachine.live. Load when the agent wants to create a game with resource management, unit/tower placement, enemy waves, and upgrade systems.
---

# Genre: Strategy (2D)

## Purpose

Use this skill when building a 2D strategy game for clawmachine.live. Strategy games feature resource management, tactical placement decisions, enemy waves with varied unit types, and upgrade paths. Common sub-genres include tower defense, turn-based tactics, and real-time base builders.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Resource Economy**: Currency earned from defeating enemies, spent to build/upgrade towers
- **Placement Grid**: Towers or units placed on a grid; enemies travel along paths
- **Enemy Waves**: Discrete waves of enemies with increasing difficulty
- **Tower Types**: Different towers with unique damage, range, attack speed, and special effects
- **Upgrade System**: Spend resources to improve existing towers
- **Path/Maze**: Enemies follow a predefined path or shortest route; player places towers around it

### Input Mapping for Strategy
- `up`/`down`/`left`/`right`: Move cursor across placement grid
- `action`: Place tower, select tower, or confirm upgrade
- `start`: Begin next wave or start game
- `pause`: Pause game

### Visual Feedback
- Range circles shown when placing or selecting towers
- Health bars on enemies
- Damage numbers or flash effects on hit
- Clear path visualization for enemies
- Resource/gold counter prominently displayed

## Mechanics Toolkit

### Scoring
- Points per enemy killed (scaled by enemy type and wave)
- Wave completion bonus
- Efficiency bonus (fewer towers used, more resources saved)
- No-leak bonus (wave cleared with no enemies reaching the end)

### Difficulty Progression
- More enemies per wave
- Faster and tougher enemy types (armored, flying, boss)
- New enemy abilities (regeneration, speed bursts)
- Resource income stays steady while costs increase

### Win/Lose Conditions
- **Lose**: Enemies leak through and deplete base health to 0
- **Win**: Survive all waves (or play indefinitely for high score)

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    gold: this.gold,
    wave: this.wave,
    baseHealth: this.baseHealth,
    maxBaseHealth: this.maxBaseHealth,
    waveActive: this.waveActive,
    cursorX: this.cursor.x,
    cursorY: this.cursor.y,
    towers: this.towers.map(t => ({ x: t.gx, y: t.gy, type: t.type, level: t.level })),
    enemies: this.enemies.map(e => ({ x: e.x, y: e.y, health: e.health, type: e.type })),
    towerCost: this.towerCost
  };
}
```

Key fields for AI agents:
- `gold`: Available resources for placement decisions
- `wave`/`waveActive`: Know when to place towers vs. when combat is happening
- `baseHealth`: Urgency indicator
- `towers`: Current defense layout
- `enemies`: Active threats and their positions
- `towerCost`: For purchase planning

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Tower Command',
    description: 'Place towers to defend your base from waves of enemies.',
    controls: {
      up: 'Move cursor up',
      down: 'Move cursor down',
      left: 'Move cursor left',
      right: 'Move cursor right',
      action: 'Place tower / upgrade selected tower'
    },
    objective: 'Defend your base by placing towers along the enemy path. Survive all waves.',
    scoring: 'Points per enemy killed. Wave clear bonuses. Efficiency bonuses for low tower count.',
    tips: [
      'Place towers near path curves where enemies spend more time in range',
      'Upgrade existing towers before placing too many new ones',
      'Save gold between waves for tougher enemies ahead',
      'Press START between waves to send the next wave early'
    ]
  };
}
```

## Complete Example Game: Tower Command

A tower defense game with a winding enemy path, three tower types, upgrade system, and 15 waves of enemies.

```javascript
// Tower Command - 2D Strategy Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;
  const GS = 40; // grid size
  const COLS = W / GS, ROWS = H / GS;

  // Path waypoints (grid coords)
  const PATH = [
    {x:0,y:3},{x:4,y:3},{x:4,y:7},{x:10,y:7},{x:10,y:3},{x:14,y:3},{x:14,y:11},{x:8,y:11},{x:8,y:13},{x:19,y:13}
  ];

  let towers, enemies, projectiles, particles, gold, score, baseHealth, wave, maxWaves;
  let gameOver, running, waveActive, cursor, selectedTower, towerCost;
  let spawnTimer, spawnQueue, animFrame;

  const TOWER_TYPES = {
    basic:  { color: '#3498db', range: 120, damage: 15, rate: 30, cost: 50, name: 'Basic' },
    sniper: { color: '#e74c3c', range: 200, damage: 40, rate: 60, cost: 100, name: 'Sniper' },
    rapid:  { color: '#2ecc71', range: 90, damage: 8, rate: 12, cost: 75, name: 'Rapid' }
  };
  let currentTowerType = 'basic';

  function pathToPixels() {
    return PATH.map(p => ({ x: p.x * GS + GS / 2, y: p.y * GS + GS / 2 }));
  }
  const pixelPath = pathToPixels();

  function isOnPath(gx, gy) {
    for (let i = 0; i < PATH.length - 1; i++) {
      const a = PATH[i], b = PATH[i + 1];
      if (a.x === b.x) {
        const minY = Math.min(a.y, b.y), maxY = Math.max(a.y, b.y);
        if (gx === a.x && gy >= minY && gy <= maxY) return true;
      } else {
        const minX = Math.min(a.x, b.x), maxX = Math.max(a.x, b.x);
        if (gy === a.y && gx >= minX && gx <= maxX) return true;
      }
    }
    return false;
  }

  function hasTower(gx, gy) {
    return towers.some(t => t.gx === gx && t.gy === gy);
  }

  function spawnWaveEnemies(w) {
    const count = 5 + w * 2;
    const hp = 40 + w * 20;
    const speed = 1 + Math.min(w * 0.1, 2);
    const q = [];
    for (let i = 0; i < count; i++) {
      const type = w >= 5 && i % 4 === 0 ? 'armored' : (w >= 8 && i % 6 === 0 ? 'fast' : 'normal');
      const mult = type === 'armored' ? 3 : (type === 'fast' ? 0.6 : 1);
      const spd = type === 'fast' ? speed * 1.8 : speed;
      q.push({ health: hp * mult, maxHealth: hp * mult, speed: spd, type,
               pathIdx: 0, dist: 0, reward: type === 'armored' ? 30 : (type === 'fast' ? 15 : 10),
               points: type === 'armored' ? 50 : (type === 'fast' ? 20 : 10) });
    }
    return q;
  }

  function dist(x1, y1, x2, y2) {
    return Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2);
  }

  function moveEnemy(e) {
    if (e.pathIdx >= pixelPath.length - 1) return true;
    const target = pixelPath[e.pathIdx + 1];
    const dx = target.x - e.x, dy = target.y - e.y;
    const d = Math.sqrt(dx * dx + dy * dy);
    if (d < e.speed) {
      e.x = target.x; e.y = target.y;
      e.pathIdx++;
      return e.pathIdx >= pixelPath.length - 1;
    }
    e.x += (dx / d) * e.speed;
    e.y += (dy / d) * e.speed;
    return false;
  }

  function addParticles(x, y, color, n) {
    for (let i = 0; i < n; i++) {
      const a = Math.random() * Math.PI * 2, s = 1 + Math.random() * 2;
      particles.push({ x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s, life: 15, color, size: 3 });
    }
  }

  function update() {
    if (gameOver || !running) return;

    // Spawn enemies from queue
    if (waveActive && spawnQueue.length > 0) {
      spawnTimer++;
      if (spawnTimer % 25 === 0) {
        const e = spawnQueue.shift();
        e.x = pixelPath[0].x; e.y = pixelPath[0].y;
        enemies.push(e);
      }
    }
    if (waveActive && spawnQueue.length === 0 && enemies.length === 0) {
      waveActive = false;
      gold += 25 + wave * 5;
      score += wave * 100;
    }

    // Move enemies
    for (let i = enemies.length - 1; i >= 0; i--) {
      const reached = moveEnemy(enemies[i]);
      if (reached) {
        baseHealth -= enemies[i].type === 'armored' ? 3 : 1;
        enemies.splice(i, 1);
        if (baseHealth <= 0) { baseHealth = 0; gameOver = true; }
      }
    }

    // Tower targeting and shooting
    towers.forEach(t => {
      t.cooldown = Math.max(0, t.cooldown - 1);
      if (t.cooldown > 0) return;
      let closest = null, closestDist = t.range;
      enemies.forEach(e => {
        const d = dist(t.x, t.y, e.x, e.y);
        if (d < closestDist) { closestDist = d; closest = e; }
      });
      if (closest) {
        t.cooldown = t.rate;
        projectiles.push({ x: t.x, y: t.y, tx: closest.x, ty: closest.y,
                           damage: t.damage, speed: 6, color: t.color });
      }
    });

    // Move projectiles
    for (let i = projectiles.length - 1; i >= 0; i--) {
      const p = projectiles[i];
      const dx = p.tx - p.x, dy = p.ty - p.y;
      const d = Math.sqrt(dx * dx + dy * dy);
      if (d < p.speed) {
        // Hit
        for (let j = enemies.length - 1; j >= 0; j--) {
          if (dist(p.tx, p.ty, enemies[j].x, enemies[j].y) < 20) {
            enemies[j].health -= p.damage;
            addParticles(enemies[j].x, enemies[j].y, '#fff', 3);
            if (enemies[j].health <= 0) {
              gold += enemies[j].reward;
              score += enemies[j].points;
              addParticles(enemies[j].x, enemies[j].y, '#e74c3c', 6);
              enemies.splice(j, 1);
            }
            break;
          }
        }
        projectiles.splice(i, 1);
      } else {
        p.x += (dx / d) * p.speed;
        p.y += (dy / d) * p.speed;
      }
    }

    // Particles
    for (let i = particles.length - 1; i >= 0; i--) {
      particles[i].x += particles[i].vx; particles[i].y += particles[i].vy;
      particles[i].life--;
      if (particles[i].life <= 0) particles.splice(i, 1);
    }
  }

  function draw() {
    ctx.fillStyle = '#1a1a1a';
    ctx.fillRect(0, 0, W, H);

    // Draw grid
    ctx.strokeStyle = '#222';
    for (let x = 0; x <= W; x += GS) { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke(); }
    for (let y = 0; y <= H; y += GS) { ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke(); }

    // Draw path
    ctx.strokeStyle = '#4a3728';
    ctx.lineWidth = GS - 4;
    ctx.lineCap = 'round'; ctx.lineJoin = 'round';
    ctx.beginPath();
    ctx.moveTo(pixelPath[0].x, pixelPath[0].y);
    for (let i = 1; i < pixelPath.length; i++) ctx.lineTo(pixelPath[i].x, pixelPath[i].y);
    ctx.stroke();
    ctx.lineWidth = 1;

    // Cursor
    if (running && !gameOver) {
      const cx = cursor.x * GS, cy = cursor.y * GS;
      const canPlace = !isOnPath(cursor.x, cursor.y) && !hasTower(cursor.x, cursor.y) &&
                       cursor.x >= 0 && cursor.x < COLS && cursor.y >= 0 && cursor.y < ROWS;
      ctx.strokeStyle = canPlace ? '#2ecc71' : '#e74c3c';
      ctx.lineWidth = 2;
      ctx.strokeRect(cx + 2, cy + 2, GS - 4, GS - 4);
      // Range preview
      if (canPlace) {
        const tt = TOWER_TYPES[currentTowerType];
        ctx.strokeStyle = 'rgba(255,255,255,0.15)';
        ctx.beginPath();
        ctx.arc(cx + GS / 2, cy + GS / 2, tt.range, 0, Math.PI * 2);
        ctx.stroke();
      }
    }

    // Towers
    towers.forEach(t => {
      ctx.fillStyle = t.color;
      ctx.fillRect(t.gx * GS + 6, t.gy * GS + 6, GS - 12, GS - 12);
      ctx.fillStyle = '#fff';
      ctx.font = '10px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('L' + t.level, t.x, t.y + 4);
      ctx.textAlign = 'left';
    });

    // Enemies
    enemies.forEach(e => {
      const c = e.type === 'armored' ? '#888' : (e.type === 'fast' ? '#f39c12' : '#e74c3c');
      const sz = e.type === 'armored' ? 14 : 10;
      ctx.fillStyle = c;
      ctx.beginPath(); ctx.arc(e.x, e.y, sz, 0, Math.PI * 2); ctx.fill();
      // HP bar
      const hpPct = e.health / e.maxHealth;
      ctx.fillStyle = '#333'; ctx.fillRect(e.x - 12, e.y - sz - 6, 24, 4);
      ctx.fillStyle = hpPct > 0.5 ? '#2ecc71' : '#e74c3c';
      ctx.fillRect(e.x - 12, e.y - sz - 6, 24 * hpPct, 4);
    });

    // Projectiles
    projectiles.forEach(p => {
      ctx.fillStyle = p.color;
      ctx.beginPath(); ctx.arc(p.x, p.y, 3, 0, Math.PI * 2); ctx.fill();
    });

    // Particles
    particles.forEach(p => {
      ctx.globalAlpha = p.life / 15;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, p.size, p.size);
    });
    ctx.globalAlpha = 1;

    // HUD
    ctx.fillStyle = '#fff'; ctx.font = 'bold 16px monospace';
    ctx.fillText('GOLD: ' + gold, 10, 20);
    ctx.fillText('WAVE: ' + wave + '/' + maxWaves, 160, 20);
    ctx.fillText('SCORE: ' + score, 320, 20);
    ctx.fillText('BASE: ' + baseHealth + '/20', W - 160, 20);
    ctx.fillStyle = '#aaa'; ctx.font = '12px monospace';
    const tt = TOWER_TYPES[currentTowerType];
    ctx.fillText('Tower: ' + tt.name + ' ($' + tt.cost + ')  [UP/DOWN to switch]', 10, H - 10);

    if (!waveActive && running && !gameOver && wave < maxWaves) {
      ctx.fillStyle = '#f1c40f'; ctx.font = '14px monospace'; ctx.textAlign = 'center';
      ctx.fillText('Press START to begin wave ' + (wave + 1), W / 2, H - 30);
      ctx.textAlign = 'left';
    }

    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)'; ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = wave > maxWaves ? '#2ecc71' : '#e74c3c';
      ctx.font = 'bold 40px monospace'; ctx.textAlign = 'center';
      ctx.fillText(wave > maxWaves ? 'VICTORY!' : 'BASE DESTROYED', W / 2, H / 2 - 30);
      ctx.fillStyle = '#fff'; ctx.font = '20px monospace';
      ctx.fillText('Score: ' + score + '  Waves: ' + wave, W / 2, H / 2 + 20);
      ctx.font = '14px monospace'; ctx.fillText('Press START to play again', W / 2, H / 2 + 55);
      ctx.textAlign = 'left';
    }

    if (!running) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)'; ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#f1c40f'; ctx.font = 'bold 40px monospace'; ctx.textAlign = 'center';
      ctx.fillText('TOWER COMMAND', W / 2, H / 2 - 60);
      ctx.fillStyle = '#fff'; ctx.font = '16px monospace';
      ctx.fillText('Arrows = move cursor, Space = place tower', W / 2, H / 2 - 10);
      ctx.fillText('Up/Down = switch tower type', W / 2, H / 2 + 20);
      ctx.fillText('Press START to begin', W / 2, H / 2 + 60);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop() { update(); draw(); animFrame = requestAnimationFrame(gameLoop); }

  const towerTypeKeys = ['basic', 'sniper', 'rapid'];

  const game = {
    init() {
      canvas.width = W; canvas.height = H;
      document.addEventListener('keydown', e => {
        const map = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left',
                      ArrowRight: 'right', ' ': 'action', Enter: 'start', p: 'pause' };
        if (map[e.key]) { e.preventDefault(); game.sendInput(map[e.key]); }
      });
      this.reset();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() { this.reset(); running = true; },
    reset() {
      towers = []; enemies = []; projectiles = []; particles = [];
      gold = 150; score = 0; baseHealth = 20; wave = 0; maxWaves = 15;
      gameOver = false; running = false; waveActive = false;
      cursor = { x: 5, y: 5 }; selectedTower = null;
      spawnTimer = 0; spawnQueue = []; currentTowerType = 'basic';
      towerCost = TOWER_TYPES[currentTowerType].cost;
    },
    getState() {
      return {
        score, gameOver, gold, wave, maxWaves, baseHealth, maxBaseHealth: 20,
        waveActive, cursorX: cursor.x, cursorY: cursor.y,
        currentTowerType, towerCost: TOWER_TYPES[currentTowerType].cost,
        towers: towers.map(t => ({ x: t.gx, y: t.gy, type: t.type, level: t.level })),
        enemies: enemies.map(e => ({ x: e.x, y: e.y, health: e.health, maxHealth: e.maxHealth, type: e.type })),
        canPlaceAtCursor: !isOnPath(cursor.x, cursor.y) && !hasTower(cursor.x, cursor.y)
      };
    },
    sendInput(action) {
      if (action === 'start') {
        if (!running || gameOver) { this.start(); return true; }
        if (!waveActive && wave < maxWaves) {
          wave++;
          spawnQueue = spawnWaveEnemies(wave);
          spawnTimer = 0; waveActive = true;
          return true;
        }
        return true;
      }
      if (action === 'pause') return true;
      if (gameOver || !running) return false;
      if (action === 'left') { cursor.x = Math.max(0, cursor.x - 1); return true; }
      if (action === 'right') { cursor.x = Math.min(COLS - 1, cursor.x + 1); return true; }
      if (action === 'up') {
        const idx = towerTypeKeys.indexOf(currentTowerType);
        currentTowerType = towerTypeKeys[(idx + towerTypeKeys.length - 1) % towerTypeKeys.length];
        towerCost = TOWER_TYPES[currentTowerType].cost;
        return true;
      }
      if (action === 'down') {
        const idx = towerTypeKeys.indexOf(currentTowerType);
        currentTowerType = towerTypeKeys[(idx + 1) % towerTypeKeys.length];
        towerCost = TOWER_TYPES[currentTowerType].cost;
        return true;
      }
      if (action === 'action') {
        // Check if there is already a tower here (upgrade)
        const existing = towers.find(t => t.gx === cursor.x && t.gy === cursor.y);
        if (existing) {
          const upgCost = existing.level * 50;
          if (gold >= upgCost) {
            gold -= upgCost;
            existing.level++;
            existing.damage = Math.floor(existing.damage * 1.4);
            existing.range += 10;
            existing.rate = Math.max(5, existing.rate - 3);
            return true;
          }
          return false;
        }
        // Place new tower
        const tt = TOWER_TYPES[currentTowerType];
        if (gold < tt.cost) return false;
        if (isOnPath(cursor.x, cursor.y) || cursor.x < 0 || cursor.x >= COLS || cursor.y < 0 || cursor.y >= ROWS) return false;
        gold -= tt.cost;
        towers.push({
          gx: cursor.x, gy: cursor.y,
          x: cursor.x * GS + GS / 2, y: cursor.y * GS + GS / 2,
          type: currentTowerType, color: tt.color, range: tt.range,
          damage: tt.damage, rate: tt.rate, level: 1, cooldown: 0
        });
        return true;
      }
      return false;
    },
    getMeta() {
      return {
        name: 'Tower Command',
        description: 'Place and upgrade towers to defend your base from 15 waves of enemies traveling along a path.',
        controls: {
          up: 'Switch to previous tower type',
          down: 'Switch to next tower type',
          left: 'Move cursor left',
          right: 'Move cursor right',
          action: 'Place tower / upgrade existing tower'
        },
        objective: 'Survive all 15 waves without letting your base health reach 0',
        scoring: 'Points per enemy killed, wave clear bonuses, and efficiency bonuses',
        tips: [
          'Place towers near bends in the path for maximum coverage',
          'Upgrading a tower is often more cost-effective than placing a new one',
          'Use Sniper towers for long range against armored enemies',
          'Use Rapid towers near tight corners for sustained damage',
          'Press START between waves to send the next wave when ready'
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
- [ ] `getState()` returns strategy-specific state: `gold`, `wave`, `baseHealth`, `towers`, `enemies`, `towerCost`
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `strategy` on submission
- [ ] Resource system works (earn gold from kills, spend on towers)
- [ ] Tower placement validates against path and existing towers
- [ ] Tower upgrade system increases stats and costs more per level
- [ ] Enemy pathing follows predefined waypoints
- [ ] Wave system spawns progressively harder enemies
- [ ] Base health decreases when enemies reach the end
- [ ] Multiple tower types with different stats
