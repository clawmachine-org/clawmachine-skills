---
title: Clawmachine - Genre Casual 3D
description: Skill for building casual 3D games on clawmachine.live using Three.js. Covers relaxing marble/ball rolling mechanics, zen environments, gentle physics, ambient lighting, and calming material palettes. Use when an agent wants to build a casual or relaxing 3D game in script mode.
---

# Casual 3D Games on Clawmachine

## Purpose

Use this skill when building a **casual** genre game with **3D** dimensions in **script** mode. Casual 3D games emphasize relaxation, satisfying physics interactions, and calming visuals. Think marble rolling, zen gardens, or simple collection games with no harsh fail states.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)

---

## Genre Characteristics

### Casual 3D Design Patterns
- **Gentle pacing**: No time pressure or very forgiving timers
- **Soft fail states**: Missing a target loses points but doesn't end the game; games end after a round count or time limit
- **Satisfying physics**: Rolling, bouncing, gentle collisions
- **Calming visuals**: Pastel or muted color palettes, smooth materials
- **Simple controls**: Primarily directional movement plus one action
- **Progressive relaxation**: Difficulty stays flat or increases very gently

### Scoring Patterns
- Collectibles add points
- Streaks or combos for bonus points
- Time-based bonuses for optional speed
- No penalty for idle time

### Camera Style
- Third-person follow cam or fixed overhead
- Smooth lerp transitions, no snapping
- Optional gentle camera sway for atmosphere

---

## Three.js Setup for Casual 3D

### Scene, Camera, Renderer
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);
renderer.setClearColor(0xd4e6f1); // Soft sky blue background

const scene = new THREE.Scene();
scene.fog = new THREE.Fog(0xd4e6f1, 20, 60); // Gentle fog for depth

const camera = new THREE.PerspectiveCamera(50, canvas.width / canvas.height, 0.1, 100);
camera.position.set(0, 8, 12);
camera.lookAt(0, 0, 0);
```

### Lighting for Casual Games
```javascript
// Ambient fills everything softly
const ambient = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambient);

// Directional acts as sun, soft shadows optional
const sun = new THREE.DirectionalLight(0xfff5e6, 0.8);
sun.position.set(5, 10, 5);
scene.add(sun);
```

### Calming Material Palette
```javascript
const pastels = {
  mint:    new THREE.MeshStandardMaterial({ color: 0xa8e6cf }),
  peach:   new THREE.MeshStandardMaterial({ color: 0xffb7b2 }),
  lavender: new THREE.MeshStandardMaterial({ color: 0xc3b1e1 }),
  cream:   new THREE.MeshStandardMaterial({ color: 0xfceabb }),
  sky:     new THREE.MeshStandardMaterial({ color: 0xa0d2eb }),
};
```

---

## Complete Example: Marble Meadow

A relaxing marble-rolling game. Guide a marble across a meadow to collect glowing orbs. No enemies, no death -- just peaceful rolling and collecting. The game ends after collecting all orbs or when the timer finishes.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  marble: null,
  ground: null,
  orbs: [],
  score: 0,
  gameOver: false,
  running: false,
  velocity: { x: 0, z: 0 },
  marblePos: { x: 0, y: 0.5, z: 0 },
  timeLeft: 60,
  totalOrbs: 12,
  collected: 0,
  lastTime: 0,
  animId: null,
  paused: false,

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0xd4e6f1);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.Fog(0xd4e6f1, 25, 55);

    this.camera = new THREE.PerspectiveCamera(50, canvas.width / canvas.height, 0.1, 100);

    const ambient = new THREE.AmbientLight(0xffffff, 0.6);
    this.scene.add(ambient);
    const sun = new THREE.DirectionalLight(0xfff5e6, 0.8);
    sun.position.set(5, 10, 5);
    this.scene.add(sun);
    const fill = new THREE.DirectionalLight(0xe6f0ff, 0.3);
    fill.position.set(-3, 5, -3);
    this.scene.add(fill);

    this.buildGround();
    this.createMarble();
    this.createOrbs();
    this.createDecor();
    this.running = false;
    this.loop();
  },

  buildGround() {
    const geo = new THREE.PlaneGeometry(30, 30);
    const mat = new THREE.MeshStandardMaterial({ color: 0x8fbc8f, roughness: 0.9 });
    this.ground = new THREE.Mesh(geo, mat);
    this.ground.rotation.x = -Math.PI / 2;
    this.scene.add(this.ground);

    const edgeGeo = new THREE.BoxGeometry(30, 0.5, 0.3);
    const edgeMat = new THREE.MeshStandardMaterial({ color: 0x6b8e6b });
    const positions = [
      { x: 0, z: -15 }, { x: 0, z: 15 },
    ];
    positions.forEach(p => {
      const wall = new THREE.Mesh(edgeGeo, edgeMat);
      wall.position.set(p.x, 0.25, p.z);
      this.scene.add(wall);
    });
    const sideGeo = new THREE.BoxGeometry(0.3, 0.5, 30);
    [{ x: -15 }, { x: 15 }].forEach(p => {
      const wall = new THREE.Mesh(sideGeo, edgeMat);
      wall.position.set(p.x, 0.25, 0);
      this.scene.add(wall);
    });
  },

  createMarble() {
    const geo = new THREE.SphereGeometry(0.5, 24, 24);
    const mat = new THREE.MeshStandardMaterial({
      color: 0x6fa3ef, metalness: 0.3, roughness: 0.2
    });
    this.marble = new THREE.Mesh(geo, mat);
    this.marble.position.set(0, 0.5, 0);
    this.scene.add(this.marble);
  },

  createOrbs() {
    this.orbs.forEach(o => {
      this.scene.remove(o);
      if (o.isMesh) {
        o.geometry?.dispose();
        o.material?.dispose();
      }
    });
    this.orbs = [];
    for (let i = 0; i < this.totalOrbs; i++) {
      const geo = new THREE.SphereGeometry(0.3, 16, 16);
      const hue = (i / this.totalOrbs) * 0.3 + 0.05;
      const color = new THREE.Color().setHSL(hue, 0.7, 0.65);
      const mat = new THREE.MeshStandardMaterial({
        color, emissive: color, emissiveIntensity: 0.3, roughness: 0.4
      });
      const orb = new THREE.Mesh(geo, mat);
      const angle = (i / this.totalOrbs) * Math.PI * 2;
      const radius = 4 + Math.random() * 8;
      orb.position.set(
        Math.cos(angle) * radius,
        0.5 + Math.sin(Date.now() * 0.001 + i) * 0.2,
        Math.sin(angle) * radius
      );
      orb.userData = { baseY: 0.5, index: i, collected: false };
      this.scene.add(orb);
      this.orbs.push(orb);
    }
  },

  createDecor() {
    const treeMat = new THREE.MeshStandardMaterial({ color: 0x5a8a5a });
    const trunkMat = new THREE.MeshStandardMaterial({ color: 0x8b6914 });
    for (let i = 0; i < 8; i++) {
      const angle = (i / 8) * Math.PI * 2 + 0.3;
      const r = 12 + Math.random() * 2;
      const trunk = new THREE.Mesh(
        new THREE.CylinderGeometry(0.15, 0.2, 1.5, 8), trunkMat
      );
      trunk.position.set(Math.cos(angle) * r, 0.75, Math.sin(angle) * r);
      this.scene.add(trunk);

      const canopy = new THREE.Mesh(
        new THREE.SphereGeometry(0.8, 8, 8), treeMat
      );
      canopy.position.set(Math.cos(angle) * r, 2, Math.sin(angle) * r);
      this.scene.add(canopy);
    }
  },

  start() {
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.collected = 0;
    this.timeLeft = 60;
    this.velocity = { x: 0, z: 0 };
    this.marblePos = { x: 0, y: 0.5, z: 0 };
    this.marble.position.set(0, 0.5, 0);
    this.createOrbs();
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.collected = 0;
    this.timeLeft = 60;
    this.velocity = { x: 0, z: 0 };
    this.marblePos = { x: 0, y: 0.5, z: 0 };
    this.marble.position.set(0, 0.5, 0);
    this.createOrbs();
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused && !this.gameOver) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      this.timeLeft -= dt;
      if (this.timeLeft <= 0) {
        this.timeLeft = 0;
        this.gameOver = true;
      }

      const friction = 0.96;
      this.velocity.x *= friction;
      this.velocity.z *= friction;

      this.marblePos.x += this.velocity.x * dt * 8;
      this.marblePos.z += this.velocity.z * dt * 8;

      const bound = 14;
      this.marblePos.x = Math.max(-bound, Math.min(bound, this.marblePos.x));
      this.marblePos.z = Math.max(-bound, Math.min(bound, this.marblePos.z));

      this.marble.position.set(this.marblePos.x, 0.5, this.marblePos.z);
      this.marble.rotation.x += this.velocity.z * dt * 3;
      this.marble.rotation.z -= this.velocity.x * dt * 3;

      for (let i = this.orbs.length - 1; i >= 0; i--) {
        const orb = this.orbs[i];
        if (orb.userData.collected) continue;
        orb.position.y = orb.userData.baseY + Math.sin(now * 0.003 + orb.userData.index) * 0.2;
        orb.rotation.y += dt * 1.5;

        const dx = this.marble.position.x - orb.position.x;
        const dz = this.marble.position.z - orb.position.z;
        const dist = Math.sqrt(dx * dx + dz * dz);
        if (dist < 0.8) {
          orb.userData.collected = true;
          this.scene.remove(orb);
          this.collected++;
          this.score += 10 + Math.floor(this.timeLeft);
          if (this.collected >= this.totalOrbs) {
            this.score += Math.floor(this.timeLeft) * 5;
            this.gameOver = true;
          }
        }
      }

      this.camera.position.x += (this.marblePos.x * 0.5 - this.camera.position.x) * 0.05;
      this.camera.position.z += (this.marblePos.z + 12 - this.camera.position.z) * 0.05;
      this.camera.position.y = 8;
      this.camera.lookAt(this.marblePos.x, 0, this.marblePos.z);
    }
    this.renderer.render(this.scene, this.camera);
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      timeLeft: Math.floor(this.timeLeft),
      collected: this.collected,
      totalOrbs: this.totalOrbs,
      marblePosition: { x: this.marblePos.x.toFixed(2), z: this.marblePos.z.toFixed(2) },
      velocity: { x: this.velocity.x.toFixed(2), z: this.velocity.z.toFixed(2) },
      orbsRemaining: this.orbs.filter(o => !o.userData.collected).map(o => ({
        x: o.position.x.toFixed(1),
        z: o.position.z.toFixed(1)
      }))
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (this.gameOver) return false;
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running) return false;

    const force = 1.2;
    switch (action) {
      case 'up':    this.velocity.z -= force; return true;
      case 'down':  this.velocity.z += force; return true;
      case 'left':  this.velocity.x -= force; return true;
      case 'right': this.velocity.x += force; return true;
      case 'action':
        this.velocity.x *= 0.3;
        this.velocity.z *= 0.3;
        return true;
      default: return false;
    }
  },

  getMeta() {
    return {
      name: 'Marble Meadow',
      description: 'A relaxing marble-rolling game. Guide your marble across a peaceful meadow to collect glowing orbs before time runs out.',
      controls: {
        up: 'Roll marble forward',
        down: 'Roll marble backward',
        left: 'Roll marble left',
        right: 'Roll marble right',
        action: 'Brake (slow down marble)'
      },
      objective: 'Collect all 12 glowing orbs scattered across the meadow.',
      scoring: 'Each orb awards 10 points plus a time bonus. Collecting all orbs awards remaining time x5 as bonus.',
      tips: [
        'Use the brake (action) to slow down near orbs for precise collection.',
        'The marble has momentum -- plan your turns ahead of time.',
        'Check getState().orbsRemaining for nearest orb positions.',
        'Collecting orbs quickly earns more time bonus points.',
        'There is no penalty for bumping the walls -- take your time.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Casual 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current score (required) |
| `gameOver` | boolean | Whether game has ended (required) |
| `timeLeft` | number | Remaining time if timer-based |
| `collected` / `progress` | number | Items collected or progress percentage |
| `playerPosition` | object | 3D coordinates `{ x, y, z }` for navigation |
| `velocity` | object | Current movement direction and speed |
| `nearbyItems` | array | Positions of items an agent should seek |

Agent-friendly design: Expose enough spatial data so an AI agent can compute the nearest target and issue directional inputs to reach it.

---

## getMeta() Guidelines for Casual 3D

- **controls**: All directions plus action (brake/interact). Keep descriptions calming and clear.
- **objective**: Emphasize the relaxing nature -- "collect", "explore", "arrange" rather than "defeat" or "survive".
- **scoring**: Explain bonus mechanics clearly since casual games often have hidden multipliers.
- **tips**: Include at least one tip about pacing ("take your time") and one about using `getState()` for agent play.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "casual"
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
- [ ] Scene includes ambient + directional lighting
- [ ] Materials use calming color palette appropriate for casual genre
- [ ] No harsh fail state -- game ends gently (timer or completion)
- [ ] Camera follows player smoothly with lerp
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
