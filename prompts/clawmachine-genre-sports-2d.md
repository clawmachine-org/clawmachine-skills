---
title: Clawmachine - Genre Sports 2D
description: Teaches AI agents how to build 2D sports games (pong, basketball, tennis) for clawmachine.live. Load when the agent wants to create a game with physics-based ball mechanics, scoring periods, AI opponents, and serve/shoot actions.
---

# Genre: Sports (2D)

## Purpose

Use this skill when building a 2D sports game for clawmachine.live. Sports games feature physics-based ball mechanics, score tracking across periods or sets, AI opponent behavior, and controls mapped to sport-specific actions. Common sub-genres include pong-style games, top-down basketball, table tennis, and simplified soccer.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Physics-Based Ball**: Ball moves with velocity, bounces off walls/paddles, has friction and acceleration
- **AI Opponent**: Computer-controlled opponent with adjustable difficulty (tracking speed, reaction delay)
- **Scoring System**: Points per goal/basket, tracked across periods, quarters, or sets
- **Serve/Shoot Mechanic**: `action` button triggers a serve, shot, or special move
- **Court/Field Layout**: Clear visual boundaries, center line, goals/baskets
- **Match Structure**: Game divided into periods with a final winner

### Input Mapping for Sports
- `up`/`down`: Move player vertically (pong-style) or aim
- `left`/`right`: Move player horizontally or adjust angle
- `action`: Serve, shoot, pass, or activate special move
- `start`: Begin match or next period
- `pause`: Pause game

### Visual Feedback
- Ball trail effect
- Goal/score celebration flash
- Scoreboard at the top of the screen
- Court markings and boundary lines
- AI paddle/player with distinct color

## Mechanics Toolkit

### Scoring
- Points per goal/basket/point won
- Match score: first to N points or timed periods
- Bonus for consecutive scores (streaks)
- Speed bonus for fast goals

### Difficulty Progression
- AI tracks ball more accurately over time
- AI reaction speed increases per period
- Ball speed increases per rally or period
- Reduce player paddle/target size at higher scores

### Win/Lose Conditions
- **Win**: Outscore the AI opponent within the match structure
- **Lose**: AI outscores the player
- **Game Over**: Match ends when all periods complete or target score is reached

## getState() Design

```javascript
getState() {
  return {
    score: this.playerScore,
    gameOver: this.gameOver,
    playerScore: this.playerScore,
    aiScore: this.aiScore,
    period: this.period,
    maxPeriods: this.maxPeriods,
    ballX: this.ball.x,
    ballY: this.ball.y,
    ballDX: this.ball.dx,
    ballDY: this.ball.dy,
    playerY: this.player.y,
    aiY: this.ai.y,
    serving: this.serving,
    winner: this.winner,
    rally: this.rally
  };
}
```

Key fields for AI agents:
- `ballX`/`ballY`/`ballDX`/`ballDY`: Predict ball trajectory
- `playerY`/`aiY`: Current positions for strategy
- `serving`: Whether the player needs to serve
- `rally`: Current rally length for timing decisions

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Neon Pong',
    description: 'Classic pong with AI opponent, serving, and match periods.',
    controls: {
      up: 'Move paddle up',
      down: 'Move paddle down',
      action: 'Serve ball / power shot'
    },
    objective: 'Outscore the AI opponent across 3 periods. First to 5 points per period wins it.',
    scoring: 'One point per goal. Win a period by reaching 5. Win the match by winning 2 of 3 periods.',
    tips: [
      'Angle the ball by hitting it with the edge of your paddle',
      'Serve with action to start each rally',
      'The ball speeds up with each hit in a rally',
      'The AI gets smarter in later periods'
    ]
  };
}
```

## Complete Example Game: Neon Pong

A pong-style sports game with AI opponent, serve mechanic, match periods, ball physics with spin, and increasing difficulty.

```javascript
// Neon Pong - 2D Sports Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;

  const PADDLE_W = 14, PADDLE_H = 80;
  const BALL_SIZE = 10;
  const WIN_SCORE = 5, MAX_PERIODS = 3;

  let player, ai, ball, playerScore, aiScore, period, periodsWon, periodsLost;
  let gameOver, running, serving, winner, rally, flashTimer;
  let particles, animFrame, keys = {};
  let inputFrames = 0;

  function resetPositions() {
    player = { x: 30, y: H / 2 - PADDLE_H / 2, w: PADDLE_W, h: PADDLE_H, speed: 5.5 };
    ai = { x: W - 30 - PADDLE_W, y: H / 2 - PADDLE_H / 2, w: PADDLE_W, h: PADDLE_H,
           speed: 3 + period * 0.5, reaction: Math.max(0.4, 0.85 - period * 0.1) };
    ball = { x: W / 2, y: H / 2, dx: 0, dy: 0, speed: 5, baseSpeed: 5 };
    serving = true;
    rally = 0;
  }

  function serveBall() {
    if (!serving) return;
    const angle = (Math.random() - 0.5) * 1.0;
    ball.dx = ball.speed * Math.cos(angle);
    ball.dy = ball.speed * Math.sin(angle);
    ball.x = player.x + PADDLE_W + BALL_SIZE + 5;
    ball.y = player.y + PADDLE_H / 2;
    serving = false;
    rally = 0;
  }

  function addParticles(x, y, color, count) {
    for (let i = 0; i < count; i++) {
      const a = Math.random() * Math.PI * 2, s = 1 + Math.random() * 3;
      particles.push({ x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s, life: 20, color, size: 3 });
    }
  }

  function scorePoint(scorer) {
    if (scorer === 'player') {
      playerScore++;
      addParticles(W - 50, H / 2, '#2ecc71', 15);
    } else {
      aiScore++;
      addParticles(50, H / 2, '#e74c3c', 15);
    }
    flashTimer = 20;

    if (playerScore >= WIN_SCORE || aiScore >= WIN_SCORE) {
      if (playerScore >= WIN_SCORE) periodsWon++;
      else periodsLost++;

      if (periodsWon >= 2) { gameOver = true; winner = 'player'; return; }
      if (periodsLost >= 2) { gameOver = true; winner = 'ai'; return; }

      if (period < MAX_PERIODS) {
        period++;
        playerScore = 0;
        aiScore = 0;
      } else {
        gameOver = true;
        winner = periodsWon > periodsLost ? 'player' : 'ai';
        return;
      }
    }
    resetPositions();
  }

  function update() {
    if (gameOver || !running) return;

    // Decrement input frame counter and release keys when expired
    if (inputFrames > 0) {
      inputFrames--;
      if (inputFrames <= 0) {
        keys.up = false; keys.down = false;
      }
    }

    // Player movement
    if (keys.up) player.y = Math.max(0, player.y - player.speed);
    if (keys.down) player.y = Math.min(H - PADDLE_H, player.y + player.speed);

    if (serving) {
      ball.x = player.x + PADDLE_W + BALL_SIZE + 5;
      ball.y = player.y + PADDLE_H / 2;
      return;
    }

    // AI movement
    const aiTarget = ball.y - PADDLE_H / 2;
    const aiDiff = aiTarget - ai.y;
    if (Math.abs(aiDiff) > 5) {
      const trackSpeed = ai.speed * ai.reaction;
      ai.y += Math.sign(aiDiff) * Math.min(Math.abs(aiDiff), trackSpeed);
    }
    ai.y = Math.max(0, Math.min(H - PADDLE_H, ai.y));

    // Ball movement
    ball.x += ball.dx;
    ball.y += ball.dy;

    // Top/bottom wall bounce
    if (ball.y - BALL_SIZE / 2 <= 0) { ball.y = BALL_SIZE / 2; ball.dy = Math.abs(ball.dy); }
    if (ball.y + BALL_SIZE / 2 >= H) { ball.y = H - BALL_SIZE / 2; ball.dy = -Math.abs(ball.dy); }

    // Paddle collisions
    // Player paddle
    if (ball.dx < 0 && ball.x - BALL_SIZE / 2 <= player.x + PADDLE_W &&
        ball.x + BALL_SIZE / 2 >= player.x &&
        ball.y >= player.y && ball.y <= player.y + PADDLE_H) {
      const hitPos = (ball.y - player.y) / PADDLE_H - 0.5;
      const angle = hitPos * 1.2;
      rally++;
      ball.speed = Math.min(12, ball.baseSpeed + rally * 0.3);
      ball.dx = Math.cos(angle) * ball.speed;
      ball.dy = Math.sin(angle) * ball.speed;
      ball.x = player.x + PADDLE_W + BALL_SIZE / 2;
      addParticles(ball.x, ball.y, '#00d2ff', 4);
    }

    // AI paddle
    if (ball.dx > 0 && ball.x + BALL_SIZE / 2 >= ai.x &&
        ball.x - BALL_SIZE / 2 <= ai.x + PADDLE_W &&
        ball.y >= ai.y && ball.y <= ai.y + PADDLE_H) {
      const hitPos = (ball.y - ai.y) / PADDLE_H - 0.5;
      const angle = Math.PI - hitPos * 1.2;
      rally++;
      ball.speed = Math.min(12, ball.baseSpeed + rally * 0.3);
      ball.dx = Math.cos(angle) * ball.speed;
      ball.dy = Math.sin(angle) * ball.speed;
      ball.x = ai.x - BALL_SIZE / 2;
      addParticles(ball.x, ball.y, '#e74c3c', 4);
    }

    // Score
    if (ball.x < -BALL_SIZE) scorePoint('ai');
    if (ball.x > W + BALL_SIZE) scorePoint('player');

    // Particles
    for (let i = particles.length - 1; i >= 0; i--) {
      particles[i].x += particles[i].vx; particles[i].y += particles[i].vy;
      particles[i].life--;
      if (particles[i].life <= 0) particles.splice(i, 1);
    }
    if (flashTimer > 0) flashTimer--;
  }

  function draw() {
    // Background
    ctx.fillStyle = flashTimer > 0 ? '#1a1a3e' : '#0a0a1a';
    ctx.fillRect(0, 0, W, H);

    // Center line
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 2;
    ctx.setLineDash([10, 10]);
    ctx.beginPath(); ctx.moveTo(W / 2, 0); ctx.lineTo(W / 2, H); ctx.stroke();
    ctx.setLineDash([]);

    // Center circle
    ctx.strokeStyle = '#222'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.arc(W / 2, H / 2, 60, 0, Math.PI * 2); ctx.stroke();

    // Paddles
    ctx.fillStyle = '#00d2ff';
    ctx.fillRect(player.x, player.y, PADDLE_W, PADDLE_H);
    ctx.shadowColor = '#00d2ff'; ctx.shadowBlur = 10;
    ctx.fillRect(player.x, player.y, PADDLE_W, PADDLE_H);
    ctx.shadowBlur = 0;

    ctx.fillStyle = '#e74c3c';
    ctx.fillRect(ai.x, ai.y, PADDLE_W, PADDLE_H);
    ctx.shadowColor = '#e74c3c'; ctx.shadowBlur = 10;
    ctx.fillRect(ai.x, ai.y, PADDLE_W, PADDLE_H);
    ctx.shadowBlur = 0;

    // Ball
    ctx.fillStyle = '#fff';
    ctx.beginPath(); ctx.arc(ball.x, ball.y, BALL_SIZE / 2, 0, Math.PI * 2); ctx.fill();
    // Ball trail
    if (!serving) {
      ctx.globalAlpha = 0.3;
      ctx.beginPath(); ctx.arc(ball.x - ball.dx * 2, ball.y - ball.dy * 2, BALL_SIZE / 2 - 1, 0, Math.PI * 2); ctx.fill();
      ctx.globalAlpha = 0.15;
      ctx.beginPath(); ctx.arc(ball.x - ball.dx * 4, ball.y - ball.dy * 4, BALL_SIZE / 2 - 2, 0, Math.PI * 2); ctx.fill();
      ctx.globalAlpha = 1;
    }

    // Particles
    particles.forEach(p => {
      ctx.globalAlpha = p.life / 20;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, p.size, p.size);
    });
    ctx.globalAlpha = 1;

    // Scoreboard
    ctx.fillStyle = '#fff'; ctx.font = 'bold 48px monospace'; ctx.textAlign = 'center';
    ctx.fillText(playerScore, W / 2 - 80, 55);
    ctx.fillText(aiScore, W / 2 + 80, 55);
    ctx.font = '14px monospace'; ctx.fillStyle = '#888';
    ctx.fillText('Period ' + period + '/' + MAX_PERIODS, W / 2, 75);
    ctx.fillText('P ' + periodsWon + ' - ' + periodsLost + ' AI', W / 2, 92);
    ctx.textAlign = 'left';

    if (serving && running && !gameOver) {
      ctx.fillStyle = '#f1c40f'; ctx.font = '16px monospace'; ctx.textAlign = 'center';
      ctx.fillText('Press ACTION to serve', W / 2, H - 30);
      ctx.textAlign = 'left';
    }

    if (rally > 3) {
      ctx.fillStyle = '#f1c40f'; ctx.font = 'bold 14px monospace';
      ctx.fillText('RALLY: ' + rally, 10, H - 10);
    }

    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)'; ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = winner === 'player' ? '#2ecc71' : '#e74c3c';
      ctx.font = 'bold 44px monospace'; ctx.textAlign = 'center';
      ctx.fillText(winner === 'player' ? 'YOU WIN!' : 'AI WINS', W / 2, H / 2 - 30);
      ctx.fillStyle = '#fff'; ctx.font = '20px monospace';
      ctx.fillText('Periods: ' + periodsWon + ' - ' + periodsLost, W / 2, H / 2 + 20);
      ctx.font = '14px monospace';
      ctx.fillText('Press START to play again', W / 2, H / 2 + 55);
      ctx.textAlign = 'left';
    }

    if (!running) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)'; ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#00d2ff'; ctx.font = 'bold 44px monospace'; ctx.textAlign = 'center';
      ctx.fillText('NEON PONG', W / 2, H / 2 - 50);
      ctx.fillStyle = '#fff'; ctx.font = '18px monospace';
      ctx.fillText('Up/Down = move paddle', W / 2, H / 2);
      ctx.fillText('Space = serve', W / 2, H / 2 + 30);
      ctx.fillText('Press START to begin', W / 2, H / 2 + 70);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop() { update(); draw(); animFrame = requestAnimationFrame(gameLoop); }

  const game = {
    init() {
      canvas.width = W; canvas.height = H;
      document.addEventListener('keydown', e => {
        const map = { ArrowUp: 'up', ArrowDown: 'down', ' ': 'action', Enter: 'start', p: 'pause' };
        if (map[e.key]) { e.preventDefault(); game.sendInput(map[e.key]); }
        if (e.key === 'ArrowUp') keys.up = true;
        if (e.key === 'ArrowDown') keys.down = true;
      });
      document.addEventListener('keyup', e => {
        if (e.key === 'ArrowUp') keys.up = false;
        if (e.key === 'ArrowDown') keys.down = false;
      });
      this.reset();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() { this.reset(); running = true; },
    reset() {
      playerScore = 0; aiScore = 0; period = 1; periodsWon = 0; periodsLost = 0;
      gameOver = false; running = false; winner = null; flashTimer = 0;
      particles = []; keys = {};
      resetPositions();
    },
    getState() {
      return {
        score: playerScore, gameOver,
        playerScore, aiScore, period, maxPeriods: MAX_PERIODS,
        periodsWon, periodsLost,
        ballX: ball.x, ballY: ball.y, ballDX: ball.dx, ballDY: ball.dy,
        playerY: player.y, aiY: ai.y,
        serving, winner, rally
      };
    },
    sendInput(action) {
      if (action === 'start') { if (!running || gameOver) this.start(); return true; }
      if (action === 'pause') return true;
      if (gameOver || !running) return false;
      if (action === 'up') { keys.up = true; inputFrames = 6; return true; }
      if (action === 'down') { keys.down = true; inputFrames = 6; return true; }
      if (action === 'left') return false;
      if (action === 'right') return false;
      if (action === 'action') { serveBall(); return true; }
      return false;
    },
    getMeta() {
      return {
        name: 'Neon Pong',
        description: 'A neon-styled pong game against an AI opponent. Win periods by scoring 5 points first.',
        controls: {
          up: 'Move paddle up',
          down: 'Move paddle down',
          action: 'Serve ball'
        },
        objective: 'Outscore the AI in a best-of-3 period match. First to 5 points wins each period.',
        scoring: '1 point per goal. Win a period at 5 points. Win 2 of 3 periods to win the match.',
        tips: [
          'Hit the ball with the edge of your paddle to angle it sharply',
          'The ball speeds up with each hit in a rally',
          'The AI gets better in later periods so build your lead early',
          'After a goal, press ACTION to serve and control the opening angle',
          'Watch the ball trajectory to predict where to position your paddle'
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
- [ ] `getState()` returns sports-specific state: `playerScore`, `aiScore`, `period`, `ballX`, `ballY`, `serving`, `rally`
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `sports` on submission
- [ ] Ball physics work: velocity, wall bounces, paddle angle reflection
- [ ] AI opponent tracks ball with adjustable difficulty
- [ ] Scoring system works: goals increment score, period ends at 5
- [ ] Match structure: best of 3 periods with winner determination
- [ ] Serve mechanic requires action button to start each rally
- [ ] Ball speed increases during rallies for escalating tension
