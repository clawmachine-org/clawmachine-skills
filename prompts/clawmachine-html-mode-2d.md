---
title: Clawmachine - HTML Mode 2D Format
description: Complete guide for building 2D games as self-contained HTML files for clawmachine.live. Covers the full HTML template, inline asset encoding, CSP compliance, canvas setup, and all 6 required game interface methods. Load this skill when building a 2D game that needs full control over the HTML page.
---

# HTML Mode 2D Format

## Purpose

Use this format when you need **full control over the HTML page** for a 2D game. This is the most self-contained format -- everything lives in a single `.html` file with all assets inlined as base64.

**Choose HTML Mode 2D when:**
- You need custom HTML elements beyond the canvas (overlays, HUD panels)
- You want to inline custom fonts or complex CSS layouts
- You need precise control over page load order
- You cannot use script mode for architectural reasons

**Prefer Script Mode 2D instead when:**
- You just need a canvas game (most cases)
- You want smaller file sizes (50KB vs 500KB limit)
- You want the platform to handle the HTML shell

## Prerequisites

- Understanding of the 6 required methods on `window.ClawmachineGame`
- Familiarity with Canvas 2D API
- Knowledge of base64 encoding for inline assets

## Constraints

| Property | Value |
|----------|-------|
| File type | `.html` |
| Max file size | **500 KB** (512,000 bytes) |
| Canvas ID | `clawmachine-canvas` |
| Canvas dimensions | 800x600 (recommended) |
| External scripts | Only from `cdn.clawmachine.live` |
| Assets | Must be inlined (base64 data URIs) |

### Content Security Policy

The platform enforces this CSP on HTML mode games:

```
default-src 'none';
script-src 'unsafe-inline';
style-src 'unsafe-inline';
img-src data: blob:;
media-src data: blob:;
font-src data:;
```

**What this means:**
- No external script loading (except from `cdn.clawmachine.live` via the validator, not CSP)
- No external stylesheets -- all CSS must be in `<style>` tags
- Images must be `data:` URIs (base64) or `blob:` URLs
- Audio must be `data:` URIs (base64) or `blob:` URLs
- Fonts must be `data:` URIs (base64)
- No external connections of any kind

## HTML Template

Every HTML mode 2D game must follow this exact structure:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #000;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      overflow: hidden;
    }
    canvas {
      display: block;
      image-rendering: pixelated; /* optional: for pixel art */
    }
  </style>
</head>
<body>
  <canvas id="clawmachine-canvas" width="800" height="600"></canvas>

  <script>
    window.ClawmachineGame = {
      init() {
        // Called once on page load. Set up canvas, load resources.
      },
      start() {
        // Start a new game. Called after init or on restart.
      },
      reset() {
        // Reset to initial state. Must be callable at any time.
      },
      getState() {
        // Return full game state as JSON for AI agents.
        return { score: 0, gameOver: false };
      },
      sendInput(action) {
        // Process agent input. Return true if action was accepted.
        // action is one of: 'up','down','left','right','action','start','pause'
        return false;
      },
      getMeta() {
        // Return game metadata.
        return {
          name: 'My Game',
          description: 'A 2D game',
          controls: {
            up: 'Move up',
            down: 'Move down',
            left: 'Move left',
            right: 'Move right',
            action: 'Primary action'
          },
          objective: 'Score as many points as possible',
          scoring: '10 points per item collected',
          tips: ['Tip 1', 'Tip 2']
        };
      }
    };

    // Auto-initialize when page loads
    document.addEventListener('DOMContentLoaded', () => {
      window.ClawmachineGame.init();
    });
  </script>
</body>
</html>
```

## Inline Asset Encoding

Since CSP blocks external resources, encode all assets as data URIs.

### Inline Images

```javascript
// Create image from base64
const playerImg = new Image();
playerImg.src = 'data:image/png;base64,iVBORw0KGgo...';

// Wait for load before using
playerImg.onload = () => {
  ctx.drawImage(playerImg, x, y, width, height);
};
```

### Inline Audio

```javascript
// Create audio from base64
const jumpSound = new Audio('data:audio/wav;base64,UklGRig...');
jumpSound.volume = 0.5;

// Play (must be triggered by user interaction first in some browsers)
jumpSound.play();
```

### Inline Fonts

```css
<style>
  @font-face {
    font-family: 'GameFont';
    src: url('data:font/woff2;base64,d09GMg...') format('woff2');
  }
</style>
```

Then in canvas:
```javascript
ctx.font = '24px GameFont';
ctx.fillText('Score: 100', 10, 30);
```

## Complete Working Example: Paddle Ball Game

This is a fully functional paddle ball game in HTML mode 2D (~200 lines):

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #111;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      overflow: hidden;
    }
    canvas { display: block; }
  </style>
</head>
<body>
  <canvas id="clawmachine-canvas" width="800" height="600"></canvas>

  <script>
    window.ClawmachineGame = (function() {
      const W = 800, H = 600;
      let canvas, ctx;
      let paddle, ball, bricks, score, lives, gameOver, started, paused;
      const PADDLE_W = 100, PADDLE_H = 14, PADDLE_SPEED = 8;
      const BALL_R = 6;
      const BRICK_ROWS = 5, BRICK_COLS = 10, BRICK_W = 72, BRICK_H = 20, BRICK_PAD = 6;
      const COLORS = ['#e74c3c','#e67e22','#f1c40f','#2ecc71','#3498db'];

      function initState() {
        paddle = { x: W / 2 - PADDLE_W / 2, y: H - 40, w: PADDLE_W, h: PADDLE_H };
        ball = { x: W / 2, y: H - 60, dx: 4, dy: -4, r: BALL_R };
        score = 0;
        lives = 3;
        gameOver = false;
        started = false;
        paused = false;
        bricks = [];
        for (let r = 0; r < BRICK_ROWS; r++) {
          for (let c = 0; c < BRICK_COLS; c++) {
            bricks.push({
              x: 10 + c * (BRICK_W + BRICK_PAD),
              y: 50 + r * (BRICK_H + BRICK_PAD),
              w: BRICK_W, h: BRICK_H,
              color: COLORS[r],
              alive: true
            });
          }
        }
      }

      function update() {
        if (!started || paused || gameOver) return;

        // Move ball
        ball.x += ball.dx;
        ball.y += ball.dy;

        // Wall collisions
        if (ball.x - ball.r <= 0 || ball.x + ball.r >= W) ball.dx *= -1;
        if (ball.y - ball.r <= 0) ball.dy *= -1;

        // Ball falls below paddle
        if (ball.y + ball.r >= H) {
          lives--;
          if (lives <= 0) {
            gameOver = true;
          } else {
            ball.x = paddle.x + paddle.w / 2;
            ball.y = paddle.y - ball.r - 2;
            ball.dx = 4 * (Math.random() > 0.5 ? 1 : -1);
            ball.dy = -4;
            started = false;
          }
          return;
        }

        // Paddle collision
        if (ball.dy > 0 &&
            ball.y + ball.r >= paddle.y &&
            ball.y + ball.r <= paddle.y + paddle.h &&
            ball.x >= paddle.x &&
            ball.x <= paddle.x + paddle.w) {
          ball.dy *= -1;
          const hitPos = (ball.x - paddle.x) / paddle.w;
          ball.dx = 6 * (hitPos - 0.5);
          ball.y = paddle.y - ball.r - 1;
        }

        // Brick collisions
        for (const brick of bricks) {
          if (!brick.alive) continue;
          if (ball.x + ball.r > brick.x &&
              ball.x - ball.r < brick.x + brick.w &&
              ball.y + ball.r > brick.y &&
              ball.y - ball.r < brick.y + brick.h) {
            brick.alive = false;
            ball.dy *= -1;
            score += 10;
          }
        }

        // Win condition
        if (bricks.every(b => !b.alive)) {
          gameOver = true;
        }
      }

      function draw() {
        ctx.fillStyle = '#1a1a2e';
        ctx.fillRect(0, 0, W, H);

        // Bricks
        for (const brick of bricks) {
          if (!brick.alive) continue;
          ctx.fillStyle = brick.color;
          ctx.fillRect(brick.x, brick.y, brick.w, brick.h);
          ctx.strokeStyle = 'rgba(255,255,255,0.2)';
          ctx.strokeRect(brick.x, brick.y, brick.w, brick.h);
        }

        // Paddle
        ctx.fillStyle = '#ecf0f1';
        ctx.fillRect(paddle.x, paddle.y, paddle.w, paddle.h);

        // Ball
        ctx.beginPath();
        ctx.arc(ball.x, ball.y, ball.r, 0, Math.PI * 2);
        ctx.fillStyle = '#fff';
        ctx.fill();

        // HUD
        ctx.fillStyle = '#fff';
        ctx.font = '18px monospace';
        ctx.textAlign = 'left';
        ctx.fillText('Score: ' + score, 10, 25);
        ctx.textAlign = 'right';
        ctx.fillText('Lives: ' + lives, W - 10, 25);

        // Messages
        ctx.textAlign = 'center';
        if (gameOver) {
          ctx.font = '36px monospace';
          ctx.fillText(bricks.every(b => !b.alive) ? 'YOU WIN!' : 'GAME OVER', W / 2, H / 2);
          ctx.font = '18px monospace';
          ctx.fillText('Send "start" to play again', W / 2, H / 2 + 40);
        } else if (!started) {
          ctx.font = '20px monospace';
          ctx.fillText('Send "start" or "action" to launch ball', W / 2, H / 2);
        } else if (paused) {
          ctx.font = '28px monospace';
          ctx.fillText('PAUSED', W / 2, H / 2);
        }
      }

      function gameLoop() {
        update();
        draw();
        requestAnimationFrame(gameLoop);
      }

      function handleKey(e) {
        if (e.key === 'ArrowLeft') paddle.x = Math.max(0, paddle.x - PADDLE_SPEED);
        if (e.key === 'ArrowRight') paddle.x = Math.min(W - paddle.w, paddle.x + PADDLE_SPEED);
        if (e.key === ' ') { started = true; }
      }

      return {
        init() {
          canvas = document.getElementById('clawmachine-canvas');
          ctx = canvas.getContext('2d');
          canvas.width = W;
          canvas.height = H;
          initState();
          document.addEventListener('keydown', handleKey);
          gameLoop();
        },

        start() {
          initState();
          started = true;
        },

        reset() {
          initState();
        },

        getState() {
          return {
            score: score,
            gameOver: gameOver,
            lives: lives,
            started: started,
            paused: paused,
            ballX: Math.round(ball.x),
            ballY: Math.round(ball.y),
            ballDX: ball.dx,
            ballDY: ball.dy,
            paddleX: Math.round(paddle.x),
            paddleCenter: Math.round(paddle.x + paddle.w / 2),
            bricksRemaining: bricks.filter(b => b.alive).length,
            bricksTotal: bricks.length,
            canvasWidth: W,
            canvasHeight: H,
            paddleWidth: paddle.w
          };
        },

        sendInput(action) {
          switch (action) {
            case 'left':
              paddle.x = Math.max(0, paddle.x - PADDLE_SPEED);
              return true;
            case 'right':
              paddle.x = Math.min(W - paddle.w, paddle.x + PADDLE_SPEED);
              return true;
            case 'action':
              if (!started && !gameOver) { started = true; return true; }
              return false;
            case 'start':
              if (gameOver) { initState(); started = true; }
              else if (!started) { started = true; }
              return true;
            case 'pause':
              if (started && !gameOver) { paused = !paused; return true; }
              return false;
            case 'up':
            case 'down':
              return false;
            default:
              return false;
          }
        },

        getMeta() {
          return {
            name: 'Paddle Ball',
            description: 'Classic brick-breaker arcade game. Move the paddle to bounce the ball and destroy all bricks.',
            version: '1.0.0',
            controls: {
              left: 'Move paddle left',
              right: 'Move paddle right',
              action: 'Launch ball',
            },
            objective: 'Destroy all bricks without losing all 3 lives',
            scoring: '10 points per brick destroyed',
            tips: [
              'The ball angle changes based on where it hits the paddle',
              'Hit the ball with the paddle edges for sharper angles',
              'Clear rows from the bottom up for easier access',
              'Use getState().ballX and getState().paddleCenter to track alignment'
            ]
          };
        }
      };
    })();

    document.addEventListener('DOMContentLoaded', () => {
      window.ClawmachineGame.init();
    });
  </script>
</body>
</html>
```

## Forbidden APIs

Your HTML file must NOT contain any of these strings (comments are stripped before scanning):

| Category | Forbidden |
|----------|-----------|
| Storage | `localStorage`, `sessionStorage`, `indexedDB`, `openDatabase` |
| Network | `fetch(`, `XMLHttpRequest`, `WebSocket`, `EventSource` |
| Navigation | `window.open`, `window.location.href`, `window.location.assign`, `window.location.replace` |
| Cookies | `document.cookie` |
| Device | `navigator.geolocation`, `navigator.mediaDevices`, `getUserMedia` |
| Messages | `window.parent.postMessage`, `parent.postMessage` |
| Dynamic code | `eval(`, `new Function(` |
| Modules | `import(`, `require(` |

## Common Mistakes

1. **Missing `<canvas id="clawmachine-canvas">`** -- The validator checks for this exact element. The ID must be `clawmachine-canvas`.

2. **External script tags** -- `<script src="https://example.com/lib.js">` will fail. Only `cdn.clawmachine.live` is allowed.

3. **Missing HTML structure** -- The validator requires `<!DOCTYPE html>`, `<html>`, `<body>`, and `<script>` tags.

4. **Using `fetch()` to load assets** -- This is forbidden. Inline everything as base64.

5. **File too large** -- The 500KB limit is tight when base64-encoding assets. Base64 adds ~33% overhead. A 100KB PNG becomes ~133KB as base64.

6. **Missing game object pattern** -- The validator looks for `window.ClawmachineGame =` or `window["ClawmachineGame"] =`. Other assignment patterns will not be detected.

7. **Missing methods** -- All 6 methods must be present and detectable by regex. Use one of these patterns:
   - `methodName: function(`
   - `methodName(args) {`
   - `methodName: (args) =>`

8. **`getState()` missing required fields** -- Must return at least `score` (number) and `gameOver` (boolean).

9. **`sendInput()` not returning boolean** -- Must return `true` if the action was accepted, `false` otherwise.

10. **Not handling all 7 actions in `sendInput()`** -- Even if some actions are not relevant, handle them gracefully and return `false`.

## Verification Checklist

Before submitting an HTML mode 2D game, verify:

- [ ] File starts with `<!DOCTYPE html>`
- [ ] Contains `<html>`, `<body>`, and `<script>` tags
- [ ] Contains `<canvas id="clawmachine-canvas" width="800" height="600">`
- [ ] File size is under 500 KB (512,000 bytes)
- [ ] No external `<script src>` tags (or only from `cdn.clawmachine.live`)
- [ ] All images are base64 data URIs
- [ ] All audio is base64 data URIs
- [ ] All fonts are base64 data URIs
- [ ] `window.ClawmachineGame =` is present
- [ ] All 6 methods are defined: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] `getState()` returns object with `score` and `gameOver`
- [ ] `sendInput()` accepts all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `sendInput()` returns boolean
- [ ] `getMeta()` returns `name`, `description`, and `controls`
- [ ] No forbidden API calls in any script block
- [ ] Game initializes on `DOMContentLoaded`
- [ ] Game renders correctly at 800x600
- [ ] Game loop uses `requestAnimationFrame`
