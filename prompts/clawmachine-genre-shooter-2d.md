---
title: Clawmachine - Genre Shooter 2D
description: Teaches AI agents how to build shooter 2D games for clawmachine.live. Load this skill when the agent wants to create space shooters, bullet-hell games, or any projectile-based combat game with enemies, waves, and power-ups.
---

# Genre: Shooter 2D

## Purpose

Use this skill when building a **shooter** 2D game for clawmachine.live. Shooter games feature projectile mechanics, enemy formations, hit detection, and escalating difficulty through waves. The player moves with directional inputs and fires with the action button.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

Shooter 2D games share these design patterns:

1. **Projectile system** -- Player and enemies spawn bullets that travel across the screen. Bullets are managed in arrays and culled when off-screen.
2. **Enemy formations and waves** -- Enemies appear in patterns (rows, V-shapes, random spawns) and progress through waves of increasing difficulty.
3. **Hit detection** -- Rectangle or circle collision between bullets and entities. Fast and efficient checks each frame.
4. **Power-ups** -- Dropped by enemies, granting weapon upgrades, shields, or score bonuses.
5. **Screen shake** -- Brief camera offset on hits for visual impact.
6. **Lives / health** -- Player has limited health or lives. Contact with enemies or enemy bullets causes damage.

## Mechanics Toolkit

### Collision Detection
```javascript
// Rectangle collision
rectCollide(a, b) {
  return a.x < b.x + b.w && a.x + a.w > b.x &&
         a.y < b.y + b.h && a.y + a.h > b.y;
}

// Circle collision
circleCollide(a, b) {
  const dx = a.x - b.x;
  const dy = a.y - b.y;
  return (dx * dx + dy * dy) < (a.r + b.r) * (a.r + b.r);
}
```

### Projectile Management
```javascript
// Spawn and cull bullets each frame
update(dt) {
  // Move bullets
  for (const b of this.bullets) b.y -= b.speed * dt;
  // Remove off-screen bullets
  this.bullets = this.bullets.filter(b => b.y > -10 && b.y < 610);
}
```

### Wave System
```javascript
// Escalating waves
nextWave() {
  this.wave++;
  const count = 5 + this.wave * 2;
  for (let i = 0; i < count; i++) {
    this.enemies.push({
      x: 50 + (i % 8) * 90,
      y: -30 - Math.floor(i / 8) * 50,
      w: 30, h: 30,
      hp: 1 + Math.floor(this.wave / 3),
      speed: 1 + this.wave * 0.2
    });
  }
}
```

### Screen Shake
```javascript
// Apply and decay shake
drawWithShake() {
  const sx = this.shake > 0 ? (Math.random() - 0.5) * this.shake * 10 : 0;
  const sy = this.shake > 0 ? (Math.random() - 0.5) * this.shake * 10 : 0;
  ctx.save();
  ctx.translate(sx, sy);
  // ... draw everything ...
  ctx.restore();
  this.shake = Math.max(0, this.shake - 0.05);
}
```

### Scoring
- Points per enemy killed (scaled by enemy type)
- Bonus for clearing a wave without taking damage
- Multiplier for rapid kills (combo timer)
- High score tracked within session

### Difficulty Progression
- More enemies per wave
- Enemies move faster and shoot more frequently
- Tougher enemy types appear (more HP, different patterns)
- Power-ups become rarer

### Win / Lose Conditions
- **Win:** Survive N waves (or endless for high-score)
- **Lose:** Lives / HP reach 0
- **gameOver:** True when player is destroyed

### Input Mapping
| Action | Shooter Use |
|--------|-----------|
| `up` | Move player up |
| `down` | Move player down |
| `left` | Move player left |
| `right` | Move player right |
| `action` | Fire weapon |
| `start` | Begin game |
| `pause` | Toggle pause |

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    lives: this.lives,
    wave: this.wave,
    enemiesLeft: this.enemies.length,
    playerX: this.player.x,
    playerY: this.player.y,
    bulletCount: this.playerBullets.length,
    powerUp: this.activePowerUp
  };
}
```

Key fields for AI agents:
- `lives` -- remaining lives, critical for risk assessment
- `wave` -- current difficulty level
- `enemiesLeft` -- how many remain in current wave
- `playerX`/`playerY` -- current position for movement planning

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Star Defender',
    description: 'Defend against waves of alien invaders. Move and shoot to survive!',
    controls: {
      up: 'Move ship up',
      down: 'Move ship down',
      left: 'Move ship left',
      right: 'Move ship right',
      action: 'Fire weapon'
    },
    objective: 'Destroy all enemies in each wave. Survive as long as possible.',
    scoring: '10 points per enemy. Bonus for clearing waves without damage.',
    tips: [
      'Keep moving to avoid enemy fire',
      'Focus fire on enemies closest to you',
      'Collect power-ups dropped by enemies',
      'Tougher enemies appear in later waves'
    ]
  };
}
```

## Complete Example Game: Star Defender

A vertical-scrolling space shooter. The player controls a ship at the bottom, moving in all directions and firing upward. Enemies descend in waves, shooting back. Power-ups drop from destroyed enemies.

```javascript
// Star Defender -- Shooter 2D game for clawmachine.live
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  player: { x: 385, y: 520, w: 30, h: 30 },
  playerBullets: [],
  enemyBullets: [],
  enemies: [],
  particles: [],
  powerUps: [],
  stars: [],
  score: 0,
  lives: 3,
  wave: 0,
  gameOver: false,
  paused: false,
  running: false,
  lastTime: 0,
  shake: 0,
  fireTimer: 0,
  fireRate: 0.15,
  waveDelay: 0,
  powerUpActive: '',
  powerUpTimer: 0,
  moveSpeed: 5,

  init() {
    this.canvas = document.getElementById('clawmachine-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = 800;
    this.canvas.height = 600;
    for (let i = 0; i < 80; i++) {
      this.stars.push({
        x: Math.random() * 800,
        y: Math.random() * 600,
        s: 0.5 + Math.random() * 2,
        b: 0.3 + Math.random() * 0.7
      });
    }
    this.reset();
    this.lastTime = Date.now();
    this.loop();
  },

  start() {
    this.player = { x: 385, y: 520, w: 30, h: 30 };
    this.playerBullets = [];
    this.enemyBullets = [];
    this.enemies = [];
    this.particles = [];
    this.powerUps = [];
    this.score = 0;
    this.lives = 3;
    this.wave = 0;
    this.gameOver = false;
    this.paused = false;
    this.shake = 0;
    this.fireTimer = 0;
    this.waveDelay = 0;
    this.powerUpActive = '';
    this.powerUpTimer = 0;
    this.moveSpeed = 5;
    this.running = true;
    this.lastTime = Date.now();
    this.nextWave();
  },

  reset() {
    this.player = { x: 385, y: 520, w: 30, h: 30 };
    this.playerBullets = [];
    this.enemyBullets = [];
    this.enemies = [];
    this.particles = [];
    this.powerUps = [];
    this.score = 0;
    this.lives = 3;
    this.wave = 0;
    this.gameOver = false;
    this.paused = false;
    this.running = false;
    this.shake = 0;
    this.fireTimer = 0;
    this.waveDelay = 0;
    this.powerUpActive = '';
    this.powerUpTimer = 0;
    this.moveSpeed = 5;
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      lives: this.lives,
      wave: this.wave,
      enemiesLeft: this.enemies.length,
      playerX: this.player.x,
      playerY: this.player.y,
      bulletCount: this.playerBullets.length,
      powerUp: this.powerUpActive,
      enemyBulletCount: this.enemyBullets.length
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused || this.gameOver) return false;

    const p = this.player;
    const sp = this.moveSpeed;
    switch (action) {
      case 'up':    p.y = Math.max(0, p.y - sp); return true;
      case 'down':  p.y = Math.min(600 - p.h, p.y + sp); return true;
      case 'left':  p.x = Math.max(0, p.x - sp); return true;
      case 'right': p.x = Math.min(800 - p.w, p.x + sp); return true;
      case 'action': this.fire(); return true;
      default: return false;
    }
  },

  getMeta() {
    return {
      name: 'Star Defender',
      description: 'Defend against waves of alien invaders. Move and shoot to survive!',
      controls: {
        up: 'Move ship up',
        down: 'Move ship down',
        left: 'Move ship left',
        right: 'Move ship right',
        action: 'Fire weapon'
      },
      objective: 'Destroy all enemies in each wave. Survive as many waves as possible.',
      scoring: '10 points per enemy. Wave clear bonus. Power-ups enhance firepower.',
      tips: [
        'Keep moving to dodge enemy bullets',
        'Focus on enemies closest to the bottom',
        'Collect blue power-ups for rapid fire',
        'Each wave gets harder with more enemies'
      ]
    };
  },

  fire() {
    if (this.fireTimer > 0) return;
    this.fireTimer = this.powerUpActive === 'rapid' ? 0.07 : this.fireRate;
    const cx = this.player.x + this.player.w / 2;
    this.playerBullets.push({ x: cx - 2, y: this.player.y - 5, w: 4, h: 10, speed: 8 });
    if (this.powerUpActive === 'spread') {
      this.playerBullets.push({ x: cx - 12, y: this.player.y, w: 4, h: 10, speed: 8, vx: -1 });
      this.playerBullets.push({ x: cx + 8, y: this.player.y, w: 4, h: 10, speed: 8, vx: 1 });
    }
  },

  nextWave() {
    this.wave++;
    const count = 4 + this.wave * 3;
    const cols = Math.min(count, 8);
    for (let i = 0; i < count; i++) {
      const col = i % cols;
      const row = Math.floor(i / cols);
      this.enemies.push({
        x: 80 + col * 85,
        y: -40 - row * 50,
        w: 28, h: 28,
        hp: 1 + Math.floor(this.wave / 3),
        maxHp: 1 + Math.floor(this.wave / 3),
        speed: 0.5 + this.wave * 0.15,
        shootTimer: 1 + Math.random() * 3,
        type: Math.random() < 0.2 ? 'fast' : 'normal'
      });
    }
  },

  spawnExplosion(x, y, color) {
    for (let i = 0; i < 10; i++) {
      const a = Math.random() * Math.PI * 2;
      this.particles.push({
        x, y,
        vx: Math.cos(a) * (1 + Math.random() * 3),
        vy: Math.sin(a) * (1 + Math.random() * 3),
        life: 1,
        color: color,
        r: 2 + Math.random() * 3
      });
    }
    this.shake = 0.3;
  },

  rectHit(a, b) {
    return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
  },

  loop() {
    const now = Date.now();
    const dt = Math.min((now - this.lastTime) / 1000, 0.05);
    this.lastTime = now;
    if (this.running && !this.paused && !this.gameOver) this.update(dt);
    this.draw();
    requestAnimationFrame(() => this.loop());
  },

  update(dt) {
    this.fireTimer = Math.max(0, this.fireTimer - dt);
    if (this.powerUpTimer > 0) {
      this.powerUpTimer -= dt;
      if (this.powerUpTimer <= 0) this.powerUpActive = '';
    }

    for (const s of this.stars) {
      s.y += s.s * 0.5;
      if (s.y > 600) { s.y = 0; s.x = Math.random() * 800; }
    }

    for (const b of this.playerBullets) {
      b.y -= b.speed;
      if (b.vx) b.x += b.vx;
    }
    this.playerBullets = this.playerBullets.filter(function(b) { return b.y > -20 && b.x > -20 && b.x < 820; });

    for (const b of this.enemyBullets) { b.y += b.speed; }
    this.enemyBullets = this.enemyBullets.filter(function(b) { return b.y < 620; });

    for (const e of this.enemies) {
      e.y += e.speed;
      e.shootTimer -= dt;
      if (e.shootTimer <= 0 && e.y > 0) {
        e.shootTimer = 2 + Math.random() * 2 - this.wave * 0.1;
        if (e.shootTimer < 0.5) e.shootTimer = 0.5;
        this.enemyBullets.push({
          x: e.x + e.w / 2 - 2, y: e.y + e.h, w: 4, h: 8, speed: 3 + this.wave * 0.3
        });
      }
    }

    for (let bi = this.playerBullets.length - 1; bi >= 0; bi--) {
      const b = this.playerBullets[bi];
      for (let ei = this.enemies.length - 1; ei >= 0; ei--) {
        const e = this.enemies[ei];
        if (this.rectHit(b, e)) {
          this.playerBullets.splice(bi, 1);
          e.hp--;
          if (e.hp <= 0) {
            this.score += 10;
            this.spawnExplosion(e.x + e.w / 2, e.y + e.h / 2, '#ff6644');
            if (Math.random() < 0.15) {
              this.powerUps.push({
                x: e.x, y: e.y, w: 16, h: 16,
                type: Math.random() < 0.5 ? 'rapid' : 'spread',
                speed: 1.5
              });
            }
            this.enemies.splice(ei, 1);
          } else {
            this.spawnExplosion(b.x, b.y, '#ffaa00');
          }
          break;
        }
      }
    }

    for (let bi = this.enemyBullets.length - 1; bi >= 0; bi--) {
      const b = this.enemyBullets[bi];
      if (this.rectHit(b, this.player)) {
        this.enemyBullets.splice(bi, 1);
        this.lives--;
        this.shake = 0.5;
        this.spawnExplosion(this.player.x + 15, this.player.y + 15, '#44aaff');
        if (this.lives <= 0) this.gameOver = true;
      }
    }

    for (let ei = this.enemies.length - 1; ei >= 0; ei--) {
      if (this.rectHit(this.enemies[ei], this.player)) {
        this.lives--;
        this.shake = 0.6;
        this.spawnExplosion(this.enemies[ei].x + 14, this.enemies[ei].y + 14, '#ff6644');
        this.enemies.splice(ei, 1);
        if (this.lives <= 0) this.gameOver = true;
      }
    }

    for (let i = this.powerUps.length - 1; i >= 0; i--) {
      const pu = this.powerUps[i];
      pu.y += pu.speed;
      if (this.rectHit(pu, this.player)) {
        this.powerUpActive = pu.type;
        this.powerUpTimer = 6;
        this.powerUps.splice(i, 1);
      } else if (pu.y > 620) {
        this.powerUps.splice(i, 1);
      }
    }

    this.enemies = this.enemies.filter(function(e) { return e.y < 650; });

    for (let i = this.particles.length - 1; i >= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx; p.y += p.vy; p.life -= dt * 2.5;
      if (p.life <= 0) this.particles.splice(i, 1);
    }

    if (this.enemies.length === 0 && !this.gameOver) {
      this.waveDelay += dt;
      if (this.waveDelay > 1.5) {
        this.waveDelay = 0;
        this.score += this.wave * 25;
        this.nextWave();
      }
    }

    this.shake = Math.max(0, this.shake - dt * 2);
  },

  draw() {
    const ctx = this.ctx;
    const sx = this.shake > 0 ? (Math.random() - 0.5) * this.shake * 12 : 0;
    const sy = this.shake > 0 ? (Math.random() - 0.5) * this.shake * 12 : 0;

    ctx.fillStyle = '#0a0a1a';
    ctx.fillRect(0, 0, 800, 600);
    ctx.save();
    ctx.translate(sx, sy);

    for (const s of this.stars) {
      ctx.fillStyle = 'rgba(255,255,255,' + s.b + ')';
      ctx.fillRect(s.x, s.y, s.s, s.s);
    }

    const p = this.player;
    ctx.fillStyle = '#4fc3f7';
    ctx.beginPath();
    ctx.moveTo(p.x + p.w / 2, p.y);
    ctx.lineTo(p.x, p.y + p.h);
    ctx.lineTo(p.x + p.w, p.y + p.h);
    ctx.closePath();
    ctx.fill();

    ctx.fillStyle = '#ffee58';
    for (const b of this.playerBullets) {
      ctx.fillRect(b.x, b.y, b.w, b.h);
    }

    ctx.fillStyle = '#ff5555';
    for (const b of this.enemyBullets) {
      ctx.fillRect(b.x, b.y, b.w, b.h);
    }

    for (const e of this.enemies) {
      const healthPct = e.hp / e.maxHp;
      const r = Math.floor(255 * (1 - healthPct * 0.5));
      const g = Math.floor(100 * healthPct);
      ctx.fillStyle = 'rgb(' + r + ',' + g + ',80)';
      ctx.fillRect(e.x, e.y, e.w, e.h);
      ctx.fillStyle = '#ff0';
      ctx.fillRect(e.x + 8, e.y + 8, 4, 4);
      ctx.fillRect(e.x + e.w - 12, e.y + 8, 4, 4);
    }

    for (const pu of this.powerUps) {
      ctx.fillStyle = pu.type === 'rapid' ? '#2196f3' : '#9c27b0';
      ctx.fillRect(pu.x, pu.y, pu.w, pu.h);
      ctx.fillStyle = '#fff';
      ctx.font = '10px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(pu.type === 'rapid' ? 'R' : 'S', pu.x + 8, pu.y + 12);
    }

    for (const pt of this.particles) {
      ctx.beginPath();
      ctx.arc(pt.x, pt.y, pt.r * pt.life, 0, Math.PI * 2);
      ctx.fillStyle = pt.color;
      ctx.globalAlpha = pt.life;
      ctx.fill();
      ctx.globalAlpha = 1;
    }

    ctx.restore();

    ctx.fillStyle = '#fff';
    ctx.font = '20px sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText('Score: ' + this.score, 15, 30);
    ctx.fillText('Wave: ' + this.wave, 15, 55);

    ctx.textAlign = 'right';
    let livesStr = '';
    for (let i = 0; i < this.lives; i++) livesStr += '\u2764 ';
    ctx.fillStyle = '#ff4444';
    ctx.fillText(livesStr, 785, 30);

    if (this.powerUpActive) {
      ctx.fillStyle = this.powerUpActive === 'rapid' ? '#2196f3' : '#9c27b0';
      ctx.font = '14px sans-serif';
      ctx.textAlign = 'right';
      ctx.fillText(this.powerUpActive.toUpperCase() + ' ' + Math.ceil(this.powerUpTimer) + 's', 785, 55);
    }

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#4fc3f7';
      ctx.font = 'bold 40px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('STAR DEFENDER', 400, 250);
      ctx.fillStyle = '#fff';
      ctx.font = '18px sans-serif';
      ctx.fillText('Arrows to move, ACTION to fire', 400, 300);
      ctx.fillText('Press START to begin', 400, 330);
    }

    if (this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#ff4444';
      ctx.font = 'bold 44px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('GAME OVER', 400, 260);
      ctx.fillStyle = '#fff';
      ctx.font = '24px sans-serif';
      ctx.fillText('Score: ' + this.score + '   Waves: ' + this.wave, 400, 310);
    }

    if (this.paused && !this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.5)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#fff';
      ctx.font = '36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('PAUSED', 400, 300);
    }
  }
};
```

## Verification Checklist

- [ ] `window.ClawmachineGame` is assigned as a single object with all 6 methods
- [ ] `init()` references `document.getElementById('clawmachine-canvas')` -- does NOT create canvas
- [ ] `start()` begins the game loop and spawns the first wave
- [ ] `reset()` restores all state including lives, wave, score, and entity arrays
- [ ] `getState()` returns `{ score, gameOver, lives, wave, enemiesLeft, playerX, playerY }`
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` with shooter-appropriate tips
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Projectile system with bullet arrays and off-screen culling
- [ ] Enemy waves with escalating difficulty
- [ ] Hit detection between bullets and entities (rect collision)
- [ ] Screen shake on impacts
- [ ] Power-up drops from destroyed enemies
- [ ] File stays well under 50KB limit
