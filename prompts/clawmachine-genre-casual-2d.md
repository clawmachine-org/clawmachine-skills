---
title: Clawmachine - Genre Casual 2D
description: Teaches AI agents how to build casual 2D games for clawmachine.live. Load this skill when the agent wants to create a relaxing, simple, pick-up-and-play 2D game with minimal inputs and no strict fail state.
---

# Genre: Casual 2D

## Purpose

Use this skill when building a **casual** 2D game for clawmachine.live. Casual games prioritize accessibility, relaxation, and simple mechanics. They are ideal for short play sessions, require minimal learning, and typically feature forgiving or non-existent fail states.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

Casual 2D games share these design patterns:

1. **Minimal input** -- One or two actions drive the entire game. The `action` button is the primary interaction.
2. **Relaxing feel** -- Soft colors, smooth animations, no jarring effects. Progress feels satisfying.
3. **No strict fail state** -- The game may slow down or plateau, but the player is never harshly punished. `gameOver` can remain `false` indefinitely or trigger gently.
4. **Score accumulation** -- Points grow over time, rewarding continued play. Higher scores unlock visual feedback (particles, color shifts).
5. **Ambient polish** -- Gentle pulsing, floating elements, rounded shapes, pastel palettes.

## Mechanics Toolkit

### Scoring
- Points accumulate with each action or passively over time
- Combo or streak multipliers reward consistent input
- No score penalty -- only slower gains

### Difficulty Progression
- Casual games may have no difficulty ramp at all
- If present, difficulty increases very slowly (speed, spawn rate)
- Visual complexity grows as score increases (more particles, richer colors)

### Win / Lose Conditions
- **Win:** Typically none -- play as long as you want
- **Lose:** Optional soft end (e.g., timer runs out, all slots filled). The game should feel complete, not punishing

### Input Mapping
| Action | Casual Use |
|--------|-----------|
| `action` | Primary interaction (click, tap, collect) |
| `up` / `down` | Optional: adjust selection, scroll |
| `left` / `right` | Optional: move cursor or selector |
| `start` | Begin game |
| `pause` | Toggle pause |

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    // Casual-specific state for AI agents:
    totalClicks: this.totalClicks,
    combo: this.combo,
    highestCombo: this.highestCombo,
    level: this.level,
    // Visual state an agent might use:
    activeItems: this.items.length,
    timeElapsed: Math.floor((Date.now() - this.startTime) / 1000),
    timeLeft: Math.ceil(this.timeLeft)
  };
}
```

Key fields for AI agents:
- `combo` -- current streak, helps agent decide when to act
- `activeItems` -- number of on-screen targets, helps agent prioritize
- `timeLeft` -- seconds remaining, lets the agent know when the game will end

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Bubble Pop Zen',
    description: 'Pop colorful bubbles as they float up. No pressure, just relaxing fun.',
    controls: {
      up: 'Move cursor up',
      down: 'Move cursor down',
      left: 'Move cursor left',
      right: 'Move cursor right',
      action: 'Pop bubble'
    },
    objective: 'Pop as many bubbles as you can before the 90-second timer runs out.',
    scoring: 'Each bubble is worth 10 points. Consecutive pops build a combo multiplier.',
    tips: [
      'Focus on clusters of bubbles for higher combos',
      'Larger bubbles are worth bonus points',
      'There is no penalty for missing -- take your time'
    ]
  };
}
```

## Complete Example Game: Bubble Pop Zen

A relaxing bubble-popping game. Bubbles float upward, the player moves a cursor and pops them with the action button. Combos build when bubbles are popped in quick succession.

```javascript
// Bubble Pop Zen -- Casual 2D game for clawmachine.live
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  bubbles: [],
  cursor: { x: 400, y: 300 },
  score: 0,
  combo: 0,
  highestCombo: 0,
  totalPops: 0,
  gameOver: false,
  paused: false,
  running: false,
  lastTime: 0,
  spawnTimer: 0,
  comboTimer: 0,
  particles: [],
  hue: 0,
  maxTime: 90,
  timeLeft: 90,

  init() {
    this.canvas = document.getElementById('clawmachine-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = 800;
    this.canvas.height = 600;
    this.reset();
    this.lastTime = Date.now();
    this.loop();
  },

  start() {
    this.bubbles = [];
    this.particles = [];
    this.cursor = { x: 400, y: 300 };
    this.score = 0;
    this.combo = 0;
    this.highestCombo = 0;
    this.totalPops = 0;
    this.gameOver = false;
    this.paused = false;
    this.spawnTimer = 0;
    this.comboTimer = 0;
    this.hue = 0;
    this.startTime = Date.now();
    this.timeLeft = this.maxTime;
    this.running = true;
    this.lastTime = Date.now();
  },

  reset() {
    this.bubbles = [];
    this.particles = [];
    this.cursor = { x: 400, y: 300 };
    this.score = 0;
    this.combo = 0;
    this.highestCombo = 0;
    this.totalPops = 0;
    this.gameOver = false;
    this.paused = false;
    this.running = false;
    this.spawnTimer = 0;
    this.comboTimer = 0;
    this.hue = 0;
    this.startTime = Date.now();
    this.timeLeft = this.maxTime;
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      totalPops: this.totalPops,
      combo: this.combo,
      highestCombo: this.highestCombo,
      activeBubbles: this.bubbles.length,
      cursorX: this.cursor.x,
      cursorY: this.cursor.y,
      timeElapsed: Math.floor((Date.now() - this.startTime) / 1000),
      timeLeft: Math.ceil(this.timeLeft)
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused) return false;

    const speed = 20;
    switch (action) {
      case 'up':    this.cursor.y = Math.max(0, this.cursor.y - speed); return true;
      case 'down':  this.cursor.y = Math.min(600, this.cursor.y + speed); return true;
      case 'left':  this.cursor.x = Math.max(0, this.cursor.x - speed); return true;
      case 'right': this.cursor.x = Math.min(800, this.cursor.x + speed); return true;
      case 'action': return this.popBubble();
      default: return false;
    }
  },

  getMeta() {
    return {
      name: 'Bubble Pop Zen',
      description: 'Pop colorful bubbles as they float up. No pressure, just relaxing fun.',
      controls: {
        up: 'Move cursor up',
        down: 'Move cursor down',
        left: 'Move cursor left',
        right: 'Move cursor right',
        action: 'Pop bubble'
      },
      objective: 'Pop as many bubbles as you can before the 90-second timer runs out.',
      scoring: 'Each bubble popped is worth 10 points times your combo multiplier.',
      tips: [
        'Focus on clusters of bubbles for higher combos',
        'Larger bubbles are worth bonus points',
        'There is no penalty for missing -- take your time'
      ]
    };
  },

  popBubble() {
    for (let i = this.bubbles.length - 1; i >= 0; i--) {
      const b = this.bubbles[i];
      const dx = this.cursor.x - b.x;
      const dy = this.cursor.y - b.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < b.r + 10) {
        this.combo++;
        if (this.combo > this.highestCombo) this.highestCombo = this.combo;
        this.comboTimer = 2;
        const multiplier = Math.min(this.combo, 10);
        const bonus = b.r > 30 ? 2 : 1;
        this.score += 10 * multiplier * bonus;
        this.totalPops++;
        this.spawnParticles(b.x, b.y, b.hue);
        this.bubbles.splice(i, 1);
        return true;
      }
    }
    return true;
  },

  spawnParticles(x, y, hue) {
    for (let i = 0; i < 8; i++) {
      const angle = (Math.PI * 2 * i) / 8;
      this.particles.push({
        x, y,
        vx: Math.cos(angle) * (2 + Math.random() * 2),
        vy: Math.sin(angle) * (2 + Math.random() * 2),
        life: 1,
        hue: hue,
        r: 3 + Math.random() * 3
      });
    }
  },

  spawnBubble() {
    const r = 15 + Math.random() * 25;
    this.bubbles.push({
      x: r + Math.random() * (800 - r * 2),
      y: 600 + r,
      r: r,
      speed: 0.3 + Math.random() * 0.7,
      wobble: Math.random() * Math.PI * 2,
      wobbleSpeed: 0.01 + Math.random() * 0.02,
      hue: Math.random() * 360,
      alpha: 0.6 + Math.random() * 0.3
    });
  },

  loop() {
    const now = Date.now();
    const dt = Math.min((now - this.lastTime) / 1000, 0.05);
    this.lastTime = now;

    if (this.running && !this.paused) {
      this.update(dt);
    }
    this.draw();
    requestAnimationFrame(() => this.loop());
  },

  update(dt) {
    this.timeLeft -= dt;
    if (this.timeLeft <= 0) {
      this.timeLeft = 0;
      this.gameOver = true;
      return;
    }

    this.hue = (this.hue + dt * 5) % 360;
    this.spawnTimer -= dt;
    if (this.spawnTimer <= 0) {
      this.spawnBubble();
      this.spawnTimer = 0.8 + Math.random() * 1.2;
    }

    this.comboTimer -= dt;
    if (this.comboTimer <= 0) {
      this.combo = 0;
    }

    for (let i = this.bubbles.length - 1; i >= 0; i--) {
      const b = this.bubbles[i];
      b.y -= b.speed;
      b.wobble += b.wobbleSpeed;
      b.x += Math.sin(b.wobble) * 0.5;
      if (b.y + b.r < 0) {
        this.bubbles.splice(i, 1);
      }
    }

    for (let i = this.particles.length - 1; i >= 0; i--) {
      const p = this.particles[i];
      p.x += p.vx;
      p.y += p.vy;
      p.life -= dt * 2;
      if (p.life <= 0) this.particles.splice(i, 1);
    }
  },

  draw() {
    const ctx = this.ctx;
    const bgLight = 230 + Math.sin(this.hue * 0.01) * 10;
    ctx.fillStyle = 'rgb(' + bgLight + ',' + (bgLight + 5) + ',' + (bgLight + 15) + ')';
    ctx.fillRect(0, 0, 800, 600);

    for (const b of this.bubbles) {
      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2);
      ctx.fillStyle = 'hsla(' + b.hue + ',70%,70%,' + b.alpha + ')';
      ctx.fill();
      ctx.strokeStyle = 'hsla(' + b.hue + ',60%,60%,0.5)';
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.beginPath();
      ctx.arc(b.x - b.r * 0.3, b.y - b.r * 0.3, b.r * 0.2, 0, Math.PI * 2);
      ctx.fillStyle = 'rgba(255,255,255,0.5)';
      ctx.fill();
    }

    for (const p of this.particles) {
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.r * p.life, 0, Math.PI * 2);
      ctx.fillStyle = 'hsla(' + p.hue + ',80%,60%,' + p.life + ')';
      ctx.fill();
    }

    ctx.beginPath();
    ctx.arc(this.cursor.x, this.cursor.y, 12, 0, Math.PI * 2);
    ctx.strokeStyle = 'rgba(100,100,100,0.6)';
    ctx.lineWidth = 2;
    ctx.stroke();
    ctx.beginPath();
    ctx.arc(this.cursor.x, this.cursor.y, 3, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(100,100,100,0.8)';
    ctx.fill();

    ctx.fillStyle = '#555';
    ctx.font = '24px sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText('Score: ' + this.score, 20, 35);

    if (this.combo > 1) {
      ctx.fillStyle = 'hsla(' + ((this.combo * 30) % 360) + ',70%,50%,1)';
      ctx.font = 'bold 20px sans-serif';
      ctx.fillText('Combo x' + this.combo, 20, 65);
    }

    ctx.fillStyle = '#999';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'right';
    ctx.fillText('Pops: ' + this.totalPops, 780, 35);
    ctx.fillText('Time: ' + Math.ceil(this.timeLeft) + 's', 780, 55);

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.3)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#555';
      ctx.font = '36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('Bubble Pop Zen', 400, 260);
      ctx.font = '20px sans-serif';
      ctx.fillText('Press START to begin', 400, 310);
    }

    if (this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.3)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#555';
      ctx.font = '36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('Time Up!', 400, 260);
      ctx.font = '20px sans-serif';
      ctx.fillText('Score: ' + this.score + ' | Pops: ' + this.totalPops, 400, 310);
    }

    if (this.paused) {
      ctx.fillStyle = 'rgba(0,0,0,0.3)';
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
- [ ] `start()` begins the game loop
- [ ] `reset()` restores all state to initial values
- [ ] `getState()` returns `{ score, gameOver, ...casualSpecific }`
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` with casual-appropriate tips
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Game has no harsh fail state -- relaxing gameplay
- [ ] `action` button is the primary interaction
- [ ] Score accumulates without penalty
- [ ] File stays well under 50KB limit
