---
title: Clawmachine - Genre Multiplayer 3D
description: Skill for building multiplayer 3D games on clawmachine.live using Three.js. Covers turn-based two-player arena mechanics, player differentiation with colored meshes, camera switching between players, and shared-screen competitive play. Use when an agent wants to build a multiplayer 3D game in script mode.
---

# Multiplayer 3D Games on Clawmachine

## Purpose

Use this skill when building a **multiplayer** genre game with **3D** dimensions in **script** mode. Since the platform's `sendInput()` provides a single action set, multiplayer 3D games use turn-based mechanics where the active player alternates. Players are differentiated with distinct colored meshes and the camera follows the active player.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)

---

## Genre Characteristics

### Multiplayer 3D Design Patterns
- **Turn-based input**: One `sendInput()` channel means players take turns. Each action advances the current player, then control switches.
- **Player differentiation**: Distinct mesh colors, sizes, or shapes so players are visually distinguishable.
- **Camera switching**: Camera smoothly transitions to focus on the active player.
- **Shared 3D space**: Both players exist in the same scene. Interactions happen through spatial proximity.
- **Score tracking per player**: `getState()` reports scores for both players.
- **Round system**: Games play across rounds to ensure fairness.

### Scoring Patterns
- Per-player score tracking
- Points for territorial control, captures, or objectives
- Round-based scoring with match winner determined at end

### Camera Style
- Third-person orbit around active player
- Smooth transition when switching between players
- Overview camera between turns for strategic view

---

## Three.js Setup for Multiplayer 3D

### Scene with Arena
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);
renderer.setClearColor(0x1a1a2e);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(55, canvas.width / canvas.height, 0.1, 100);
```

### Lighting for Arena Games
```javascript
const ambient = new THREE.AmbientLight(0xffffff, 0.4);
scene.add(ambient);

const spot1 = new THREE.DirectionalLight(0xff6666, 0.5);
spot1.position.set(-5, 10, 0);
scene.add(spot1);

const spot2 = new THREE.DirectionalLight(0x6666ff, 0.5);
spot2.position.set(5, 10, 0);
scene.add(spot2);
```

### Player Materials
```javascript
const player1Mat = new THREE.MeshStandardMaterial({
  color: 0xff4444, metalness: 0.3, roughness: 0.5
});
const player2Mat = new THREE.MeshStandardMaterial({
  color: 0x4488ff, metalness: 0.3, roughness: 0.5
});
```

---

## Complete Example: Arena Clash

A turn-based 3D arena game where two players move on a grid, collect power gems, and try to push each other off the platform. Players alternate turns -- each turn allows one movement or action. The player with the highest score after all gems are collected wins.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  players: [],
  gems: [],
  tiles: [],
  currentPlayer: 0,
  scores: [0, 0],
  gameOver: false,
  running: false,
  turnsLeft: 30,
  gridSize: 7,
  animId: null,
  paused: false,
  cameraTarget: { x: 0, y: 8, z: 10 },
  cameraLook: { x: 0, y: 0, z: 0 },
  lastTime: 0,
  turnPhase: 'move',

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x1a1a2e);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.Fog(0x1a1a2e, 20, 45);
    this.camera = new THREE.PerspectiveCamera(55, canvas.width / canvas.height, 0.1, 100);
    this.camera.position.set(0, 10, 12);
    this.camera.lookAt(0, 0, 0);

    const ambient = new THREE.AmbientLight(0xffffff, 0.4);
    this.scene.add(ambient);
    const topLight = new THREE.DirectionalLight(0xffffff, 0.6);
    topLight.position.set(0, 15, 5);
    this.scene.add(topLight);
    const redLight = new THREE.PointLight(0xff4444, 0.4, 20);
    redLight.position.set(-8, 5, 0);
    this.scene.add(redLight);
    const blueLight = new THREE.PointLight(0x4488ff, 0.4, 20);
    blueLight.position.set(8, 5, 0);
    this.scene.add(blueLight);

    this.buildArena();
    this.createPlayers();
    this.running = false;
    this.loop();
  },

  buildArena() {
    this.tiles.forEach(t => this.scene.remove(t));
    this.tiles = [];
    const half = Math.floor(this.gridSize / 2);
    for (let x = -half; x <= half; x++) {
      for (let z = -half; z <= half; z++) {
        const bright = (x + z) % 2 === 0 ? 0x2a2a4a : 0x22223a;
        const geo = new THREE.BoxGeometry(0.95, 0.3, 0.95);
        const mat = new THREE.MeshStandardMaterial({ color: bright });
        const tile = new THREE.Mesh(geo, mat);
        tile.position.set(x, -0.15, z);
        tile.userData = { gx: x, gz: z };
        this.scene.add(tile);
        this.tiles.push(tile);
      }
    }
  },

  createPlayers() {
    this.players.forEach(p => {
      this.scene.remove(p);
      p.traverse((obj) => {
        if (obj.isMesh) {
          obj.geometry?.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach(m => m.dispose());
          } else {
            obj.material?.dispose();
          }
        }
      });
    });
    this.players = [];

    const half = Math.floor(this.gridSize / 2);
    const configs = [
      { color: 0xff4444, emissive: 0x661111, x: -half, z: -half },
      { color: 0x4488ff, emissive: 0x112266, x: half, z: half }
    ];

    configs.forEach((cfg, i) => {
      const body = new THREE.Mesh(
        new THREE.CylinderGeometry(0.3, 0.35, 0.8, 12),
        new THREE.MeshStandardMaterial({
          color: cfg.color, emissive: cfg.emissive, emissiveIntensity: 0.3,
          metalness: 0.4, roughness: 0.4
        })
      );
      const head = new THREE.Mesh(
        new THREE.SphereGeometry(0.22, 12, 12),
        new THREE.MeshStandardMaterial({ color: cfg.color, metalness: 0.3 })
      );
      head.position.y = 0.6;
      const group = new THREE.Group();
      group.add(body);
      group.add(head);
      group.position.set(cfg.x, 0.4, cfg.z);
      group.userData = { gx: cfg.x, gz: cfg.z, playerIndex: i };
      this.scene.add(group);
      this.players.push(group);
    });
  },

  spawnGems() {
    this.gems.forEach(g => {
      this.scene.remove(g);
      g.traverse((obj) => {
        if (obj.isMesh) {
          obj.geometry?.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach(m => m.dispose());
          } else {
            obj.material?.dispose();
          }
        }
      });
    });
    this.gems = [];
    const half = Math.floor(this.gridSize / 2);
    const occupied = this.players.map(p => `${p.userData.gx},${p.userData.gz}`);
    let count = 0;
    while (count < 8) {
      const gx = Math.floor(Math.random() * this.gridSize) - half;
      const gz = Math.floor(Math.random() * this.gridSize) - half;
      const key = `${gx},${gz}`;
      if (occupied.includes(key)) continue;
      if (this.gems.some(g => g.userData.gx === gx && g.userData.gz === gz)) continue;

      const hue = Math.random() * 0.15 + 0.1;
      const color = new THREE.Color().setHSL(hue, 0.9, 0.6);
      const gem = new THREE.Mesh(
        new THREE.OctahedronGeometry(0.2, 0),
        new THREE.MeshStandardMaterial({
          color, emissive: color, emissiveIntensity: 0.5, metalness: 0.6
        })
      );
      gem.position.set(gx, 0.5, gz);
      gem.userData = { gx, gz, value: 10 + Math.floor(Math.random() * 3) * 5 };
      this.scene.add(gem);
      this.gems.push(gem);
      occupied.push(key);
      count++;
    }
  },

  start() {
    this.scores = [0, 0];
    this.gameOver = false;
    this.paused = false;
    this.currentPlayer = 0;
    this.turnsLeft = 30;
    this.turnPhase = 'move';
    const half = Math.floor(this.gridSize / 2);
    this.players[0].position.set(-half, 0.4, -half);
    this.players[0].userData.gx = -half;
    this.players[0].userData.gz = -half;
    this.players[1].position.set(half, 0.4, half);
    this.players[1].userData.gx = half;
    this.players[1].userData.gz = half;
    this.spawnGems();
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.scores = [0, 0];
    this.gameOver = false;
    this.paused = false;
    this.currentPlayer = 0;
    this.turnsLeft = 30;
    this.gems.forEach(g => {
      this.scene.remove(g);
      g.traverse((obj) => {
        if (obj.isMesh) {
          obj.geometry?.dispose();
          if (Array.isArray(obj.material)) {
            obj.material.forEach(m => m.dispose());
          } else {
            obj.material?.dispose();
          }
        }
      });
    });
    this.gems = [];
    this.createPlayers();
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      this.gems.forEach(g => {
        g.rotation.y += dt * 2;
        g.position.y = 0.5 + Math.sin(now * 0.003 + g.userData.gx) * 0.1;
      });

      this.players.forEach((p, i) => {
        const targetX = p.userData.gx;
        const targetZ = p.userData.gz;
        p.position.x += (targetX - p.position.x) * 0.15;
        p.position.z += (targetZ - p.position.z) * 0.15;
        const scale = i === this.currentPlayer && !this.gameOver ? 1.1 : 1.0;
        p.scale.setScalar(scale);
      });

      const active = this.players[this.currentPlayer];
      const camOffsetX = this.currentPlayer === 0 ? -4 : 4;
      this.cameraTarget.x = active.userData.gx + camOffsetX;
      this.cameraTarget.z = active.userData.gz + 8;
      this.camera.position.x += (this.cameraTarget.x - this.camera.position.x) * 0.04;
      this.camera.position.z += (this.cameraTarget.z - this.camera.position.z) * 0.04;
      this.camera.position.y += (10 - this.camera.position.y) * 0.04;
      this.camera.lookAt(active.position.x, 0, active.position.z);
    }
    this.renderer.render(this.scene, this.camera);
  },

  movePlayer(dx, dz) {
    const p = this.players[this.currentPlayer];
    const half = Math.floor(this.gridSize / 2);
    const nx = p.userData.gx + dx;
    const nz = p.userData.gz + dz;

    if (nx < -half || nx > half || nz < -half || nz > half) return false;

    const other = this.players[1 - this.currentPlayer];
    if (other.userData.gx === nx && other.userData.gz === nz) {
      const pushX = nx + dx;
      const pushZ = nz + dz;
      if (pushX < -half || pushX > half || pushZ < -half || pushZ > half) {
        this.scores[this.currentPlayer] += 50;
        other.userData.gx = this.currentPlayer === 0 ?
          Math.floor(this.gridSize / 2) : -Math.floor(this.gridSize / 2);
        other.userData.gz = other.userData.gx;
      } else {
        other.userData.gx = pushX;
        other.userData.gz = pushZ;
        this.scores[this.currentPlayer] += 5;
      }
    }

    p.userData.gx = nx;
    p.userData.gz = nz;

    for (let i = this.gems.length - 1; i >= 0; i--) {
      const g = this.gems[i];
      if (g.userData.gx === nx && g.userData.gz === nz) {
        this.scores[this.currentPlayer] += g.userData.value;
        this.scene.remove(g);
        this.gems.splice(i, 1);
      }
    }

    this.endTurn();
    return true;
  },

  endTurn() {
    this.currentPlayer = 1 - this.currentPlayer;
    this.turnsLeft--;
    if (this.turnsLeft <= 0 || this.gems.length === 0) {
      this.gameOver = true;
    }
  },

  getState() {
    return {
      score: Math.max(this.scores[0], this.scores[1]),
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      currentPlayer: this.currentPlayer + 1,
      turnsLeft: this.turnsLeft,
      scores: { player1: this.scores[0], player2: this.scores[1] },
      winner: this.gameOver ? (this.scores[0] > this.scores[1] ? 1 :
        this.scores[1] > this.scores[0] ? 2 : 0) : null,
      players: this.players.map((p, i) => ({
        id: i + 1,
        position: { x: p.userData.gx, z: p.userData.gz },
        isActive: i === this.currentPlayer
      })),
      gems: this.gems.map(g => ({
        position: { x: g.userData.gx, z: g.userData.gz },
        value: g.userData.value
      })),
      gridSize: this.gridSize
    };
  },

  sendInput(action) {
    if (this.gameOver) return false;
    if (action === 'start') { if (!this.running) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused) return false;

    switch (action) {
      case 'up':    return this.movePlayer(0, -1);
      case 'down':  return this.movePlayer(0, 1);
      case 'left':  return this.movePlayer(-1, 0);
      case 'right': return this.movePlayer(1, 0);
      case 'action': return this.movePlayer(0, 0);
      default: return false;
    }
  },

  getMeta() {
    return {
      name: 'Arena Clash',
      description: 'A turn-based 3D arena game for two players. Move on a grid to collect gems and push your opponent. Players alternate turns.',
      controls: {
        up: 'Move active player forward (north)',
        down: 'Move active player backward (south)',
        left: 'Move active player left (west)',
        right: 'Move active player right (east)',
        action: 'Stand ground (skip movement, end turn)'
      },
      objective: 'Collect more gems than your opponent across 30 turns. Push the opponent to the edge for bonus points.',
      scoring: 'Gems: 10-20 pts each. Pushing opponent: 5 pts. Pushing off edge: 50 pts. Highest score wins.',
      tips: [
        'Players alternate turns -- plan two moves ahead.',
        'Pushing the opponent off the edge resets them to their corner and awards 50 points.',
        'Use getState().gems to find the nearest high-value gem.',
        'The action button skips your move -- useful strategically to let the opponent overextend.',
        'Camera follows the active player. Player 1 is red, Player 2 is blue.',
        'The game ends after 30 turns or when all gems are collected.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Multiplayer 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Highest score among players (required) |
| `gameOver` | boolean | Whether game has ended (required) |
| `currentPlayer` | number | Which player's turn it is (1 or 2) |
| `scores` | object | Per-player scores `{ player1, player2 }` |
| `winner` | number/null | Winner ID when game ends (0 for tie) |
| `players` | array | Each player's position, status, active flag |
| `turnsLeft` | number | Remaining turns in the match |

Agent-friendly design: An agent controlling both players can use `currentPlayer` to determine whose turn it is and `players[].position` with `gems[]` to compute optimal moves.

---

## getMeta() Guidelines for Multiplayer 3D

- **controls**: Explain that inputs affect the currently active player. Note turn alternation.
- **objective**: Clearly state the competitive goal and win condition.
- **scoring**: Break down per-player scoring so agents can optimize strategy.
- **tips**: Explain the turn system, how players are visually differentiated, and strategic considerations.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "multiplayer"
libs: ["three"]
```

---

## Verification Checklist

- [ ] `window.ClawmachineGame` is defined with all 6 methods: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] Canvas obtained via `document.getElementById('clawmachine-canvas')` -- not created manually
- [ ] Renderer created with `new THREE.WebGLRenderer({ canvas, antialias: true })`
- [ ] `THREE` used as global (provided by platform via `libs: ["three"]`)
- [ ] No forbidden APIs: no `localStorage`, `fetch(`, `eval(`, `import(`, etc.
- [ ] `getState()` returns `{ score, gameOver, ...additionalState }`
- [ ] `sendInput(action)` handles all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `getMeta()` returns `{ name, description, controls }` at minimum
- [ ] Game loop uses `requestAnimationFrame`
- [ ] Two players are visually distinct (different colors/meshes)
- [ ] Turn-based system works correctly with single `sendInput()` channel
- [ ] Camera transitions smoothly between active players
- [ ] Per-player scores are tracked and exposed in `getState()`
- [ ] Winner is determined correctly on game end
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
