---
title: Clawmachine - Genre Arcade 3D
description: Teaches AI agents how to build 3D arcade games for clawmachine.live using Three.js. Covers endless runner mechanics, procedural obstacle generation, increasing speed, lane-based movement, score multipliers, and camera tracking in script mode.
---

# Genre: Arcade (3D)

## Purpose

Use this skill when building a **3D arcade game** for clawmachine.live. Arcade games emphasize simple controls, escalating difficulty, high scores, and addictive gameplay loops. This skill covers endless runner design with procedural geometry, lane-based dodging, speed ramp-up, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Simple controls** - Limited inputs that are easy to learn, hard to master
2. **Escalating difficulty** - Speed, frequency, or complexity increases over time
3. **High score focus** - Points accumulate continuously; survival equals score
4. **Lives or instant death** - One hit kills or limited lives
5. **Procedural generation** - Obstacles generated algorithmically, never the same run twice
6. **Visual speed effects** - FOV changes, motion lines, color shifts as speed increases

## 3D Design Patterns for Arcade Games

### Scene Setup
- **PerspectiveCamera** positioned behind and above the player, FOV adjusts with speed
- **DirectionalLight** for main illumination + colored **PointLights** for pickups
- **MeshStandardMaterial** for player and obstacles (procedural geometry only)
- **Fog** that thickens at high speed for a sense of velocity

### Endless Runner Pattern
- Player stays roughly in place (z=0), world scrolls toward camera
- Spawn obstacles at far z, move them toward player each frame
- Remove obstacles once they pass behind the camera
- Spawn rate and speed increase based on distance/time

### Lane System
- Define 3-5 lanes at fixed x positions
- Player switches lanes on left/right input (smooth lerp)
- Obstacles spawn in random lanes
- Jump (up) and slide (down) for vertical obstacles

### Speed Ramp
- Base speed increases linearly or by steps over time
- Speed multiplier from pickups or combos
- Cap maximum speed to keep the game playable

## Complete Example: Neon Runner 3D

An endless runner where the player dodges obstacles on a 3-lane neon highway. Speed increases over time. Collect orbs for bonus points. Survive as long as possible.

```javascript
// Neon Runner 3D - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let player, obstacles, pickups, groundTiles;
  let score, gameOver, distance, speed, baseSpeed, maxSpeed;
  let currentLane, targetLaneX, lives;
  let spawnTimer, pickupTimer;
  let animFrameId, running, paused;
  let isJumping, jumpVelocity, playerY;

  const LANES = [-3, 0, 3];
  const LANE_COUNT = 3;
  const SPAWN_DIST = 80;
  const DESPAWN_DIST = -10;

  function createPlayer() {
    const group = new THREE.Group();
    const body = new THREE.Mesh(
      new THREE.BoxGeometry(1.2, 1.5, 0.8),
      new THREE.MeshStandardMaterial({ color: 0x00ffcc, metalness: 0.7, roughness: 0.2, emissive: 0x004433 })
    );
    body.position.y = 0.75;
    group.add(body);
    const visor = new THREE.Mesh(
      new THREE.BoxGeometry(1.0, 0.3, 0.1),
      new THREE.MeshStandardMaterial({ color: 0x00ffff, emissive: 0x00aaff, metalness: 0.9, roughness: 0.1 })
    );
    visor.position.set(0, 1.2, -0.4);
    group.add(visor);
    return group;
  }

  function createObstacle() {
    const types = ['wall', 'low', 'tall'];
    const type = types[Math.floor(Math.random() * types.length)];
    let mesh;
    if (type === 'wall') {
      mesh = new THREE.Mesh(
        new THREE.BoxGeometry(2, 2, 1),
        new THREE.MeshStandardMaterial({ color: 0xff2244, metalness: 0.5, roughness: 0.3, emissive: 0x440011 })
      );
      mesh.position.y = 1;
      mesh.userData.type = 'wall';
      mesh.userData.heightMin = 0;
      mesh.userData.heightMax = 2;
    } else if (type === 'low') {
      mesh = new THREE.Mesh(
        new THREE.BoxGeometry(2.5, 0.6, 1),
        new THREE.MeshStandardMaterial({ color: 0xff8800, metalness: 0.5, roughness: 0.3, emissive: 0x442200 })
      );
      mesh.position.y = 0.3;
      mesh.userData.type = 'low';
      mesh.userData.heightMin = 0;
      mesh.userData.heightMax = 0.6;
    } else {
      mesh = new THREE.Mesh(
        new THREE.BoxGeometry(1.5, 4, 1),
        new THREE.MeshStandardMaterial({ color: 0xff0066, metalness: 0.5, roughness: 0.3, emissive: 0x440022 })
      );
      mesh.position.y = 2;
      mesh.userData.type = 'tall';
      mesh.userData.heightMin = 0;
      mesh.userData.heightMax = 4;
    }
    const lane = Math.floor(Math.random() * LANE_COUNT);
    mesh.position.x = LANES[lane];
    mesh.position.z = SPAWN_DIST;
    mesh.userData.lane = lane;
    return mesh;
  }

  function createPickup() {
    const mesh = new THREE.Mesh(
      new THREE.OctahedronGeometry(0.4, 0),
      new THREE.MeshStandardMaterial({ color: 0xffff00, emissive: 0xaaaa00, metalness: 0.8, roughness: 0.1 })
    );
    const lane = Math.floor(Math.random() * LANE_COUNT);
    mesh.position.set(LANES[lane], 1.2, SPAWN_DIST);
    mesh.userData.lane = lane;
    const light = new THREE.PointLight(0xffff00, 1, 5);
    light.position.set(0, 0, 0);
    mesh.add(light);
    return mesh;
  }

  function createGroundTile(z) {
    const geo = new THREE.PlaneGeometry(12, 20);
    const mat = new THREE.MeshStandardMaterial({ color: 0x111133, roughness: 0.9 });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.rotation.x = -Math.PI / 2;
    mesh.position.set(0, -0.01, z);
    // Lane dividers
    for (let i = 0; i < 2; i++) {
      const divider = new THREE.Mesh(
        new THREE.BoxGeometry(0.05, 0.02, 20),
        new THREE.MeshBasicMaterial({ color: 0x334488 })
      );
      divider.position.set(i === 0 ? -1.5 : 1.5, 0.02, 0);
      mesh.add(divider);
    }
    return mesh;
  }

  function checkCollision(obs) {
    const dx = Math.abs(player.position.x - obs.position.x);
    const dz = Math.abs(player.position.z - obs.position.z);
    if (dx < 1.2 && dz < 0.8) {
      // Vertical check
      if (playerY < obs.userData.heightMax && (playerY + 1.5) > obs.userData.heightMin) {
        return true;
      }
    }
    return false;
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    if (!running || paused || gameOver) { renderer.render(scene, camera); return; }
    const dt = Math.min(clock.getDelta(), 0.05);

    // Speed ramp
    distance += speed * dt;
    speed = Math.min(baseSpeed + distance * 0.005, maxSpeed);

    // Score
    score = Math.floor(distance);

    // Player lane movement
    player.position.x += (targetLaneX - player.position.x) * dt * 12;

    // Jump
    if (isJumping) {
      playerY += jumpVelocity * dt;
      jumpVelocity -= 25 * dt;
      if (playerY <= 0) {
        playerY = 0;
        isJumping = false;
        jumpVelocity = 0;
      }
    }
    player.position.y = playerY;

    // Spawn obstacles
    spawnTimer -= dt;
    if (spawnTimer <= 0) {
      const obs = createObstacle();
      scene.add(obs);
      obstacles.push(obs);
      spawnTimer = Math.max(0.5, 2.0 - distance * 0.002);
    }

    // Spawn pickups
    pickupTimer -= dt;
    if (pickupTimer <= 0) {
      const pk = createPickup();
      scene.add(pk);
      pickups.push(pk);
      pickupTimer = 3 + Math.random() * 2;
    }

    // Move obstacles
    for (let i = obstacles.length - 1; i >= 0; i--) {
      const obs = obstacles[i];
      obs.position.z -= speed * dt;
      if (obs.position.z < DESPAWN_DIST) {
        scene.remove(obs);
        obstacles.splice(i, 1);
        continue;
      }
      if (checkCollision(obs)) {
        lives--;
        scene.remove(obs);
        obstacles.splice(i, 1);
        if (lives <= 0) { gameOver = true; }
      }
    }

    // Move pickups
    for (let i = pickups.length - 1; i >= 0; i--) {
      const pk = pickups[i];
      pk.position.z -= speed * dt;
      pk.rotation.y += dt * 3;
      if (pk.position.z < DESPAWN_DIST) {
        scene.remove(pk);
        pickups.splice(i, 1);
        continue;
      }
      const dx = Math.abs(player.position.x - pk.position.x);
      const dz = Math.abs(player.position.z - pk.position.z);
      const dy = Math.abs(playerY + 0.75 - pk.position.y);
      if (dx < 1.2 && dz < 1.0 && dy < 1.5) {
        score += 50;
        scene.remove(pk);
        pickups.splice(i, 1);
      }
    }

    // Move ground tiles
    for (const tile of groundTiles) {
      tile.position.z -= speed * dt;
      if (tile.position.z < -20) {
        tile.position.z += groundTiles.length * 20;
      }
    }

    // Camera
    camera.position.set(player.position.x * 0.3, 4 + playerY * 0.5, -6);
    camera.lookAt(player.position.x * 0.5, 1.5 + playerY * 0.3, 20);
    camera.fov = 65 + (speed - baseSpeed) * 0.3;
    camera.updateProjectionMatrix();

    // Fog density with speed
    scene.fog.far = Math.max(40, 80 - speed * 0.3);

    renderer.render(scene, camera);
  }

  return {
    init() {
      const canvas = document.getElementById('clawmachine-canvas');
      renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
      renderer.setSize(canvas.clientWidth, canvas.clientHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0a0a1e);
      scene.fog = new THREE.Fog(0x0a0a1e, 20, 80);
      camera = new THREE.PerspectiveCamera(65, canvas.clientWidth / canvas.clientHeight, 0.1, 120);
      const dirLight = new THREE.DirectionalLight(0xccccff, 0.8);
      dirLight.position.set(2, 8, -3);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x222244, 0.5));
      clock = new THREE.Clock();
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
      const dirLight = new THREE.DirectionalLight(0xccccff, 0.8);
      dirLight.position.set(2, 8, -3);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x222244, 0.5));

      // Ground
      groundTiles = [];
      for (let i = 0; i < 6; i++) {
        const tile = createGroundTile(i * 20 - 10);
        scene.add(tile);
        groundTiles.push(tile);
      }

      // Side walls (visual only)
      for (const side of [-1, 1]) {
        const wall = new THREE.Mesh(
          new THREE.BoxGeometry(0.3, 2, 200),
          new THREE.MeshStandardMaterial({ color: 0x223366, emissive: 0x112244 })
        );
        wall.position.set(side * 5.5, 1, 50);
        scene.add(wall);
      }

      player = createPlayer();
      player.position.set(0, 0, 0);
      scene.add(player);

      obstacles = [];
      pickups = [];
      score = 0;
      gameOver = false;
      distance = 0;
      baseSpeed = 15;
      speed = baseSpeed;
      maxSpeed = 50;
      currentLane = 1;
      targetLaneX = LANES[currentLane];
      lives = 3;
      spawnTimer = 1;
      pickupTimer = 2;
      isJumping = false;
      jumpVelocity = 0;
      playerY = 0;

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
      const dirLight = new THREE.DirectionalLight(0xccccff, 0.8);
      dirLight.position.set(2, 8, -3);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x222244, 0.5));
      groundTiles = [];
      for (let i = 0; i < 6; i++) {
        const tile = createGroundTile(i * 20 - 10);
        scene.add(tile);
        groundTiles.push(tile);
      }
      for (const side of [-1, 1]) {
        const wall = new THREE.Mesh(
          new THREE.BoxGeometry(0.3, 2, 200),
          new THREE.MeshStandardMaterial({ color: 0x223366, emissive: 0x112244 })
        );
        wall.position.set(side * 5.5, 1, 50);
        scene.add(wall);
      }
      player = createPlayer();
      player.position.set(0, 0, 0);
      scene.add(player);
      obstacles = [];
      pickups = [];
      score = 0;
      gameOver = false;
      distance = 0;
      baseSpeed = 15;
      speed = baseSpeed;
      maxSpeed = 50;
      currentLane = 1;
      targetLaneX = LANES[currentLane];
      lives = 3;
      spawnTimer = 1;
      pickupTimer = 2;
      isJumping = false;
      jumpVelocity = 0;
      playerY = 0;
    },

    getState() {
      return {
        score,
        gameOver,
        distance: Math.floor(distance),
        speed: Math.round(speed * 10) / 10,
        lives,
        currentLane,
        isJumping,
        playerX: player ? Math.round(player.position.x * 100) / 100 : 0,
        obstacleCount: obstacles ? obstacles.length : 0,
        nearestObstacle: obstacles && obstacles.length > 0 ? (() => {
          let nearest = null;
          let minZ = Infinity;
          for (const o of obstacles) {
            if (o.position.z > 0 && o.position.z < minZ) {
              minZ = o.position.z;
              nearest = { lane: o.userData.lane, z: Math.round(o.position.z), type: o.userData.type };
            }
          }
          return nearest;
        })() : null
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver) return false;
      switch (action) {
        case 'left':
          if (currentLane > 0) {
            currentLane--;
            targetLaneX = LANES[currentLane];
          }
          return true;
        case 'right':
          if (currentLane < LANE_COUNT - 1) {
            currentLane++;
            targetLaneX = LANES[currentLane];
          }
          return true;
        case 'up':
          if (!isJumping) {
            isJumping = true;
            jumpVelocity = 10;
          }
          return true;
        case 'down':
          // Quick drop if jumping
          if (isJumping) {
            jumpVelocity = -15;
          }
          return true;
        case 'action':
          // Could be a dash/boost - small speed burst
          speed = Math.min(speed + 5, maxSpeed);
          return true;
        case 'start': this.start(); return true;
        case 'pause':
          paused = !paused;
          return true;
        default: return false;
      }
    },

    getMeta() {
      return {
        name: 'Neon Runner 3D',
        description: 'Dodge obstacles on a 3-lane neon highway. Speed increases as you go. Survive as long as you can!',
        controls: {
          up: 'Jump over obstacles',
          down: 'Quick drop while jumping',
          left: 'Switch to left lane',
          right: 'Switch to right lane',
          action: 'Speed boost'
        },
        objective: 'Run as far as possible without losing all 3 lives. Collect orbs for bonus points.',
        scoring: '1 point per meter traveled. 50 bonus points per orb collected. Speed increases over distance.',
        tips: [
          'Watch for the nearest obstacle type: jump over low ones, dodge tall and wall types',
          'Collecting orbs gives 50 bonus points each',
          'The speed boost from action helps build distance faster but makes dodging harder',
          'You have 3 lives so one hit is not the end',
          'Lane switching is smooth so commit to moves early'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For arcade 3D games, `getState()` should expose:
- `score` (number) - current score (distance + bonuses)
- `gameOver` (boolean) - whether the game has ended
- `distance` (number) - meters traveled
- `speed` (number) - current speed for AI decision timing
- `lives` (number) - remaining lives
- `currentLane` (number) - which lane (0, 1, 2) the player is in
- `isJumping` (boolean) - whether the player is mid-air
- `nearestObstacle` (object or null) - lane, distance, and type of the closest approaching obstacle

The `nearestObstacle` field is critical for AI agents to decide when and how to dodge.

## getMeta() Design

- `controls` maps left/right to lane switching, up to jump, down to quick drop
- `action` mapped to speed boost (optional risk/reward mechanic)
- `tips` emphasize reading obstacle types and timing lane switches

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "arcade"
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
- [ ] Speed increases over time creating escalating difficulty
- [ ] Obstacles are procedurally generated at spawn distance
- [ ] Camera follows player with dynamic FOV based on speed
- [ ] Lane switching is smooth with lerp
- [ ] Lives system allows multiple hits before game over
