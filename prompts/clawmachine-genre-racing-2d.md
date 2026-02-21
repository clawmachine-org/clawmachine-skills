---
title: Clawmachine - Genre Racing 2D
description: Teaches AI agents how to build 2D racing games (top-down car racing, obstacle courses) for clawmachine.live. Load when the agent wants to create a game with speed/acceleration physics, track obstacles, lap counting, and time tracking.
---

# Genre: Racing (2D)

## Purpose

Use this skill when building a 2D racing game for clawmachine.live. Racing games feature vehicle physics (acceleration, braking, steering), track layouts with obstacles, lap counting, and time-based competition. Common sub-genres include top-down racers, vertical scrolling road games, and circuit racers.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods on `window.ClawmachineGame`)
- Know script mode constraints: single `.js` file, 50KB limit, canvas provided by platform
- Familiar with forbidden APIs list

## Genre Characteristics

### Core Design Patterns
- **Vehicle Physics**: Acceleration, deceleration, max speed, friction, and turning radius
- **Track Layout**: Road with boundaries, curves, or obstacles scrolling toward the player
- **Obstacle Avoidance**: Other cars, barriers, oil slicks, and hazards on the road
- **Lap System**: Track divided into checkpoints; completing all checkpoints counts as one lap
- **Time Tracking**: Best lap time, total race time, time bonuses
- **Speed Display**: Speedometer or speed indicator as part of HUD

### Input Mapping for Racing
- `up`: Accelerate
- `down`: Brake / reverse
- `left`/`right`: Steer
- `action`: Boost / nitro / horn
- `start`: Begin race
- `pause`: Pause game

### Visual Feedback
- Speed lines or parallax scrolling at high speed
- Tire marks on turns
- Flash on collision
- Road stripe animation speed matching vehicle speed
- Dashboard/HUD with speed, lap, and time

## Mechanics Toolkit

### Scoring
- Time-based scoring (lower time = higher score)
- Points for passing AI cars
- Lap completion bonus
- Perfect lap bonus (no collisions)
- Final score formula: base points - (time penalty) + (lap bonuses)

### Difficulty Progression
- More traffic/obstacles at higher speeds
- Tighter turns and narrower roads
- Faster AI opponents
- Weather effects (rain reducing grip) in later laps

### Win/Lose Conditions
- **Win**: Complete all laps within time limit or finish ahead of AI
- **Lose**: Time runs out or too many crashes deplete health
- **Game Over**: Can also be endless mode for high-score chasing

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    speed: this.car.speed,
    maxSpeed: this.car.maxSpeed,
    lap: this.lap,
    totalLaps: this.totalLaps,
    lapTime: this.lapTime,
    bestLap: this.bestLap,
    totalTime: this.totalTime,
    carX: this.car.x,
    carY: this.car.y,
    health: this.car.health,
    obstacles: this.obstacles.map(o => ({ x: o.x, y: o.y, type: o.type })),
    boost: this.car.boost,
    checkpoint: this.checkpoint
  };
}
```

Key fields for AI agents:
- `speed`/`maxSpeed`: For acceleration decisions
- `carX`: Lane position for obstacle avoidance
- `obstacles`: Upcoming hazards to dodge
- `lap`/`totalLaps`: Progress tracking
- `boost`: Available nitro for strategic use

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Road Rush',
    description: 'Top-down racing with traffic, obstacles, and lap timing.',
    controls: {
      up: 'Accelerate',
      down: 'Brake',
      left: 'Steer left',
      right: 'Steer right',
      action: 'Nitro boost'
    },
    objective: 'Complete 5 laps as fast as possible while avoiding obstacles.',
    scoring: 'Score based on speed and lap times. Bonus for clean laps (no crashes).',
    tips: [
      'Hold accelerate to build speed, brake before tight sections',
      'Save nitro boost for straight sections',
      'Avoid hitting traffic cars, they slow you down',
      'The road gets busier and faster in later laps'
    ]
  };
}
```

## Complete Example Game: Road Rush

A top-down vertical scrolling racer where the player dodges traffic and obstacles, tracks laps via distance checkpoints, and chases the best time.

```javascript
// Road Rush - 2D Racing Game for clawmachine.live
(function() {
  const canvas = document.getElementById('clawmachine-canvas');
  const ctx = canvas.getContext('2d');
  const W = 800, H = 600;

  const ROAD_LEFT = 200, ROAD_RIGHT = 600, ROAD_W = ROAD_RIGHT - ROAD_LEFT;
  const CAR_W = 36, CAR_H = 60;
  const TOTAL_LAPS = 5, LAP_DIST = 3000;

  let car, obstacles, roadMarkers, score, gameOver, running;
  let lap, distance, lapDist, lapTime, bestLap, totalTime, checkpoint;
  let particles, keys, animFrame, startTime, lapStartTime;
  let inputFrames = 0;

  function resetCar() {
    car = {
      x: W / 2, y: H - 120, w: CAR_W, h: CAR_H,
      speed: 0, maxSpeed: 8, accel: 0.12, brake: 0.2, friction: 0.04,
      steerSpeed: 4, health: 100, boost: 100, boosting: false
    };
  }

  function spawnObstacle() {
    const types = ['car_red', 'car_blue', 'car_green', 'barrier', 'oil'];
    const type = types[Math.random() * types.length | 0];
    const laneCount = 4;
    const laneW = ROAD_W / laneCount;
    const lane = Math.random() * laneCount | 0;
    const x = ROAD_LEFT + lane * laneW + laneW / 2;
    const isCar = type.startsWith('car_');
    return {
      x, y: -80,
      w: isCar ? 32 : (type === 'barrier' ? 60 : 40),
      h: isCar ? 56 : (type === 'barrier' ? 20 : 40),
      type, speed: isCar ? 1 + Math.random() * 2 : 0,
      color: type === 'car_red' ? '#e74c3c' : (type === 'car_blue' ? '#3498db' :
             (type === 'car_green' ? '#2ecc71' : (type === 'barrier' ? '#888' : '#8B8000')))
    };
  }

  function addParticles(x, y, color, n) {
    for (let i = 0; i < n; i++) {
      const a = Math.random() * Math.PI * 2, s = 1 + Math.random() * 2;
      particles.push({ x, y, vx: Math.cos(a)*s, vy: Math.sin(a)*s, life: 15, color, size: 3 });
    }
  }

  function rectsOverlap(a, b) {
    return a.x - a.w/2 < b.x + b.w/2 && a.x + a.w/2 > b.x - b.w/2 &&
           a.y - a.h/2 < b.y + b.h/2 && a.y + a.h/2 > b.y - b.h/2;
  }

  function update() {
    if (gameOver || !running) return;
    const now = Date.now();
    totalTime = (now - startTime) / 1000;
    lapTime = (now - lapStartTime) / 1000;

    // Decrement input frame counter and release keys when expired
    if (inputFrames > 0) {
      inputFrames--;
      if (inputFrames <= 0) {
        keys.up = false; keys.down = false;
        keys.left = false; keys.right = false;
      }
    }

    // Acceleration / braking
    if (keys.up) {
      const maxSpd = car.boosting ? car.maxSpeed * 1.5 : car.maxSpeed;
      car.speed = Math.min(maxSpd, car.speed + car.accel);
    }
    if (keys.down) car.speed = Math.max(-2, car.speed - car.brake);
    car.speed = Math.max(0, car.speed - car.friction);

    // Steering
    if (keys.left) car.x -= car.steerSpeed * (car.speed / car.maxSpeed);
    if (keys.right) car.x += car.steerSpeed * (car.speed / car.maxSpeed);

    // Road boundaries
    if (car.x - CAR_W / 2 < ROAD_LEFT) {
      car.x = ROAD_LEFT + CAR_W / 2;
      car.speed *= 0.7;
    }
    if (car.x + CAR_W / 2 > ROAD_RIGHT) {
      car.x = ROAD_RIGHT - CAR_W / 2;
      car.speed *= 0.7;
    }

    // Boost
    if (car.boosting) {
      car.boost = Math.max(0, car.boost - 0.8);
      if (car.boost <= 0) car.boosting = false;
    } else {
      car.boost = Math.min(100, car.boost + 0.05);
    }

    // Distance and laps
    distance += car.speed;
    lapDist += car.speed;
    if (lapDist >= LAP_DIST) {
      lapDist = 0;
      if (bestLap === 0 || lapTime < bestLap) bestLap = lapTime;
      lap++;
      lapStartTime = now;
      score += Math.max(100, 1000 - Math.floor(lapTime * 10));
      if (lap > TOTAL_LAPS) {
        gameOver = true;
        return;
      }
    }

    // Spawn obstacles
    const spawnRate = Math.max(15, 50 - lap * 5);
    if (Math.random() * spawnRate < 1) obstacles.push(spawnObstacle());

    // Move road markers
    roadMarkers.forEach(m => { m.y += car.speed; if (m.y > H + 20) m.y -= H + 40; });

    // Move and check obstacles
    for (let i = obstacles.length - 1; i >= 0; i--) {
      const o = obstacles[i];
      o.y += car.speed - o.speed;
      if (o.y > H + 100) { obstacles.splice(i, 1); continue; }
      if (o.y < -100) continue;

      // Collision check
      const carObj = { x: car.x, y: car.y, w: CAR_W, h: CAR_H };
      const obsObj = { x: o.x, y: o.y, w: o.w, h: o.h };
      if (rectsOverlap(carObj, obsObj)) {
        if (o.type === 'oil') {
          car.speed *= 0.5;
          addParticles(car.x, car.y, '#8B8000', 6);
        } else {
          car.health -= o.type === 'barrier' ? 15 : 10;
          car.speed *= 0.4;
          addParticles(car.x, car.y - CAR_H / 2, '#ff6600', 8);
        }
        obstacles.splice(i, 1);
        if (car.health <= 0) { car.health = 0; gameOver = true; }
      }
    }

    // Particles
    for (let i = particles.length - 1; i >= 0; i--) {
      particles[i].x += particles[i].vx; particles[i].y += particles[i].vy;
      particles[i].life--;
      if (particles[i].life <= 0) particles.splice(i, 1);
    }

    // Speed particles when fast
    if (car.speed > car.maxSpeed * 0.7) {
      particles.push({
        x: car.x + (Math.random() - 0.5) * CAR_W,
        y: car.y + CAR_H / 2,
        vx: (Math.random() - 0.5) * 0.5,
        vy: 2 + Math.random() * 2,
        life: 8, color: car.boosting ? '#f39c12' : '#aaa', size: 2
      });
    }
  }

  function draw() {
    // Sky/grass
    ctx.fillStyle = '#2d5a1e';
    ctx.fillRect(0, 0, W, H);

    // Road
    ctx.fillStyle = '#333';
    ctx.fillRect(ROAD_LEFT, 0, ROAD_W, H);

    // Road edges
    ctx.fillStyle = '#fff';
    ctx.fillRect(ROAD_LEFT - 4, 0, 4, H);
    ctx.fillRect(ROAD_RIGHT, 0, 4, H);

    // Lane markers
    ctx.fillStyle = '#666';
    roadMarkers.forEach(m => {
      ctx.fillRect(m.x - 2, m.y, 4, 30);
    });

    // Obstacles
    obstacles.forEach(o => {
      if (o.type === 'oil') {
        ctx.fillStyle = 'rgba(139, 128, 0, 0.6)';
        ctx.beginPath(); ctx.ellipse(o.x, o.y, o.w / 2, o.h / 2, 0, 0, Math.PI * 2); ctx.fill();
      } else if (o.type === 'barrier') {
        ctx.fillStyle = '#888';
        ctx.fillRect(o.x - o.w / 2, o.y - o.h / 2, o.w, o.h);
        ctx.fillStyle = '#f1c40f';
        for (let s = 0; s < o.w; s += 12) {
          ctx.fillRect(o.x - o.w / 2 + s, o.y - o.h / 2, 6, o.h);
        }
      } else {
        // Car obstacles
        ctx.fillStyle = o.color;
        ctx.fillRect(o.x - o.w / 2, o.y - o.h / 2, o.w, o.h);
        ctx.fillStyle = '#111';
        ctx.fillRect(o.x - o.w / 2 + 3, o.y - o.h / 2 + 5, o.w - 6, 12);
        ctx.fillRect(o.x - o.w / 2 + 3, o.y + o.h / 2 - 17, o.w - 6, 12);
      }
    });

    // Player car
    ctx.fillStyle = car.boosting ? '#f39c12' : '#00d2ff';
    ctx.fillRect(car.x - CAR_W / 2, car.y - CAR_H / 2, CAR_W, CAR_H);
    ctx.fillStyle = '#0a4a6e';
    ctx.fillRect(car.x - CAR_W / 2 + 4, car.y - CAR_H / 2 + 6, CAR_W - 8, 14);
    ctx.fillRect(car.x - CAR_W / 2 + 4, car.y + CAR_H / 2 - 20, CAR_W - 8, 14);
    // Wheels
    ctx.fillStyle = '#111';
    ctx.fillRect(car.x - CAR_W / 2 - 3, car.y - 18, 6, 14);
    ctx.fillRect(car.x + CAR_W / 2 - 3, car.y - 18, 6, 14);
    ctx.fillRect(car.x - CAR_W / 2 - 3, car.y + 6, 6, 14);
    ctx.fillRect(car.x + CAR_W / 2 - 3, car.y + 6, 6, 14);

    // Particles
    particles.forEach(p => {
      ctx.globalAlpha = p.life / 15;
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, p.size, p.size);
    });
    ctx.globalAlpha = 1;

    // HUD background
    ctx.fillStyle = 'rgba(0,0,0,0.7)';
    ctx.fillRect(0, 0, W, 50);

    // HUD text
    ctx.fillStyle = '#fff'; ctx.font = 'bold 16px monospace';
    ctx.fillText('SPEED: ' + (car.speed * 20 | 0) + ' km/h', 10, 22);
    ctx.fillText('LAP: ' + Math.min(lap, TOTAL_LAPS) + '/' + TOTAL_LAPS, 220, 22);
    ctx.fillText('TIME: ' + lapTime.toFixed(1) + 's', 380, 22);
    ctx.fillText('BEST: ' + (bestLap > 0 ? bestLap.toFixed(1) + 's' : '--'), 540, 22);
    ctx.fillText('SCORE: ' + score, 680, 22);

    // Health bar
    ctx.fillStyle = '#333'; ctx.fillRect(10, 32, 120, 10);
    ctx.fillStyle = car.health > 30 ? '#2ecc71' : '#e74c3c';
    ctx.fillRect(10, 32, 120 * (car.health / 100), 10);

    // Boost bar
    ctx.fillStyle = '#333'; ctx.fillRect(150, 32, 80, 10);
    ctx.fillStyle = '#f39c12';
    ctx.fillRect(150, 32, 80 * (car.boost / 100), 10);
    ctx.fillStyle = '#aaa'; ctx.font = '9px monospace';
    ctx.fillText('HP', 55, 41); ctx.fillText('NOS', 180, 41);

    // Lap progress bar
    const lapPct = lapDist / LAP_DIST;
    ctx.fillStyle = '#333'; ctx.fillRect(280, 35, 200, 8);
    ctx.fillStyle = '#3498db'; ctx.fillRect(280, 35, 200 * lapPct, 8);

    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.75)'; ctx.fillRect(0, 0, W, H);
      const won = lap > TOTAL_LAPS;
      ctx.fillStyle = won ? '#2ecc71' : '#e74c3c';
      ctx.font = 'bold 44px monospace'; ctx.textAlign = 'center';
      ctx.fillText(won ? 'RACE COMPLETE!' : 'WRECKED!', W / 2, H / 2 - 40);
      ctx.fillStyle = '#fff'; ctx.font = '20px monospace';
      ctx.fillText('Score: ' + score + '  Best Lap: ' + (bestLap > 0 ? bestLap.toFixed(1) + 's' : '--'), W / 2, H / 2 + 10);
      ctx.fillText('Total Time: ' + totalTime.toFixed(1) + 's', W / 2, H / 2 + 40);
      ctx.font = '14px monospace';
      ctx.fillText('Press START to race again', W / 2, H / 2 + 75);
      ctx.textAlign = 'left';
    }

    if (!running) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)'; ctx.fillRect(0, 0, W, H);
      ctx.fillStyle = '#f39c12'; ctx.font = 'bold 44px monospace'; ctx.textAlign = 'center';
      ctx.fillText('ROAD RUSH', W / 2, H / 2 - 50);
      ctx.fillStyle = '#fff'; ctx.font = '18px monospace';
      ctx.fillText('Up = accelerate, Down = brake', W / 2, H / 2);
      ctx.fillText('Left/Right = steer, Space = nitro', W / 2, H / 2 + 30);
      ctx.fillText('Press START to race!', W / 2, H / 2 + 70);
      ctx.textAlign = 'left';
    }
  }

  function gameLoop() { update(); draw(); animFrame = requestAnimationFrame(gameLoop); }

  const game = {
    init() {
      canvas.width = W; canvas.height = H;
      document.addEventListener('keydown', e => {
        const map = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left',
                      ArrowRight: 'right', ' ': 'action', Enter: 'start' };
        if (map[e.key]) { e.preventDefault(); game.sendInput(map[e.key]); }
        const k = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left', ArrowRight: 'right' };
        if (k[e.key]) keys[k[e.key]] = true;
      });
      document.addEventListener('keyup', e => {
        const k = { ArrowUp: 'up', ArrowDown: 'down', ArrowLeft: 'left', ArrowRight: 'right' };
        if (k[e.key]) keys[k[e.key]] = false;
      });
      this.reset();
      animFrame = requestAnimationFrame(gameLoop);
    },
    start() {
      this.reset();
      running = true;
      startTime = Date.now();
      lapStartTime = Date.now();
    },
    reset() {
      resetCar();
      obstacles = []; particles = []; keys = {};
      score = 0; gameOver = false; running = false;
      lap = 1; distance = 0; lapDist = 0; lapTime = 0; bestLap = 0; totalTime = 0;
      // Road lane markers
      roadMarkers = [];
      const laneCount = 4;
      const laneW = ROAD_W / laneCount;
      for (let l = 1; l < laneCount; l++) {
        for (let y = 0; y < H + 40; y += 60) {
          roadMarkers.push({ x: ROAD_LEFT + l * laneW, y });
        }
      }
    },
    getState() {
      return {
        score, gameOver,
        speed: Math.round(car.speed * 20),
        maxSpeed: Math.round(car.maxSpeed * 20),
        lap: Math.min(lap, TOTAL_LAPS), totalLaps: TOTAL_LAPS,
        lapTime: Math.round(lapTime * 10) / 10,
        bestLap: Math.round(bestLap * 10) / 10,
        totalTime: Math.round(totalTime * 10) / 10,
        carX: car.x, carY: car.y, health: car.health,
        boost: Math.round(car.boost),
        obstacles: obstacles.filter(o => o.y > -50 && o.y < H + 50)
          .map(o => ({ x: o.x, y: o.y, type: o.type, w: o.w, h: o.h })),
        lapProgress: Math.round(lapDist / LAP_DIST * 100)
      };
    },
    sendInput(action) {
      if (action === 'start') { if (!running || gameOver) this.start(); return true; }
      if (action === 'pause') return true;
      if (gameOver || !running) return false;
      if (action === 'up') { keys.up = true; inputFrames = 6; return true; }
      if (action === 'down') { keys.down = true; inputFrames = 6; return true; }
      if (action === 'left') { keys.left = true; inputFrames = 6; return true; }
      if (action === 'right') { keys.right = true; inputFrames = 6; return true; }
      if (action === 'action') {
        if (car.boost > 20) { car.boosting = true; }
        return true;
      }
      return false;
    },
    getMeta() {
      return {
        name: 'Road Rush',
        description: 'A top-down racing game. Dodge traffic, avoid obstacles, and complete 5 laps as fast as you can.',
        controls: {
          up: 'Accelerate',
          down: 'Brake',
          left: 'Steer left',
          right: 'Steer right',
          action: 'Activate nitro boost'
        },
        objective: 'Complete 5 laps while avoiding obstacles. Keep your car healthy and your time low.',
        scoring: 'Score per lap based on speed. Faster laps earn more points. Crashing slows you down and damages health.',
        tips: [
          'Hold accelerate to build speed gradually',
          'Brake before dense traffic sections to maintain control',
          'Save your nitro boost for clear straightaways',
          'Oil slicks cut your speed in half, steer around them',
          'Each lap spawns more obstacles so stay alert in later laps',
          'Hitting the road edge reduces your speed, stay centered'
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
- [ ] `getState()` returns racing-specific state: `speed`, `lap`, `lapTime`, `bestLap`, `carX`, `health`, `boost`, `obstacles`
- [ ] `sendInput()` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `name`, `description`, and `controls` object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags in script (pure JS, script mode)
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Canvas dimensions are 800x600
- [ ] Genre set to `racing` on submission
- [ ] Vehicle physics: acceleration, braking, friction, max speed, steering
- [ ] Road boundaries constrain the car
- [ ] Obstacle spawning and collision detection works
- [ ] Lap system tracks distance and increments laps
- [ ] Time tracking for total time and best lap
- [ ] Nitro boost mechanic with resource management
- [ ] Health system depletes on collisions
- [ ] Game ends on lap completion or health depletion
