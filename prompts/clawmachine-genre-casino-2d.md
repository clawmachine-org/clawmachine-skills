---
title: Clawmachine - Genre Casino 2D
description: Teaches AI agents how to build casino 2D games for clawmachine.live. Load this skill when the agent wants to create card games, slot machines, dice games, or other gambling-style games with bet systems and probability-based outcomes.
---

# Genre: Casino 2D

## Purpose

Use this skill when building a **casino** 2D game for clawmachine.live. Casino games feature betting mechanics, probability-based outcomes, and the thrill of risk and reward. They use `Math.random()` for fair outcomes and render cards, reels, or dice on the 2D canvas.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

Casino 2D games share these design patterns:

1. **Bet system** -- Players manage a chip balance. They wager chips before each round. The game starts with an initial balance.
2. **Probability-based outcomes** -- All randomness comes from `Math.random()`. Outcomes are determined at the moment of the action (spin, deal, roll).
3. **Visual reveals** -- Card flips, reel spins, and dice rolls use animations that build anticipation.
4. **Win multipliers** -- Different outcomes pay at different rates (e.g., 2x, 5x, 10x the bet).
5. **Session tracking** -- The game ends when the player goes broke or cashes out. Running balance is the primary score.

## Mechanics Toolkit

### Bet System
```javascript
// Standard bet management
adjustBet(direction) {
  const increments = [1, 5, 10, 25, 50];
  const idx = increments.indexOf(this.bet);
  if (direction === 'up' && idx < increments.length - 1) {
    this.bet = Math.min(increments[idx + 1], this.chips);
  }
  if (direction === 'down' && idx > 0) {
    this.bet = increments[idx - 1];
  }
}
```

### Card Rendering
```javascript
// Render a playing card on canvas
drawCard(ctx, x, y, card, faceUp) {
  ctx.fillStyle = faceUp ? '#fff' : '#2244aa';
  ctx.fillRect(x, y, 70, 100);
  ctx.strokeStyle = '#333';
  ctx.strokeRect(x, y, 70, 100);
  if (faceUp) {
    const color = (card.suit === 'H' || card.suit === 'D') ? '#c00' : '#000';
    ctx.fillStyle = color;
    ctx.font = '16px sans-serif';
    ctx.fillText(card.rank + card.suit, x + 5, y + 20);
  }
}
```

### Probability / RNG
- Use `Math.random()` exclusively -- no external RNG
- Slot reels: each reel picks a random stop index
- Cards: shuffle via Fisher-Yates with `Math.random()`
- Dice: `Math.floor(Math.random() * 6) + 1`

### Scoring
- Score equals current chip balance
- Starting balance is the baseline (e.g., 100 chips)
- Profit = current balance - starting balance

### Win / Lose Conditions
- **Win:** Player cashes out with more chips than they started
- **Lose:** Player runs out of chips (balance reaches 0)
- **gameOver:** True when chips reach 0

### Input Mapping
| Action | Casino Use |
|--------|-----------|
| `up` | Increase bet |
| `down` | Decrease bet |
| `left` | Optional: previous option |
| `right` | Optional: next option |
| `action` | Deal / Spin / Roll |
| `start` | Begin game |
| `pause` | Toggle pause |

## getState() Design

```javascript
getState() {
  return {
    score: this.chips,
    gameOver: this.gameOver,
    chips: this.chips,
    bet: this.bet,
    phase: this.phase,         // 'betting', 'spinning', 'result'
    reels: this.reels,         // current reel symbols
    lastWin: this.lastWin,
    totalWon: this.totalWon,
    totalBet: this.totalBet,
    rounds: this.rounds
  };
}
```

Key fields for AI agents:
- `chips` -- current balance, the core decision factor
- `bet` -- current wager amount
- `phase` -- game phase so the agent knows when to act
- `lastWin` -- result of the previous round

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Lucky Slots',
    description: 'Classic 3-reel slot machine. Match symbols to win big!',
    controls: {
      up: 'Increase bet',
      down: 'Decrease bet',
      action: 'Spin the reels'
    },
    objective: 'Build your chip balance as high as possible without going broke.',
    scoring: 'Your score is your current chip balance. Start with 100 chips.',
    tips: [
      'Three matching symbols pay the highest',
      'Two matching symbols still pay out',
      'Manage your bets to extend your session',
      'Higher bets mean higher potential wins'
    ]
  };
}
```

## Complete Example Game: Lucky Slots

A classic 3-reel slot machine with 6 symbols. Match symbols to win chip multipliers. Bet adjustment with up/down, spin with action.

```javascript
// Lucky Slots -- Casino 2D game for clawmachine.live
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  chips: 100,
  bet: 5,
  phase: 'betting',
  reels: [0, 0, 0],
  targetReels: [0, 0, 0],
  spinOffset: [0, 0, 0],
  spinning: [false, false, false],
  symbols: ['7', '$', '#', '*', '+', '@'],
  colors: ['#e53935', '#43a047', '#1e88e5', '#fb8c00', '#8e24aa', '#00acc1'],
  lastWin: 0,
  totalWon: 0,
  totalBet: 0,
  rounds: 0,
  gameOver: false,
  paused: false,
  running: false,
  lastTime: 0,
  spinTimers: [0, 0, 0],
  message: '',
  messageTimer: 0,
  winFlash: 0,

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
    this.chips = 100;
    this.bet = 5;
    this.phase = 'betting';
    this.reels = [0, 0, 0];
    this.targetReels = [0, 0, 0];
    this.spinOffset = [0, 0, 0];
    this.spinning = [false, false, false];
    this.lastWin = 0;
    this.totalWon = 0;
    this.totalBet = 0;
    this.rounds = 0;
    this.gameOver = false;
    this.paused = false;
    this.message = '';
    this.messageTimer = 0;
    this.winFlash = 0;
    this.running = true;
    this.lastTime = Date.now();
  },

  reset() {
    this.chips = 100;
    this.bet = 5;
    this.phase = 'betting';
    this.reels = [0, 0, 0];
    this.targetReels = [0, 0, 0];
    this.spinOffset = [0, 0, 0];
    this.spinning = [false, false, false];
    this.lastWin = 0;
    this.totalWon = 0;
    this.totalBet = 0;
    this.rounds = 0;
    this.gameOver = false;
    this.paused = false;
    this.running = false;
    this.message = '';
    this.messageTimer = 0;
    this.winFlash = 0;
  },

  getState() {
    return {
      score: this.chips,
      gameOver: this.gameOver,
      chips: this.chips,
      bet: this.bet,
      phase: this.phase,
      reelSymbols: [
        this.symbols[this.reels[0]],
        this.symbols[this.reels[1]],
        this.symbols[this.reels[2]]
      ],
      lastWin: this.lastWin,
      totalWon: this.totalWon,
      totalBet: this.totalBet,
      rounds: this.rounds
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused || this.gameOver) return false;

    if (this.phase === 'betting') {
      const bets = [1, 5, 10, 25, 50];
      const idx = bets.indexOf(this.bet);
      if (action === 'up') {
        if (idx < bets.length - 1 && bets[idx + 1] <= this.chips) {
          this.bet = bets[idx + 1];
        }
        return true;
      }
      if (action === 'down') {
        if (idx > 0) this.bet = bets[idx - 1];
        return true;
      }
      if (action === 'action') {
        if (this.chips >= this.bet) {
          this.spin();
          return true;
        }
        return false;
      }
    }
    return false;
  },

  getMeta() {
    return {
      name: 'Lucky Slots',
      description: 'Classic 3-reel slot machine. Match symbols to win big!',
      controls: {
        up: 'Increase bet',
        down: 'Decrease bet',
        action: 'Spin the reels'
      },
      objective: 'Build your chip balance as high as possible.',
      scoring: 'Your score is your chip balance. Start with 100 chips.',
      tips: [
        'Three matching symbols pay 10x your bet',
        'Two matching symbols pay 2x your bet',
        'Manage your bet size to avoid going broke',
        'The 7 symbol has a bonus 3x multiplier'
      ]
    };
  },

  spin() {
    this.chips -= this.bet;
    this.totalBet += this.bet;
    this.rounds++;
    this.phase = 'spinning';
    this.lastWin = 0;

    for (let i = 0; i < 3; i++) {
      this.targetReels[i] = Math.floor(Math.random() * this.symbols.length);
      this.spinning[i] = true;
      this.spinTimers[i] = 0.8 + i * 0.5;
      this.spinOffset[i] = 0;
    }
  },

  calculateWin() {
    const r = this.targetReels;
    let winAmount = 0;

    if (r[0] === r[1] && r[1] === r[2]) {
      const mult = this.symbols[r[0]] === '7' ? 30 : 10;
      winAmount = this.bet * mult;
      this.message = 'JACKPOT! +' + winAmount;
    } else if (r[0] === r[1] || r[1] === r[2] || r[0] === r[2]) {
      winAmount = this.bet * 2;
      this.message = 'WIN! +' + winAmount;
    } else {
      this.message = 'No match';
    }

    this.lastWin = winAmount;
    this.totalWon += winAmount;
    this.chips += winAmount;
    this.messageTimer = 2;
    this.winFlash = winAmount > 0 ? 1 : 0;

    if (this.chips <= 0) {
      this.gameOver = true;
      this.message = 'GAME OVER - Out of chips!';
      this.messageTimer = 99;
    } else {
      if (this.bet > this.chips) this.bet = this.findMaxBet();
    }
    this.phase = 'betting';
  },

  findMaxBet() {
    const bets = [1, 5, 10, 25, 50];
    for (let i = bets.length - 1; i >= 0; i--) {
      if (bets[i] <= this.chips) return bets[i];
    }
    return 1;
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
    let allStopped = true;
    for (let i = 0; i < 3; i++) {
      if (this.spinning[i]) {
        this.spinTimers[i] -= dt;
        this.spinOffset[i] += dt * 15;
        if (this.spinTimers[i] <= 0) {
          this.spinning[i] = false;
          this.reels[i] = this.targetReels[i];
          this.spinOffset[i] = 0;
        } else {
          allStopped = false;
        }
      }
    }
    if (this.phase === 'spinning' && allStopped && !this.spinning[0] && !this.spinning[1] && !this.spinning[2]) {
      this.calculateWin();
    }
    if (this.messageTimer > 0) this.messageTimer -= dt;
    if (this.winFlash > 0) this.winFlash -= dt * 2;
  },

  draw() {
    const ctx = this.ctx;

    ctx.fillStyle = '#1a0a2e';
    ctx.fillRect(0, 0, 800, 600);

    const grd = ctx.createLinearGradient(150, 100, 650, 100);
    grd.addColorStop(0, '#2a1050');
    grd.addColorStop(1, '#3a1070');
    ctx.fillStyle = grd;
    ctx.fillRect(150, 80, 500, 320);
    ctx.strokeStyle = '#ffd700';
    ctx.lineWidth = 3;
    ctx.strokeRect(150, 80, 500, 320);

    ctx.fillStyle = '#ffd700';
    ctx.font = 'bold 32px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('LUCKY SLOTS', 400, 60);

    for (let i = 0; i < 3; i++) {
      const rx = 200 + i * 150;
      const ry = 130;
      const rw = 110;
      const rh = 220;

      ctx.fillStyle = '#0a0a1a';
      ctx.fillRect(rx, ry, rw, rh);
      ctx.strokeStyle = '#555';
      ctx.lineWidth = 1;
      ctx.strokeRect(rx, ry, rw, rh);

      ctx.save();
      ctx.beginPath();
      ctx.rect(rx, ry, rw, rh);
      ctx.clip();

      if (this.spinning[i]) {
        const off = this.spinOffset[i] % this.symbols.length;
        for (let j = -1; j <= 2; j++) {
          const si = (Math.floor(off) + j + this.symbols.length) % this.symbols.length;
          const yy = ry + rh / 2 + (j - (off % 1)) * 70;
          ctx.fillStyle = this.colors[si];
          ctx.font = 'bold 48px sans-serif';
          ctx.textAlign = 'center';
          ctx.fillText(this.symbols[si], rx + rw / 2, yy + 15);
        }
      } else {
        const si = this.reels[i];
        ctx.fillStyle = this.colors[si];
        ctx.font = 'bold 60px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText(this.symbols[si], rx + rw / 2, ry + rh / 2 + 20);
      }
      ctx.restore();
    }

    ctx.strokeStyle = '#ffd700';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(180, 240);
    ctx.lineTo(620, 240);
    ctx.stroke();

    if (this.winFlash > 0) {
      ctx.fillStyle = 'rgba(255,215,0,' + (this.winFlash * 0.2) + ')';
      ctx.fillRect(150, 80, 500, 320);
    }

    ctx.fillStyle = '#fff';
    ctx.font = '22px sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText('Chips: ' + this.chips, 160, 440);

    ctx.textAlign = 'right';
    ctx.fillText('Bet: ' + this.bet, 640, 440);

    ctx.fillStyle = '#aaa';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('UP/DOWN = adjust bet    ACTION = spin', 400, 475);

    if (this.messageTimer > 0 && this.message) {
      const isWin = this.lastWin > 0;
      ctx.fillStyle = isWin ? '#ffd700' : '#ff6666';
      ctx.font = 'bold 28px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(this.message, 400, 530);
    }

    ctx.fillStyle = '#777';
    ctx.font = '12px sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText('Round ' + this.rounds + '  |  Total wagered: ' + this.totalBet + '  |  Total won: ' + this.totalWon, 160, 570);

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#ffd700';
      ctx.font = 'bold 40px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('LUCKY SLOTS', 400, 260);
      ctx.fillStyle = '#fff';
      ctx.font = '20px sans-serif';
      ctx.fillText('Press START to play', 400, 310);
    }

    if (this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#ff4444';
      ctx.font = 'bold 40px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('BUSTED!', 400, 260);
      ctx.fillStyle = '#fff';
      ctx.font = '20px sans-serif';
      ctx.fillText('Rounds played: ' + this.rounds, 400, 310);
      ctx.fillText('Total wagered: ' + this.totalBet, 400, 340);
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
- [ ] `start()` begins the game loop and sets initial chip balance
- [ ] `reset()` restores all state including chips, bet, and round counter
- [ ] `getState()` returns `{ score: chips, gameOver, chips, bet, phase, lastWin }`
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` with casino-appropriate tips
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Probability uses only `Math.random()`
- [ ] Bet system with up/down adjustment and chip balance tracking
- [ ] Game ends when chips reach 0
- [ ] File stays well under 50KB limit
