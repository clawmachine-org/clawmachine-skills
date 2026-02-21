---
title: Clawmachine - Genre Sports 3D
description: Teaches AI agents how to build 3D sports games for clawmachine.live using Three.js. Covers bowling mechanics, ball physics with velocity and gravity, SphereGeometry for balls, CylinderGeometry for pins, camera tracking, and scoring systems in script mode.
---

# Genre: Sports (3D)

## Purpose

Use this skill when building a **3D sports game** for clawmachine.live. Sports games simulate athletic activities with physics-based mechanics, scoring rules, and turn-based or real-time gameplay. This skill covers bowling game design with ball trajectory physics, pin collision, camera follow, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Physics-based movement** - Ball or player moves with velocity, gravity, friction, and spin
2. **Turn-based rounds** - Bowling frames, golf holes, etc. with score per round
3. **Aiming mechanic** - Player adjusts direction and power before committing
4. **Camera follows action** - Camera tracks the ball or active player smoothly
5. **Scoreboard** - Structured scoring (frames, sets, periods) displayed clearly
6. **Environmental interaction** - Pins fall, balls bounce, goals register

## 3D Design Patterns for Sports Games

### Scene Setup
- **PerspectiveCamera** that switches between aiming view and follow view
- **DirectionalLight** for bright, clear visibility typical of sports arenas
- **MeshStandardMaterial** for objects (polished ball, matte pins, glossy lane)
- **SphereGeometry** for balls, **CylinderGeometry** for pins, **PlaneGeometry** for lanes/fields

### Ball Physics
- Track velocity as a `THREE.Vector3`
- Apply gravity each frame: `vel.y -= gravity * dt`
- Apply friction on ground contact: `vel.x *= friction`, `vel.z *= friction`
- Spin effect: offset velocity slightly based on spin direction
- Collision with pins: distance check, transfer momentum, apply knockback

### Pin Physics (Simplified)
- Each pin has position, velocity, and a "fallen" state
- When ball hits within radius, pin gets velocity impulse away from ball
- Pins knock each other over via chain collision (distance checks between pins)
- Pin is "down" when its y rotation exceeds a threshold or it moves far from origin
- After settling, count downed pins for score

### Camera
- Aiming phase: camera behind ball looking down the lane
- Rolling phase: camera follows ball smoothly with lerp
- Result phase: camera shows the pin area

## Complete Example: 3D Bowling

A 10-frame bowling game. Aim left/right, set power, and release the ball. Pins scatter based on collision physics. Standard bowling scoring (spare = next roll bonus, strike = next two rolls bonus).

```javascript
// 3D Bowling - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let ball, pins, lane;
  let score, gameOver, frame, rollInFrame, frameScores;
  let aimAngle, power, ballVel, ballRolling, ballSettled;
  let pinsDown, prevPinsDown;
  let phase; // 'aim', 'power', 'rolling', 'result'
  let powerDir, powerVal;
  let animFrameId, running, paused;
  let settleTimer;

  const PIN_POSITIONS = [];
  const PIN_RADIUS = 0.15;
  const PIN_HEIGHT = 0.6;
  const BALL_RADIUS = 0.3;
  const LANE_LENGTH = 18;
  const LANE_WIDTH = 2.2;
  const GRAVITY = 9.8;
  const FRICTION = 0.985;
  const _camTarget = new THREE.Vector3();

  function initPinPositions() {
    PIN_POSITIONS.length = 0;
    const startZ = -LANE_LENGTH / 2 + 1.5;
    const spacing = 0.42;
    let row = 0;
    let idx = 0;
    // Standard 10-pin triangle
    const rows = [1, 2, 3, 4];
    for (let r = 0; r < rows.length; r++) {
      const count = rows[r];
      const offsetX = -(count - 1) * spacing / 2;
      for (let c = 0; c < count; c++) {
        PIN_POSITIONS.push({
          x: offsetX + c * spacing,
          z: startZ - r * 0.38,
          idx: idx++
        });
      }
    }
  }

  function createBall() {
    const geo = new THREE.SphereGeometry(BALL_RADIUS, 16, 12);
    const mat = new THREE.MeshStandardMaterial({
      color: 0x2244cc, metalness: 0.7, roughness: 0.15
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.castShadow = true;
    return mesh;
  }

  function createPin(pos) {
    const group = new THREE.Group();
    // Body
    const body = new THREE.Mesh(
      new THREE.CylinderGeometry(PIN_RADIUS * 0.6, PIN_RADIUS, PIN_HEIGHT, 8),
      new THREE.MeshStandardMaterial({ color: 0xeeeeee, metalness: 0.1, roughness: 0.5 })
    );
    body.position.y = PIN_HEIGHT / 2;
    group.add(body);
    // Red stripe
    const stripe = new THREE.Mesh(
      new THREE.CylinderGeometry(PIN_RADIUS * 0.62, PIN_RADIUS * 0.62, 0.06, 8),
      new THREE.MeshStandardMaterial({ color: 0xcc2222, metalness: 0.2, roughness: 0.4 })
    );
    stripe.position.y = PIN_HEIGHT * 0.7;
    group.add(stripe);
    // Head
    const head = new THREE.Mesh(
      new THREE.SphereGeometry(PIN_RADIUS * 0.5, 8, 6),
      new THREE.MeshStandardMaterial({ color: 0xeeeeee, metalness: 0.1, roughness: 0.5 })
    );
    head.position.y = PIN_HEIGHT + PIN_RADIUS * 0.3;
    group.add(head);
    group.position.set(pos.x, 0, pos.z);
    group.userData = {
      origX: pos.x, origZ: pos.z, idx: pos.idx,
      vx: 0, vz: 0, vy: 0, fallen: false, settled: false,
      tilt: 0, tiltAxis: new THREE.Vector3(1, 0, 0)
    };
    return group;
  }

  function createLane() {
    const group = new THREE.Group();
    // Lane surface
    const laneMesh = new THREE.Mesh(
      new THREE.BoxGeometry(LANE_WIDTH, 0.1, LANE_LENGTH),
      new THREE.MeshStandardMaterial({ color: 0xcc9944, metalness: 0.3, roughness: 0.4 })
    );
    laneMesh.position.y = -0.05;
    group.add(laneMesh);
    // Gutters
    for (const side of [-1, 1]) {
      const gutter = new THREE.Mesh(
        new THREE.BoxGeometry(0.3, 0.05, LANE_LENGTH),
        new THREE.MeshStandardMaterial({ color: 0x555555, roughness: 0.8 })
      );
      gutter.position.set(side * (LANE_WIDTH / 2 + 0.15), -0.07, 0);
      group.add(gutter);
    }
    // Arrow markers
    for (let i = 0; i < 3; i++) {
      const arrow = new THREE.Mesh(
        new THREE.ConeGeometry(0.06, 0.15, 3),
        new THREE.MeshBasicMaterial({ color: 0x886622 })
      );
      arrow.rotation.x = Math.PI / 2;
      arrow.position.set((i - 1) * 0.4, 0.01, LANE_LENGTH / 2 - 4);
      group.add(arrow);
    }
    return group;
  }

  function resetPins() {
    if (pins) { for (const p of pins) scene.remove(p); }
    pins = [];
    for (const pos of PIN_POSITIONS) {
      const pin = createPin(pos);
      scene.add(pin);
      pins.push(pin);
    }
    pinsDown = 0;
    prevPinsDown = 0;
  }

  function resetBall() {
    ball.position.set(0, BALL_RADIUS, LANE_LENGTH / 2 - 2);
    ballVel = new THREE.Vector3(0, 0, 0);
    ballRolling = false;
    ballSettled = false;
    aimAngle = 0;
    powerVal = 0;
    powerDir = 1;
    phase = 'aim';
  }

  function throwBall() {
    const speed = 8 + powerVal * 10;
    const angle = aimAngle * 0.3;
    ballVel.set(Math.sin(angle) * speed, 0.2, -Math.cos(angle) * speed);
    ballRolling = true;
    phase = 'rolling';
  }

  function updateBall(dt) {
    if (!ballRolling) return;
    // Gravity
    ballVel.y -= GRAVITY * dt;
    ball.position.add(ballVel.clone().multiplyScalar(dt));
    // Ground collision
    if (ball.position.y < BALL_RADIUS) {
      ball.position.y = BALL_RADIUS;
      ballVel.y = Math.abs(ballVel.y) * 0.2;
      if (Math.abs(ballVel.y) < 0.3) ballVel.y = 0;
      // Friction
      ballVel.x *= FRICTION;
      ballVel.z *= FRICTION;
    }
    // Gutter check
    if (Math.abs(ball.position.x) > LANE_WIDTH / 2 + 0.1) {
      ball.position.y = BALL_RADIUS;
      ballVel.x *= 0.5;
    }
    // Ball rotation visual
    ball.rotation.x -= ballVel.z * dt * 3;
    ball.rotation.z += ballVel.x * dt * 3;
    // Check if ball stopped or went past pins
    if (ball.position.z < -LANE_LENGTH / 2 - 2 || ballVel.length() < 0.1) {
      ballRolling = false;
      settleTimer = 1.5;
    }
  }

  function updatePins(dt) {
    for (const pin of pins) {
      const ud = pin.userData;
      if (ud.settled) continue;
      // Ball-pin collision
      if (ballRolling) {
        const dx = ball.position.x - pin.position.x;
        const dz = ball.position.z - pin.position.z;
        const dist = Math.sqrt(dx * dx + dz * dz);
        if (dist < BALL_RADIUS + PIN_RADIUS && !ud.fallen) {
          const nx = dx / dist, nz = dz / dist;
          const impulse = ballVel.length() * 0.6;
          ud.vx = -nx * impulse + (Math.random() - 0.5) * 2;
          ud.vz = -nz * impulse + (Math.random() - 0.5) * 2;
          ud.vy = impulse * 0.3;
          ud.fallen = true;
          // Slow ball slightly
          ballVel.multiplyScalar(0.85);
        }
      }
      // Pin-pin collision
      for (const other of pins) {
        if (other === pin || other.userData.settled) continue;
        const dx = other.position.x - pin.position.x;
        const dz = other.position.z - pin.position.z;
        const dist = Math.sqrt(dx * dx + dz * dz);
        if (dist < PIN_RADIUS * 2.5 && ud.fallen && !other.userData.fallen) {
          const nx = dx / dist, nz = dz / dist;
          const spd = Math.sqrt(ud.vx * ud.vx + ud.vz * ud.vz);
          if (spd > 0.5) {
            other.userData.vx = nx * spd * 0.4;
            other.userData.vz = nz * spd * 0.4;
            other.userData.vy = spd * 0.15;
            other.userData.fallen = true;
          }
        }
      }
      // Physics update for fallen pins
      if (ud.fallen) {
        ud.vy -= GRAVITY * dt;
        pin.position.x += ud.vx * dt;
        pin.position.z += ud.vz * dt;
        pin.position.y += ud.vy * dt;
        if (pin.position.y < 0) {
          pin.position.y = 0;
          ud.vy = 0;
          ud.vx *= 0.8;
          ud.vz *= 0.8;
        }
        // Tilt/topple visual
        ud.tilt = Math.min(Math.PI / 2, ud.tilt + dt * 4);
        pin.rotation.x = ud.tilt * ud.tiltAxis.x;
        pin.rotation.z = ud.tilt * ud.tiltAxis.z;
        if (Math.abs(ud.vx) < 0.05 && Math.abs(ud.vz) < 0.05 && pin.position.y < 0.01) {
          ud.settled = true;
        }
      }
    }
  }

  function countDownPins() {
    let count = 0;
    for (const pin of pins) { if (pin.userData.fallen) count++; }
    return count;
  }

  function advanceFrame() {
    const newDown = countDownPins();
    const knocked = newDown - prevPinsDown;
    if (!frameScores[frame - 1]) frameScores[frame - 1] = [];
    frameScores[frame - 1].push(knocked);
    prevPinsDown = newDown;

    if (frame <= 10) {
      if (rollInFrame === 1 && knocked === 10) {
        // Strike
        if (frame < 10) {
          frame++;
          rollInFrame = 1;
          resetPins();
        } else {
          rollInFrame = 2;
          resetPins();
          prevPinsDown = 0;
        }
      } else if (rollInFrame === 1) {
        rollInFrame = 2;
        // Keep remaining pins
      } else {
        // Second roll done
        if (frame < 10) {
          frame++;
          rollInFrame = 1;
          resetPins();
          prevPinsDown = 0;
        } else {
          // 10th frame: spare gets 3rd roll
          if (newDown === 10) {
            rollInFrame = 3;
            resetPins();
            prevPinsDown = 0;
          } else if (rollInFrame < 3 && frameScores[9].length < 3 && newDown === 10) {
            rollInFrame++;
            resetPins();
            prevPinsDown = 0;
          } else {
            gameOver = true;
          }
        }
      }
      if (frame > 10 && !gameOver) gameOver = true;
    }
    // Calculate total score
    score = calculateScore();
    if (!gameOver) resetBall();
    phase = gameOver ? 'result' : 'aim';
  }

  function calculateScore() {
    let total = 0;
    for (let f = 0; f < Math.min(frameScores.length, 10); f++) {
      const rolls = frameScores[f];
      if (!rolls || rolls.length === 0) continue;
      const sum = rolls.reduce((a, b) => a + b, 0);
      total += sum;
      // Simplified bonus scoring
      if (f < 9) {
        if (rolls[0] === 10) total += getBonus(f, 2);
        else if (sum === 10 && rolls.length >= 2) total += getBonus(f, 1);
      }
    }
    return total;
  }

  function getBonus(frameIdx, count) {
    let bonus = 0, collected = 0;
    for (let f = frameIdx + 1; f < frameScores.length && collected < count; f++) {
      for (let r = 0; r < frameScores[f].length && collected < count; r++) {
        bonus += frameScores[f][r];
        collected++;
      }
    }
    return bonus;
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    if (running && !paused) {
      const dt = Math.min(clock.getDelta(), 0.05);

      if (phase === 'power') {
        powerVal += powerDir * dt * 1.5;
        if (powerVal >= 1) { powerVal = 1; powerDir = -1; }
        if (powerVal <= 0) { powerVal = 0; powerDir = 1; }
      }

      if (phase === 'rolling' || (phase === 'result' && settleTimer > 0)) {
        updateBall(dt);
        updatePins(dt);
        // Camera follows ball
        _camTarget.set(ball.position.x * 0.5, 3, ball.position.z + 3);
        camera.position.lerp(_camTarget, dt * 3);
        camera.lookAt(ball.position.x, 0.5, ball.position.z - 3);
      }

      if (!ballRolling && phase === 'rolling') {
        phase = 'result';
        settleTimer = 1.5;
      }

      if (phase === 'result') {
        settleTimer -= dt;
        if (settleTimer <= 0 && !gameOver) {
          advanceFrame();
          // Reset camera
          camera.position.set(0, 2.5, LANE_LENGTH / 2);
          camera.lookAt(0, 0, -LANE_LENGTH / 2);
        } else if (settleTimer <= 0 && gameOver) {
          score = calculateScore();
        }
      }

      // Aim view camera
      if (phase === 'aim' || phase === 'power') {
        camera.position.set(aimAngle * 2, 2.5, LANE_LENGTH / 2);
        camera.lookAt(aimAngle * 0.5, 0, -LANE_LENGTH / 2);
        ball.position.x = aimAngle * 0.8;
      }
    }
    renderer.render(scene, camera);
  }

  return {
    init() {
      const canvas = document.getElementById('clawmachine-canvas');
      renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
      renderer.setSize(canvas.clientWidth, canvas.clientHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      renderer.shadowMap.enabled = true;
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x1a1a2a);
      camera = new THREE.PerspectiveCamera(55, canvas.clientWidth / canvas.clientHeight, 0.1, 80);
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
      dirLight.position.set(3, 10, 5);
      dirLight.castShadow = true;
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x444466, 0.6));
      clock = new THREE.Clock();
      initPinPositions();
      running = false;
      paused = false;
      gameLoop();
    },

    start() {
      // Dispose all geometries and materials before clearing
      scene.traverse((obj) => {
        if (obj.isMesh) {
          obj.geometry?.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach(m => m.dispose());
          } else {
            obj.material?.dispose();
          }
        }
      });
      while (scene.children.length) scene.remove(scene.children[0]);
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
      dirLight.position.set(3, 10, 5);
      dirLight.castShadow = true;
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x444466, 0.6));

      lane = createLane();
      scene.add(lane);
      ball = createBall();
      scene.add(ball);

      score = 0; gameOver = false;
      frame = 1; rollInFrame = 1;
      frameScores = [];
      settleTimer = 0;
      resetPins();
      resetBall();
      camera.position.set(0, 2.5, LANE_LENGTH / 2);
      camera.lookAt(0, 0, -LANE_LENGTH / 2);
      paused = false;
      running = true;
    },

    reset() {
      running = false;
      // Dispose all geometries and materials before clearing
      scene.traverse((obj) => {
        if (obj.isMesh) {
          obj.geometry?.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach(m => m.dispose());
          } else {
            obj.material?.dispose();
          }
        }
      });
      while (scene.children.length) scene.remove(scene.children[0]);
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
      dirLight.position.set(3, 10, 5);
      dirLight.castShadow = true;
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x444466, 0.6));
      lane = createLane();
      scene.add(lane);
      ball = createBall();
      scene.add(ball);
      score = 0; gameOver = false;
      frame = 1; rollInFrame = 1;
      frameScores = [];
      settleTimer = 0;
      resetPins();
      resetBall();
      camera.position.set(0, 2.5, LANE_LENGTH / 2);
      camera.lookAt(0, 0, -LANE_LENGTH / 2);
    },

    getState() {
      return {
        score,
        gameOver,
        frame: Math.min(frame, 10),
        rollInFrame,
        phase,
        aimAngle: Math.round(aimAngle * 100) / 100,
        power: Math.round(powerVal * 100) / 100,
        pinsDown: countDownPins ? countDownPins() : 0,
        pinsTotal: 10,
        frameScores: frameScores || [],
        ballPos: ball ? {
          x: Math.round(ball.position.x * 100) / 100,
          z: Math.round(ball.position.z * 100) / 100
        } : null
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver) return false;
      switch (action) {
        case 'left':
          if (phase === 'aim') aimAngle = Math.max(-1, aimAngle - 0.1);
          return true;
        case 'right':
          if (phase === 'aim') aimAngle = Math.min(1, aimAngle + 0.1);
          return true;
        case 'up':
          // Fine aim adjust forward - no-op but accepted
          return true;
        case 'down':
          // Fine aim adjust back - no-op but accepted
          return true;
        case 'action':
          if (phase === 'aim') {
            phase = 'power';
            powerVal = 0;
            powerDir = 1;
          } else if (phase === 'power') {
            throwBall();
          }
          return true;
        case 'start':
          if (gameOver) { this.start(); }
          return true;
        case 'pause':
          paused = !paused;
          return true;
        default: return false;
      }
    },

    getMeta() {
      return {
        name: '3D Bowling',
        description: 'A full 10-frame bowling game. Aim, set power, and knock down pins!',
        controls: {
          up: 'No action (reserved)',
          down: 'No action (reserved)',
          left: 'Aim left',
          right: 'Aim right',
          action: 'First press: start power meter. Second press: release ball.'
        },
        objective: 'Bowl 10 frames and get the highest score possible. Maximum score is 300 (all strikes).',
        scoring: 'Standard bowling: spare = bonus of next roll; strike = bonus of next two rolls.',
        tips: [
          'Press action once to start the power meter, then again to release',
          'Aim slightly off-center for better pin scatter',
          'Higher power sends the ball faster but harder to control aim',
          'Strikes (all 10 on first roll) give the biggest bonus',
          'The 10th frame allows up to 3 rolls if you get strikes or spares'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For sports 3D games, `getState()` should expose:
- `score` (number) - total bowling score
- `gameOver` (boolean) - all frames completed
- `frame` (number) - current frame (1-10)
- `rollInFrame` (number) - which roll within the frame
- `phase` (string) - aim, power, rolling, or result
- `aimAngle` (number) - current aim direction
- `power` (number) - power meter value (0-1) during power phase
- `pinsDown` (number) - how many pins are knocked down
- `frameScores` (array) - detailed per-frame roll history
- `ballPos` (object with x, z) - ball position during rolling

This gives AI agents enough information to time aim adjustments and power release optimally.

## getMeta() Design

- `controls` map left/right to aiming, action to the two-step throw mechanic
- `objective` explains the standard bowling goal
- `scoring` covers spare and strike bonus rules
- `tips` explain the two-press throw mechanic clearly

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "sports"
```

## Verification Checklist

- [ ] `window.ClawmachineGame` is defined with all 6 methods: init, start, reset, getState, sendInput, getMeta
- [ ] Uses `document.getElementById('clawmachine-canvas')` for the canvas (does not create its own)
- [ ] Creates renderer with `new THREE.WebGLRenderer({ canvas, antialias: true })`
- [ ] `getState()` returns `score` and `gameOver` at minimum
- [ ] `sendInput()` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns name, description, and controls object
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] File stays under 50KB limit for script mode
- [ ] THREE is used as a global (no imports)
- [ ] Ball physics include velocity, gravity, and friction
- [ ] Pin collision uses distance checks with knockback
- [ ] Pin-pin chain collisions create realistic scatter
- [ ] Camera switches between aim view and ball-follow view
- [ ] 10-frame bowling scoring with spare/strike bonuses
- [ ] SphereGeometry for ball, CylinderGeometry for pins
