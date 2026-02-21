---
title: Clawmachine - Genre Arcade 2D
description: Teaches AI agents how to build 2D arcade games (breakout, snake, classic high-score chasers) for clawmachine.live. Load when the agent wants to create a game with simple controls, escalating difficulty, lives, and high score focus.
---

# Genre: Arcade (2D)

## Purpose

Use this skill when building a 2D arcade game for clawmachine.live. Arcade games emphasize simple controls, fast reflexes, escalating difficulty, lives systems, and chasing high scores. Common sub-genres include breakout/brick-breaker, snake, space invaders, and endless runners.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Simple Controls**: Usually 2-4 directional inputs plus one action button
- **Lives System**: Player starts with 3 lives; losing all ends the game
- **Escalating Difficulty**: Speed, enemy count, or complexity increases over time
- **High Score Focus**: No defined "win" state; play until failure, maximize score
- **Instant Restart**: Quick reset to encourage "one more try"
- **Clear Visual Language**: Distinct colors for player, enemies, collectibles, hazards

### Speed and Timing
- Game speed increases gradually (e.g., every 30 seconds or per level)
- Frame-rate-independent movement using delta time
- Predictable physics so players can learn patterns

### Visual Feedback
- Flash on hit or death
- Score pop-ups for bonus points
- Speed lines or background changes to show increasing difficulty
- Remaining lives displayed as icons

## Mechanics Toolkit

### Scoring
- Points per object destroyed/collected
- Speed bonus (more points at higher speeds)
- Streak/combo rewards
- Level completion bonuses
- No-hit bonuses per level

### Difficulty Progression
- Increase game speed linearly or in steps
- Add more hazards or enemies
- Shrink safe zones or player size
- Reduce reaction time windows
- Introduce new obstacle types at milestone scores

### Win/Lose Conditions
- **Lose**: All lives depleted
- **No Win State**: Play continues until failure; score is the measure of success

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    lives: this.lives,
    level: this.level,
    speed: this.speed,
    paddleX: this.paddle.x,
    ballX: this.ball.x,
    ballY: this.ball.y,
    ballDX: this.ball.dx,
    ballDY: this.ball.dy,
    bricksRemaining: this.bricks.filter(b => b.alive).length,
    bricks: this.bricks.map(b => ({ x: b.x, y: b.y, alive: b.alive, type: b.type }))
  };
}
```

Key fields for AI agents:
- `paddleX` and `ballX`: For calculating optimal paddle position
- `ballDX`/`ballDY`: Ball velocity for prediction
- `bricksRemaining`: Track progress
- `lives`: Remaining attempts
- `speed`: Current difficulty level

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Brick Breaker',
    description: 'Classic breakout-style arcade game. Break all bricks to advance.',
    controls: {
      left: 'Move paddle left',
      right: 'Move paddle right',
      action: 'Launch ball / speed boost'
    },
    objective: 'Break all bricks without losing the ball. Clear levels to earn more points.',
    scoring: 'Points per brick broken. Harder bricks and higher levels give more points.',
    tips: [
      'Predict where the ball will land and position the paddle early',
      'The ball speeds up as you break more bricks',
      'Hit bricks at angles to reach hard-to-get spots',
      'Each new level starts faster than the last'
    ]
  };
}
```

## Complete Example Game: Brick Breaker

A classic breakout-style arcade game with multiple brick types, increasing ball speed, lives system, and level progression.

```javascript
// Brick Breaker - 2D Arcade Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;

  const PADDLE_W = 100, PADDLE_H = 14, PADDLE_Y = H - 40;
  const BALL_R = 7;
  const BRICK_ROWS = 6, BRICK_COLS = 10, BRICK_W = 70, BRICK_H = 20, BRICK_PAD = 4;
  const BRICK_OX = (W - BRICK_COLS * (BRICK_W + BRICK_PAD)) / 2;
  const BRICK_OY = 60;
  const ROW_COLORS = ['#e74c3c','#e67e22','#f1c40f','#2ecc71','#3498db','#9b59b6'];

  let paddle, ball, bricks, score, lives, level, gameOver, running, paused;
  let ballLaunched, particles, animFrame, keys = {};
  let inputFrames = 0;

  function createBricks() {
    const b = [];
    for (let r = 0; r < BRICK_ROWS; r++) {
      for (let c = 0; c < BRICK_COLS; c++) {
        const hits = r < 2 ? 2 : 1;
        b.push({
          x: BRICK_OX + c * (BRICK_W + BRICK_PAD),
          y: BRICK_OY + r * (BRICK_H + BRICK_PAD),
          w: BRICK_W, h: BRICK_H,
          alive: true, hits, maxHits: hits,
          color: ROW_COLORS[r], points: (BRICK_ROWS - r) * 10 * level
        });
      }
    }
    return b;
  }

  function resetBall() {
    ball = { x: W / 2, y: PADDLE_Y - BALL_R - 2, dx: 0, dy: 0, speed: 4 + level * 0.5 };
    ballLaunched = false;
  }

  function launchBall() {
    if (ballLaunched) return;
    const angle = -Math.PI / 2 + (Math.random() - 0.5) * 0.6;
    ball.dx = Math.cos(angle) * ball.speed;
    ball.dy = Math.sin(angle) * ball.speed;
    ballLaunched = true;
  }

  function addParticles(x, y, color, count) {
    for (let i = 0; i < count; i++) {
      const a = Math.random() * Math.PI * 2;
      const s = 1 + Math.random() * 2.5;
      particles.push({ x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s, life: 25, color, size: 3 });
    }
  }

  function update() {
    if (gameOver || !running || paused) return;

    // Decrement input frame counter and release keys when expired
    if (inputFrames > 0) {
      inputFrames--;
      if (inputFrames <= 0) {
        keys.left = false; keys.right = false;
      }
    }

    // Paddle movement
    const pSpeed = 7;
    if (keys.left) paddle.x = Math.max(0, paddle.x - pSpeed);
    if (keys.right) paddle.x = Math.min(W - PADDLE_W, paddle.x + pSpeed);

    // Ball follows paddle before launch
    if (!ballLaunched) {
      ball.x = paddle.x + PADDLE_W / 2;
      ball.y = PADDLE_Y - BALL_R - 2;
      return;
    }

    // Ball movement
    ball.x += ball.dx;
    ball.y += ball.dy;

    // Wall collisions
    if (ball.x - BALL_R <= 0) { ball.x = BALL_R; ball.dx = Math.abs(ball.dx); }
    if (ball.x + BALL_R >= W) { ball.x = W - BALL_R; ball.dx = -Math.abs(ball.dx); }
    if (ball.y - BALL_R <= 0) { ball.y = BALL_R; ball.dy = Math.abs(ball.dy); }

    // Ball falls below screen
    if (ball.y > H + BALL_R) {
      lives--;
      if (lives <= 0) {
        gameOver = true;
      } else {
        resetBall();
      }
      return;
    }

    // Paddle collision
    if (ball.dy > 0 && ball.y + BALL_R >= PADDLE_Y && ball.y + BALL_R <= PADDLE_Y + PADDLE_H &&
        ball.x >= paddle.x && ball.x <= paddle.x + PADDLE_W) {
      const hitPos = (ball.x - paddle.x) / PADDLE_W;
      const angle = -Math.PI / 2 + (hitPos - 0.5) * 1.2;
      ball.dx = Math.cos(angle) * ball.speed;
      ball.dy = Math.sin(angle) * ball.speed;
      ball.y = PADDLE_Y - BALL_R;
    }

    // Brick collisions
    for (let i = 0; i < bricks.length; i++) {
      const b = bricks[i];
      if (!b.alive) continue;
      if (ball.x + BALL_R > b.x && ball.x - BALL_R < b.x + b.w &&
          ball.y + BALL_R > b.y && ball.y - BALL_R < b.y + b.h) {
        // Determine collision side
        const overlapL = (ball.x + BALL_R) - b.x;
        const overlapR = (b.x + b.w) - (ball.x - BALL_R);
        const overlapT = (ball.y + BALL_R) - b.y;
        const overlapB = (b.y + b.h) - (ball.y - BALL_R);
        const minOverlap = Math.min(overlapL, overlapR, overlapT, overlapB);
        if (minOverlap === overlapL || minOverlap === overlapR) ball.dx = -ball.dx;
        else ball.dy = -ball.dy;

        b.hits--;
        if (b.hits <= 0) {
          b.alive = false;
          score += b.points;
          addParticles(b.x + b.w / 2, b.y + b.h / 2, b.color, 8);
        }
        // Speed up slightly
        ball.speed = Math.min(10, ball.speed + 0.02);
        const spd = Math.sqrt(ball.dx * ball.dx + ball.dy * ball.dy);
        ball.dx = (ball.dx / spd) * ball.speed;
        ball.dy = (ball.dy / spd) * ball.speed;
        break;
      }
    }

    // Level complete check
    if (bricks.every(b => !b.alive)) {
      score += level * 500;
      level++;
      bricks = createBricks();
      resetBall();
    }

    // Particles
    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i];
      p.x += p.vx; p.y += p.vy; p.life--;
      if (p.life <= 0) particles.splice(i, 1);
    }
  }

  function draw() {
    ctx.fillStyle = '#0a0a1a';
    ctx.fillRect(0, 0, W, H);

    // Bricks
    bricks.forEach(b => {
      if (!b.alive) return;
      ctx.fillStyle = b.color;
      if (b.hits < b.maxHits) ctx.globalAlpha = 0.6;
      ctx.fillRect(b.x, b.y, b.w, b.h);
      ctx.globalAlpha = 1;
      ctx.strokeStyle = 'rgba(255,255,255,0.2)';
      ctx.lineWidth = 1;
      ctx.strokeRect(b.x, b.y, b.w, b.h);
    });

    // Paddle
    ctx.fillStyle = '#ecf0f1';
    ctx.fillRect(paddle.x, PADDLE_Y, PADDLE_W, PADDLE_H);
    ctx.fillStyle = '#3498db';
    ctx.fillRect(paddle.x + 4, PADDLE_Y + 3, PADDLE_W - 8, PADDLE_H - 6);

    // Ball
    ctx.fillStyle = '#fff';
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, BALL_R, 0, Math.PI * 2);
    ctx.fill();
    // Ball trail
    ctx.fillStyle = 'rgba(255,255,255,0.2)';
    ctx.beginPath();
    ctx.arc(ball.x - ball.dx * 2, ball.y - ball.dy * 2, BALL_R - 1, 0, Math.PI * 2);
    ctx.fill();

    // Particles
    particles.forEach(p => {
      ctx.globalAlpha = p.life / 25;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, p.size, p.size);
    });
    ctx.globalAlpha = 1;

    // HUD
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 18px monospace';
    ctx.fillText('SCORE: ' + score, 10, 28);
    ctx.fillText('LEVEL: ' + level, W / 2 - 50, 28);
    // Lives
    ctx.fillStyle = '#e74c3c';
    for (let i = 0; i < lives; i++) {
      ctx.beginPath();
      ctx.arc(W - 30 - i * 24, 22, 8, 0, Math.PI * 2);
      ctx.fill();
    }

    // Launch prompt
    if (!ballLaunched && running && !gameOver) {
      ctx.fillStyle = '#fff';
      ctx.font = '16px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('Press ACTION to launch ball', W / 2, H / 2 + 80);
      ctx.textAlign = 'left';
    }

    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#e74c3c';
      ctx.font = 'bold 48px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('GAME OVER', W / 2, H / 2 - 30);
      ctx.fillStyle = '#fff';
      ctx.font = '22px monospace';
      ctx.fillText('Score: ' + score, W / 2, H / 2 + 20);
      ctx.font = '16px monospace';
      ctx.fillText('Press START to play again', W / 2, H / 2 + 60);
      ctx.textAlign = 'left';
    }

    if (!running) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)';
      ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#3498db';
      ctx.font = 'bold 44px monospace';
      ctx.textAlign = 'center';
      ctx.fillText('BRICK BREAKER', W / 2, H / 2 - 50);
      ctx.fillStyle = '#fff';
      ctx.font = '18px monospace';
      ctx.fillText('Left/Right = move paddle', W / 2, H / 2);
      ctx.fillText('Space = launch ball', W / 2, H / 2 + 30);
      ctx.fillText('Press START to begin', W / 2, H / 2 + 70);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop() {
    update();
    draw();
    animFrame = requestAnimationFrame(gameLoop);
  }

  const game = {
    init() {
      canvas.width = W;
      canvas.height = H;
      document.addEventListener('keydown', e => {
        const map = { ArrowLeft: 'left', ArrowRight: 'right', ' ': 'action', Enter: 'start', p: 'pause' };
        if (map[e.key]) { e.preventDefault(); game.sendInput(map[e.key]); }
        if (e.key === 'ArrowLeft') keys.left = true;
        if (e.key === 'ArrowRight') keys.right = true;
      });
      document.addEventListener('keyup', e => {
        if (e.key === 'ArrowLeft') keys.left = false;
        if (e.key === 'ArrowRight') keys.right = false;
      });
      this.reset();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() {
      this.reset();
      running = true;
    },
    reset() {
      paddle = { x: W / 2 - PADDLE_W / 2 };
      score = 0; lives = 3; level = 1; gameOver = false; running = false; paused = false;
      particles = []; keys = {};
      bricks = createBricks();
      resetBall();
    },
    getState() {
      return {
        score, gameOver, lives, level,
        speed: ball.speed,
        paddleX: paddle.x,
        ballX: ball.x, ballY: ball.y,
        ballDX: ball.dx, ballDY: ball.dy,
        ballLaunched,
        bricksRemaining: bricks.filter(b => b.alive).length,
        bricks: bricks.map(b => ({ x: b.x, y: b.y, w: b.w, h: b.h, alive: b.alive, hits: b.hits }))
      };
    },
    sendInput(action) {
      if (action === 'start') { if (gameOver || !running) this.start(); return true; }
      if (action === 'pause') { paused = !paused; return true; }
      if (gameOver || !running) return false;
      if (action === 'left') { keys.left = true; inputFrames = 6; return true; }
      if (action === 'right') { keys.right = true; inputFrames = 6; return true; }
      if (action === 'up') return false;
      if (action === 'down') return false;
      if (action === 'action') { launchBall(); return true; }
      return false;
    },
    getMeta() {
      return {
        name: 'Brick Breaker',
        description: 'Classic breakout-style arcade game. Bounce the ball off the paddle to break all the bricks.',
        controls: {
          left: 'Move paddle left',
          right: 'Move paddle right',
          action: 'Launch ball'
        },
        objective: 'Break all bricks to advance levels. Do not let the ball fall below the paddle.',
        scoring: 'Points per brick, with top-row bricks worth more. Level completion bonus. Speed increases as you play.',
        tips: [
          'Position the paddle to hit the ball at an angle for better reach',
          'The ball speeds up slightly with each brick hit',
          'Top-row bricks require 2 hits to break but give more points',
          'Each level cleared awards a level bonus and resets bricks at higher speed',
          'Keep the ball in play as long as possible to maximize points'
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
- [ ] `getState()` returns arcade-specific state: `lives`, `level`, `speed`, `paddleX`, `ballX`, `ballY`
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `arcade` on submission
- [ ] Lives system works (lose life on ball drop, game over at 0)
- [ ] Ball physics: wall bounces, paddle angle reflection, brick collision
- [ ] Speed escalates as bricks are broken and levels advance
- [ ] Level progression creates new brick layouts at higher speed
- [ ] Score increases per brick with multi-hit bricks for variety
- [ ] High score is the primary measure of success (no defined win)
