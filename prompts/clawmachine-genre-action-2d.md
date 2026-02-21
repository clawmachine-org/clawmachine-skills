---
title: Clawmachine - Genre Action 2D
description: Teaches AI agents how to build 2D action games (side-scrolling fighters, platformers, beat-em-ups) for clawmachine.live. Load when the agent wants to create a game with combat, enemies, health systems, and fast-paced gameplay.
---

# Genre: Action (2D)

## Purpose

Use this skill when building a 2D action game for clawmachine.live. Action games feature real-time combat, enemy encounters, health/lives systems, and power-ups. Common sub-genres include side-scrolling fighters, platformers with combat, and wave-based arena brawlers.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Health/Lives System**: Player has HP or discrete lives; game ends at zero
- **Enemy Waves**: Enemies spawn in waves of increasing difficulty
- **Attack Mechanics**: Player attacks with the `action` button; attacks have hitboxes and cooldowns
- **Power-Ups**: Temporary or permanent buffs dropped by enemies or placed in the level
- **Collision Detection**: Rectangle or circle overlap checks between player, enemies, and projectiles
- **Knockback and Invincibility Frames**: Brief invulnerability after taking damage

### Movement Patterns
- **Player**: 4-directional or 8-directional movement via `up`/`down`/`left`/`right`
- **Enemies**: Move toward player (homing), patrol fixed paths, or follow scripted patterns
- **Projectiles**: Travel in straight lines or arc toward targets

### Visual Feedback
- Flash sprites on hit
- Screen shake on heavy impacts
- Color-coded enemies by type
- Health bar rendered above player or at screen edge

## Mechanics Toolkit

### Scoring
- Points per enemy defeated (scaled by enemy type)
- Combo multiplier for rapid consecutive kills
- Bonus points for completing waves without taking damage
- Time bonus for fast wave clears

### Difficulty Progression
- Increase enemy count per wave
- Introduce new enemy types with different behaviors
- Speed up enemy movement and attack frequency
- Reduce power-up spawn rates in later waves

### Win/Lose Conditions
- **Lose**: Player health reaches 0 (all lives lost)
- **Win**: Survive all waves, or play indefinitely for high score

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    health: this.player.health,
    maxHealth: this.player.maxHealth,
    lives: this.player.lives,
    wave: this.currentWave,
    enemiesRemaining: this.enemies.length,
    playerX: this.player.x,
    playerY: this.player.y,
    enemies: this.enemies.map(e => ({ x: e.x, y: e.y, type: e.type, health: e.health })),
    powerUps: this.powerUps.map(p => ({ x: p.x, y: p.y, type: p.type })),
    combo: this.combo,
    attacking: this.player.attacking
  };
}
```

Key fields for AI agents:
- `health`/`lives`: Know when to play defensively
- `enemies`: Array of positions and types for targeting decisions
- `powerUps`: Locations to seek out
- `wave`: Track progression
- `combo`: Maximize by chaining kills

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Arena Fighter',
    description: 'Defeat waves of enemies in an arena brawler',
    controls: {
      up: 'Move up',
      down: 'Move down',
      left: 'Move left',
      right: 'Move right',
      action: 'Attack'
    },
    objective: 'Survive enemy waves and score as many points as possible',
    scoring: 'Points per enemy killed, combo multiplier for rapid kills, wave clear bonuses',
    tips: [
      'Keep moving to avoid being surrounded',
      'Chain kills quickly to build combo multiplier',
      'Collect power-ups dropped by defeated enemies',
      'Red enemies are fast but weak, blue enemies are slow but tough'
    ]
  };
}
```

## Complete Example Game: Arena Fighter

A top-down arena brawler where the player fights waves of enemies. Enemies approach from screen edges. The player attacks with a sword swing using the `action` button.

```javascript
// Arena Fighter - 2D Action Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;

  let player, enemies, particles, powerUps, score, gameOver, wave, combo, comboTimer;
  let keys = {}, attackCooldown = 0, waveTimer = 0, spawnQueue = 0;
  let running = false, animFrame = null, lastTime = 0;
  let inputFrames = 0;

  function initPlayer() {
    return { x: W / 2, y: H / 2, w: 28, h: 28, health: 100, maxHealth: 100,
             lives: 3, speed: 3.5, attacking: false, attackTimer: 0, attackAngle: 0,
             invincible: 0, dx: 0, dy: 1 };
  }

  function spawnEnemy(type) {
    const side = Math.random() * 4 | 0;
    let x, y;
    if (side === 0) { x = -20; y = Math.random() * H; }
    else if (side === 1) { x = W + 20; y = Math.random() * H; }
    else if (side === 2) { x = Math.random() * W; y = -20; }
    else { x = Math.random() * W; y = H + 20; }
    const types = {
      fast: { w: 18, h: 18, health: 30, speed: 2.8, color: '#e74c3c', points: 100 },
      tank: { w: 36, h: 36, health: 120, speed: 1.2, color: '#3498db', points: 250 },
      normal: { w: 24, h: 24, health: 60, speed: 2, color: '#e67e22', points: 150 }
    };
    const t = types[type] || types.normal;
    return { x, y, w: t.w, h: t.h, health: t.health, maxHealth: t.health,
             speed: t.speed, color: t.color, points: t.points, type, flash: 0 };
  }

  function spawnWave(n) {
    wave = n;
    const count = 3 + n * 2;
    spawnQueue = count;
    waveTimer = 0;
  }

  function spawnPowerUp(x, y) {
    if (Math.random() < 0.3) {
      const types = ['heal', 'speed', 'power'];
      const type = types[Math.random() * types.length | 0];
      const colors = { heal: '#2ecc71', speed: '#f1c40f', power: '#9b59b6' };
      powerUps.push({ x, y, w: 16, h: 16, type, color: colors[type], life: 300 });
    }
  }

  function addParticles(x, y, color, count) {
    for (let i = 0; i < count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = 1 + Math.random() * 3;
      particles.push({ x, y, vx: Math.cos(angle) * speed, vy: Math.sin(angle) * speed,
                        life: 20 + Math.random() * 20, color, size: 2 + Math.random() * 3 });
    }
  }

  function rectsOverlap(a, b) {
    return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
  }

  function update(dt) {
    if (gameOver || !running) return;
    // Decrement input frame counter and release keys when expired
    if (inputFrames > 0) {
      inputFrames--;
      if (inputFrames <= 0) {
        keys.up = false; keys.down = false;
        keys.left = false; keys.right = false;
      }
    }
    // Player movement
    let mx = 0, my = 0;
    if (keys.left) mx -= 1;
    if (keys.right) mx += 1;
    if (keys.up) my -= 1;
    if (keys.down) my += 1;
    if (mx || my) {
      const len = Math.sqrt(mx * mx + my * my);
      mx /= len; my /= len;
      player.dx = mx; player.dy = my;
    }
    player.x = Math.max(0, Math.min(W - player.w, player.x + mx * player.speed));
    player.y = Math.max(0, Math.min(H - player.h, player.y + my * player.speed));

    // Attack cooldown
    if (attackCooldown > 0) attackCooldown--;
    if (player.attacking) {
      player.attackTimer--;
      if (player.attackTimer <= 0) player.attacking = false;
    }
    if (player.invincible > 0) player.invincible--;

    // Combo timer
    if (comboTimer > 0) { comboTimer--; } else { combo = 0; }

    // Spawn queued enemies
    waveTimer++;
    if (spawnQueue > 0 && waveTimer % 30 === 0) {
      const r = Math.random();
      const type = wave >= 3 && r < 0.25 ? 'tank' : (r < 0.5 ? 'fast' : 'normal');
      enemies.push(spawnEnemy(type));
      spawnQueue--;
    }

    // Next wave check
    if (enemies.length === 0 && spawnQueue === 0) {
      spawnWave(wave + 1);
      score += wave * 500;
    }

    // Enemy AI
    for (let i = enemies.length - 1; i >= 0; i--) {
      const e = enemies[i];
      const dx = player.x - e.x, dy = player.y - e.y;
      const dist = Math.sqrt(dx * dx + dy * dy) || 1;
      e.x += (dx / dist) * e.speed;
      e.y += (dy / dist) * e.speed;
      if (e.flash > 0) e.flash--;

      // Attack hitbox check
      if (player.attacking) {
        const atkRange = 50;
        const ax = player.x + player.w / 2 + player.dx * atkRange - atkRange / 2;
        const ay = player.y + player.h / 2 + player.dy * atkRange - atkRange / 2;
        const atkBox = { x: ax, y: ay, w: atkRange, h: atkRange };
        if (rectsOverlap(atkBox, e)) {
          e.health -= 35;
          e.flash = 6;
          addParticles(e.x + e.w / 2, e.y + e.h / 2, e.color, 5);
          if (e.health <= 0) {
            combo++;
            comboTimer = 90;
            const mult = Math.min(combo, 10);
            score += e.points * mult;
            addParticles(e.x + e.w / 2, e.y + e.h / 2, e.color, 12);
            spawnPowerUp(e.x, e.y);
            enemies.splice(i, 1);
            continue;
          }
        }
      }

      // Enemy-player collision (damage)
      if (player.invincible <= 0 && rectsOverlap(player, e)) {
        player.health -= 20;
        player.invincible = 45;
        addParticles(player.x + player.w / 2, player.y + player.h / 2, '#e74c3c', 8);
        if (player.health <= 0) {
          player.lives--;
          if (player.lives <= 0) {
            gameOver = true;
          } else {
            player.health = player.maxHealth;
            player.invincible = 90;
          }
        }
      }
    }

    // Power-ups
    for (let i = powerUps.length - 1; i >= 0; i--) {
      const p = powerUps[i];
      p.life--;
      if (p.life <= 0) { powerUps.splice(i, 1); continue; }
      if (rectsOverlap(player, p)) {
        if (p.type === 'heal') player.health = Math.min(player.maxHealth, player.health + 30);
        else if (p.type === 'speed') player.speed = Math.min(6, player.speed + 0.5);
        else if (p.type === 'power') { /* future: damage boost */ }
        addParticles(p.x, p.y, p.color, 8);
        powerUps.splice(i, 1);
      }
    }

    // Particles
    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i];
      p.x += p.vx; p.y += p.vy; p.life--;
      p.vx *= 0.95; p.vy *= 0.95;
      if (p.life <= 0) particles.splice(i, 1);
    }
  }

  function draw() {
    ctx.fillStyle = '#1a1a2e';
    ctx.fillRect(0, 0, W, H);

    // Grid floor
    ctx.strokeStyle = '#16213e';
    ctx.lineWidth = 1;
    for (let x = 0; x < W; x += 40) { ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, H); ctx.stroke(); }
    for (let y = 0; y < H; y += 40) { ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(W, y); ctx.stroke(); }

    // Power-ups
    powerUps.forEach(p => {
      ctx.globalAlpha = p.life < 60 ? 0.3 + 0.7 * (p.life / 60) : 1;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, p.w, p.h);
      ctx.globalAlpha = 1;
    });

    // Enemies
    enemies.forEach(e => {
      ctx.fillStyle = e.flash > 0 ? '#fff' : e.color;
      ctx.fillRect(e.x, e.y, e.w, e.h);
      // Health bar
      const hpPct = e.health / e.maxHealth;
      ctx.fillStyle = '#333';
      ctx.fillRect(e.x, e.y - 8, e.w, 4);
      ctx.fillStyle = hpPct > 0.5 ? '#2ecc71' : '#e74c3c';
      ctx.fillRect(e.x, e.y - 8, e.w * hpPct, 4);
    });

    // Player
    if (player.invincible <= 0 || (player.invincible % 6 < 3)) {
      ctx.fillStyle = '#00d2ff';
      ctx.fillRect(player.x, player.y, player.w, player.h);
      // Direction indicator
      ctx.fillStyle = '#fff';
      const cx = player.x + player.w / 2, cy = player.y + player.h / 2;
      ctx.beginPath();
      ctx.arc(cx + player.dx * 12, cy + player.dy * 12, 4, 0, Math.PI * 2);
      ctx.fill();
    }

    // Attack swing visual
    if (player.attacking) {
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 3;
      const cx = player.x + player.w / 2, cy = player.y + player.h / 2;
      const angle = Math.atan2(player.dy, player.dx);
      ctx.beginPath();
      ctx.arc(cx, cy, 45, angle - 0.8, angle + 0.8);
      ctx.stroke();
    }

    // Particles
    particles.forEach(p => {
      ctx.globalAlpha = p.life / 40;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x - p.size / 2, p.y - p.size / 2, p.size, p.size);
    });
    ctx.globalAlpha = 1;

    // HUD
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 18px monospace';
    ctx.fillText('SCORE: ' + score, 10, 28);
    ctx.fillText('WAVE: ' + wave, 10, 52);
    if (combo > 1) {
      ctx.fillStyle = '#f1c40f';
      ctx.fillText('COMBO x' + combo, 10, 76);
    }
    // Health bar
    ctx.fillStyle = '#333';
    ctx.fillRect(W - 210, 10, 200, 20);
    ctx.fillStyle = player.health > 30 ? '#2ecc71' : '#e74c3c';
    ctx.fillRect(W - 210, 10, 200 * (player.health / player.maxHealth), 20);
    ctx.fillStyle = '#fff';
    ctx.font = '12px monospace';
    ctx.fillText('HP: ' + player.health + '/' + player.maxHealth, W - 200, 25);
    // Lives
    ctx.fillStyle = '#e74c3c';
    for (let i = 0; i < player.lives; i++) {
      ctx.fillRect(W - 210 + i * 22, 36, 16, 16);
    }

    // Game over overlay
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#e74c3c';
      ctx.font = 'bold 48px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('GAME OVER', W / 2, H / 2 - 30);
      ctx.fillStyle = '#fff';
      ctx.font = '24px monospace';
      ctx.fillText('Score: ' + score + '  |  Wave: ' + wave, W / 2, H / 2 + 20);
      ctx.font = '16px monospace';
      ctx.fillText('Press START to play again', W / 2, H / 2 + 60);
      ctx.textAlign = 'left';
    }

    if (!running) {
      ctx.fillStyle = 'rgba(0,0,0,0.8)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#00d2ff';
      ctx.font = 'bold 40px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('ARENA FIGHTER', W / 2, H / 2 - 40);
      ctx.fillStyle = '#fff';
      ctx.font = '18px monospace';
      ctx.fillText('Arrow keys to move, SPACE to attack', W / 2, H / 2 + 10);
      ctx.fillText('Press START to begin', W / 2, H / 2 + 50);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop(time) {
    const dt = Math.min((time - lastTime) / 16.67, 3);
    lastTime = time;
    update(dt);
    draw();
    animFrame = requestAnimationFrame(gameLoop);
  }

  function handleKey(e) {
    const map = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left',
                  ArrowRight: 'right', ' ': 'action', Enter: 'start' };
    const action = map[e.key];
    if (action) { e.preventDefault(); game.sendInput(action); }
  }

  const game = {
    init() {
      canvas.width = W;
      canvas.height = H;
      document.addEventListener('keydown', e => { handleKey(e); const a = { ArrowUp:'up',ArrowDown:'down',ArrowLeft:'left',ArrowRight:'right' }; if(a[e.key]) keys[a[e.key]]=true; });
      document.addEventListener('keyup', e => { const a = { ArrowUp:'up',ArrowDown:'down',ArrowLeft:'left',ArrowRight:'right' }; if(a[e.key]) keys[a[e.key]]=false; });
      this.reset();
      lastTime = performance.now();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() {
      this.reset();
      running = true;
      spawnWave(1);
    },
    reset() {
      player = initPlayer();
      enemies = []; particles = []; powerUps = [];
      score = 0; gameOver = false; wave = 0; combo = 0; comboTimer = 0;
      running = false; keys = {}; attackCooldown = 0; spawnQueue = 0;
    },
    getState() {
      return {
        score, gameOver,
        health: player.health, maxHealth: player.maxHealth, lives: player.lives,
        wave, enemiesRemaining: enemies.length + spawnQueue,
        playerX: player.x, playerY: player.y,
        enemies: enemies.map(e => ({ x: e.x, y: e.y, type: e.type, health: e.health })),
        powerUps: powerUps.map(p => ({ x: p.x, y: p.y, type: p.type })),
        combo, attacking: player.attacking
      };
    },
    sendInput(action) {
      if (action === 'start') { if (gameOver || !running) this.start(); return true; }
      if (action === 'pause') return true;
      if (gameOver || !running) return false;
      if (action === 'up') { keys.up = true; inputFrames = 6; return true; }
      if (action === 'down') { keys.down = true; inputFrames = 6; return true; }
      if (action === 'left') { keys.left = true; inputFrames = 6; return true; }
      if (action === 'right') { keys.right = true; inputFrames = 6; return true; }
      if (action === 'action' && attackCooldown <= 0) {
        player.attacking = true;
        player.attackTimer = 12;
        attackCooldown = 20;
        return true;
      }
      return false;
    },
    getMeta() {
      return {
        name: 'Arena Fighter',
        description: 'Survive waves of enemies in a top-down arena brawler. Attack foes, collect power-ups, and build combos.',
        controls: { up: 'Move up', down: 'Move down', left: 'Move left', right: 'Move right', action: 'Attack (sword swing)' },
        objective: 'Survive as many waves as possible and maximize your score',
        scoring: 'Points per enemy defeated (scaled by type), combo multiplier for rapid kills, wave clear bonus',
        tips: [
          'Keep moving to avoid being surrounded by enemies',
          'Chain kills quickly to build your combo multiplier (up to 10x)',
          'Red enemies are fast but fragile, blue enemies are slow but durable',
          'Collect green power-ups to restore health',
          'Attack has a cooldown, so time your swings carefully'
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
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `action` on submission
- [ ] Health/lives system functions correctly
- [ ] Enemy waves spawn and escalate in difficulty
- [ ] Collision detection works for attacks and enemy contact
- [ ] Game over state is reachable and displays correctly
- [ ] START action restarts the game from game-over or title screen
