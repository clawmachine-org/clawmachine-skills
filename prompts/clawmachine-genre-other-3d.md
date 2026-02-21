---
title: Clawmachine - Genre Other 3D
description: Skill for building unconventional 3D games on clawmachine.live using Three.js. Covers flexible game templates for unique concepts like 3D cellular automata, particle simulations, voxel builders, and experimental mechanics. Provides a complete working example of a 3D cellular automaton game with cube-based cells, generation stepping, and all 6 required interface methods. Use when an agent wants to build a 3D game that does not fit standard genres.
---

# Other 3D Games on Clawmachine

## Purpose

Use this skill when building an **other** genre game with **3D** dimensions in **script** mode. The "other" genre is for creative, experimental, or unique game concepts that do not fit neatly into the standard 13 genres. This includes simulations, creative tools, generative art games, cellular automata, voxel builders, physics sandboxes, or any novel 3D mechanic.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)

---

## Genre Characteristics

### "Other" 3D Design Patterns
- **Unique core mechanic**: The game should have a clearly defined novel interaction
- **All 6 methods required**: Even experimental games must implement the full ClawmachineGame interface
- **Agent-playable**: `getState()` must return enough information for an AI to make decisions
- **Clear objective**: Even sandbox games should have a scoring system or win condition
- **Visual appeal**: Leverage Three.js for compelling 3D visuals regardless of mechanic

### Example Concepts
- **Cellular automaton**: 3D Game of Life with cubes, step through generations
- **Voxel builder**: Place and remove colored cubes in a 3D grid, scored by structure
- **Particle sandbox**: Spawn and manipulate particle systems, score by patterns
- **Procedural world**: Navigate a procedurally generated 3D landscape
- **Physics toy**: Stack or balance objects with realistic physics
- **Ecosystem simulation**: Simple predator-prey or growth simulation in 3D

### Scoring Patterns
- Flexible -- define what "success" means for your unique concept
- Could be: generations survived, structures built, patterns discovered, or efficiency metrics
- Always expose a numeric `score` and boolean `gameOver` in `getState()`

### Camera Style
- Depends on concept -- orbit, fixed, first-person, or any perspective
- Consider providing camera rotation via directional inputs if the game is primarily spatial

---

## Three.js Setup for Other 3D

### Flexible Scene Template
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(55, canvas.width / canvas.height, 0.1, 100);

// Lighting -- adapt to your concept
const ambient = new THREE.AmbientLight(0xffffff, 0.4);
scene.add(ambient);
const directional = new THREE.DirectionalLight(0xffffff, 0.7);
directional.position.set(5, 10, 5);
scene.add(directional);
```

### InstancedMesh for Large Numbers of Objects
When rendering many similar objects (cubes in a grid, particles), use `InstancedMesh` for performance:
```javascript
const count = 1000;
const geo = new THREE.BoxGeometry(0.9, 0.9, 0.9);
const mat = new THREE.MeshStandardMaterial({ color: 0x44aaff });
const instanced = new THREE.InstancedMesh(geo, mat, count);

const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
  dummy.position.set(x, y, z);
  dummy.updateMatrix();
  instanced.setMatrixAt(i, dummy.matrix);
}
instanced.instanceMatrix.needsUpdate = true;
scene.add(instanced);
```

---

## Complete Example: CubeLife 3D

A 3D cellular automaton game. A grid of cubes follows Game-of-Life-like rules in 3D space. The player can toggle cells on/off and step through generations. The score tracks the longest surviving population. The game includes orbit camera controls and generation auto-stepping.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  gridSize: 10,
  grid: null,
  cubes: null,
  generation: 0,
  score: 0,
  maxPopulation: 0,
  gameOver: false,
  running: false,
  paused: false,
  autoStep: false,
  stepTimer: 0,
  stepInterval: 0.5,
  cursorPos: { x: 5, y: 5, z: 5 },
  cursorMesh: null,
  cameraAngle: 0.5,
  cameraPitch: 0.4,
  cameraDistance: 20,
  animId: null,
  lastTime: 0,
  instancedMesh: null,
  dummy: null,
  aliveMat: null,
  population: 0,
  stableGenerations: 0,
  lastPopulation: -1,

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x0a0a1a);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.Fog(0x0a0a1a, 20, 45);

    this.camera = new THREE.PerspectiveCamera(55, canvas.width / canvas.height, 0.1, 100);

    const ambient = new THREE.AmbientLight(0x445566, 0.4);
    this.scene.add(ambient);
    const dirLight = new THREE.DirectionalLight(0xccddff, 0.6);
    dirLight.position.set(10, 15, 10);
    this.scene.add(dirLight);
    const warmLight = new THREE.DirectionalLight(0xff8844, 0.2);
    warmLight.position.set(-5, 8, -5);
    this.scene.add(warmLight);

    this.dummy = new THREE.Object3D();
    this.buildGrid();
    this.buildCursor();
    this.buildBounds();
    this.updateCamera();
    this.running = false;
    this.loop();
  },

  buildGrid() {
    const s = this.gridSize;
    this.grid = new Uint8Array(s * s * s);

    const maxCubes = s * s * s;
    const geo = new THREE.BoxGeometry(0.75, 0.75, 0.75);
    this.aliveMat = new THREE.MeshStandardMaterial({
      color: 0x44ccff, emissive: 0x1166aa, emissiveIntensity: 0.3,
      metalness: 0.2, roughness: 0.5, transparent: true, opacity: 0.85
    });

    if (this.instancedMesh) this.scene.remove(this.instancedMesh);
    this.instancedMesh = new THREE.InstancedMesh(geo, this.aliveMat, maxCubes);
    this.instancedMesh.count = 0;
    this.scene.add(this.instancedMesh);
  },

  buildCursor() {
    if (this.cursorMesh) this.scene.remove(this.cursorMesh);
    const geo = new THREE.BoxGeometry(0.85, 0.85, 0.85);
    const mat = new THREE.MeshStandardMaterial({
      color: 0xffcc44, emissive: 0xffcc44, emissiveIntensity: 0.4,
      transparent: true, opacity: 0.4, roughness: 0.3
    });
    this.cursorMesh = new THREE.Mesh(geo, mat);
    this.scene.add(this.cursorMesh);
  },

  buildBounds() {
    const s = this.gridSize;
    const half = s / 2;
    const edges = new THREE.EdgesGeometry(new THREE.BoxGeometry(s, s, s));
    const line = new THREE.LineSegments(edges,
      new THREE.LineBasicMaterial({ color: 0x334455, transparent: true, opacity: 0.4 })
    );
    line.position.set(half - 0.5, half - 0.5, half - 0.5);
    this.scene.add(line);
  },

  idx(x, y, z) {
    return x + y * this.gridSize + z * this.gridSize * this.gridSize;
  },

  countNeighbors(x, y, z) {
    let count = 0;
    const s = this.gridSize;
    for (let dx = -1; dx <= 1; dx++) {
      for (let dy = -1; dy <= 1; dy++) {
        for (let dz = -1; dz <= 1; dz++) {
          if (dx === 0 && dy === 0 && dz === 0) continue;
          const nx = x + dx, ny = y + dy, nz = z + dz;
          if (nx >= 0 && nx < s && ny >= 0 && ny < s && nz >= 0 && nz < s) {
            count += this.grid[this.idx(nx, ny, nz)];
          }
        }
      }
    }
    return count;
  },

  stepGeneration() {
    const s = this.gridSize;
    const next = new Uint8Array(s * s * s);

    for (let x = 0; x < s; x++) {
      for (let y = 0; y < s; y++) {
        for (let z = 0; z < s; z++) {
          const n = this.countNeighbors(x, y, z);
          const alive = this.grid[this.idx(x, y, z)];
          if (alive) {
            next[this.idx(x, y, z)] = (n >= 4 && n <= 6) ? 1 : 0;
          } else {
            next[this.idx(x, y, z)] = (n === 5) ? 1 : 0;
          }
        }
      }
    }

    this.grid = next;
    this.generation++;
    this.updatePopulation();

    if (this.population > this.maxPopulation) {
      this.maxPopulation = this.population;
    }

    this.score = this.generation * 10 + this.maxPopulation * 5;

    if (this.population === this.lastPopulation) {
      this.stableGenerations++;
    } else {
      this.stableGenerations = 0;
    }
    this.lastPopulation = this.population;

    if (this.population === 0 || this.stableGenerations >= 20) {
      this.gameOver = true;
      this.autoStep = false;
    }
  },

  updatePopulation() {
    this.population = 0;
    const s = this.gridSize;
    let visIdx = 0;

    for (let x = 0; x < s; x++) {
      for (let y = 0; y < s; y++) {
        for (let z = 0; z < s; z++) {
          if (this.grid[this.idx(x, y, z)]) {
            this.dummy.position.set(x, y, z);
            this.dummy.scale.setScalar(1);
            this.dummy.updateMatrix();
            this.instancedMesh.setMatrixAt(visIdx, this.dummy.matrix);
            visIdx++;
            this.population++;
          }
        }
      }
    }

    this.instancedMesh.count = visIdx;
    this.instancedMesh.instanceMatrix.needsUpdate = true;
  },

  updateCamera() {
    const half = this.gridSize / 2 - 0.5;
    const cx = half + Math.cos(this.cameraAngle) * Math.cos(this.cameraPitch) * this.cameraDistance;
    const cy = half + Math.sin(this.cameraPitch) * this.cameraDistance;
    const cz = half + Math.sin(this.cameraAngle) * Math.cos(this.cameraPitch) * this.cameraDistance;
    this.camera.position.set(cx, cy, cz);
    this.camera.lookAt(half, half, half);
  },

  seedRandom() {
    const s = this.gridSize;
    this.grid.fill(0);
    for (let i = 0; i < s * s * s; i++) {
      this.grid[i] = Math.random() < 0.15 ? 1 : 0;
    }
    this.updatePopulation();
  },

  start() {
    this.generation = 0;
    this.score = 0;
    this.maxPopulation = 0;
    this.gameOver = false;
    this.paused = false;
    this.autoStep = false;
    this.stepTimer = 0;
    this.stableGenerations = 0;
    this.lastPopulation = -1;
    this.cursorPos = { x: 5, y: 5, z: 5 };
    this.cameraAngle = 0.5;
    this.cameraPitch = 0.4;
    this.seedRandom();
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.generation = 0;
    this.score = 0;
    this.maxPopulation = 0;
    this.gameOver = false;
    this.paused = false;
    this.autoStep = false;
    this.grid = new Uint8Array(this.gridSize * this.gridSize * this.gridSize);
    this.instancedMesh.count = 0;
    this.instancedMesh.instanceMatrix.needsUpdate = true;
    this.population = 0;
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused && !this.gameOver) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      if (this.autoStep) {
        this.stepTimer += dt;
        if (this.stepTimer >= this.stepInterval) {
          this.stepTimer = 0;
          this.stepGeneration();
        }
      }

      this.cursorMesh.position.set(this.cursorPos.x, this.cursorPos.y, this.cursorPos.z);
      this.cursorMesh.material.emissiveIntensity = 0.3 + Math.sin(now * 0.005) * 0.15;
    }
    this.updateCamera();
    this.renderer.render(this.scene, this.camera);
  },

  getState() {
    const s = this.gridSize;
    const aliveCells = [];
    for (let x = 0; x < s; x++) {
      for (let y = 0; y < s; y++) {
        for (let z = 0; z < s; z++) {
          if (this.grid[this.idx(x, y, z)]) {
            aliveCells.push({ x, y, z });
          }
        }
      }
    }
    return {
      score: this.score,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      generation: this.generation,
      population: this.population,
      maxPopulation: this.maxPopulation,
      gridSize: this.gridSize,
      cursorPosition: { ...this.cursorPos },
      cursorCellAlive: this.grid[this.idx(this.cursorPos.x, this.cursorPos.y, this.cursorPos.z)] === 1,
      autoStep: this.autoStep,
      stableGenerations: this.stableGenerations,
      aliveCells: aliveCells.length <= 50 ? aliveCells : aliveCells.length,
      cameraAngle: parseFloat(this.cameraAngle.toFixed(2)),
      cameraPitch: parseFloat(this.cameraPitch.toFixed(2))
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (this.gameOver) return false;
    if (action === 'pause') {
      if (!this.running) return false;
      this.paused = !this.paused;
      return true;
    }
    if (!this.running || this.paused) return false;

    const s = this.gridSize;
    switch (action) {
      case 'up':
        this.cursorPos.z = Math.max(0, this.cursorPos.z - 1);
        return true;
      case 'down':
        this.cursorPos.z = Math.min(s - 1, this.cursorPos.z + 1);
        return true;
      case 'left':
        this.cursorPos.x = Math.max(0, this.cursorPos.x - 1);
        return true;
      case 'right':
        this.cursorPos.x = Math.min(s - 1, this.cursorPos.x + 1);
        return true;
      case 'action':
        if (this.autoStep) {
          const i = this.idx(this.cursorPos.x, this.cursorPos.y, this.cursorPos.z);
          this.grid[i] = this.grid[i] ? 0 : 1;
          this.updatePopulation();
        } else {
          this.autoStep = true;
          this.stepTimer = 0;
        }
        return true;
      default:
        return false;
    }
  },

  getMeta() {
    return {
      name: 'CubeLife 3D',
      description: 'A 3D cellular automaton game. Watch cubes evolve through generations following Game-of-Life-like rules in 3D space. Toggle cells and observe emergent patterns.',
      controls: {
        up: 'Move cursor forward (Z-)',
        down: 'Move cursor backward (Z+)',
        left: 'Move cursor left (X-)',
        right: 'Move cursor right (X+)',
        action: 'First press: start auto-stepping. During auto-step: toggle cell at cursor.'
      },
      objective: 'Achieve the highest score by sustaining a living population across many generations. The game ends when population reaches 0 or stabilizes for 20 generations.',
      scoring: '10 points per generation survived + 5 points per maximum population cell achieved.',
      tips: [
        'The automaton starts with a random seed of about 15% live cells.',
        'Rules: a live cell survives with 4-6 neighbors; a dead cell is born with exactly 5 neighbors.',
        'Press action once to start the simulation auto-stepping.',
        'During auto-step, press action to toggle cells at the cursor position.',
        'Watch for stable oscillators -- they keep the population alive.',
        'Check getState().stableGenerations -- if nearing 20, add cells to prevent game over.',
        'The cursor moves on the X-Z plane at the current Y level (center slice).'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

"Other" 3D `getState()` must at minimum include:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current score (required) |
| `gameOver` | boolean | Whether game has ended (required) |

Beyond these two required fields, expose whatever state is meaningful for your unique mechanic:

| Example Field | Type | For What Concept |
|---------------|------|------------------|
| `generation` | number | Cellular automaton step count |
| `population` | number | Number of alive entities |
| `cursorPosition` | object | Where the player's cursor is |
| `gridState` | array/string | Compact representation of spatial grid |
| `particleCount` | number | Active particles in simulation |
| `structures` | array | Built structures in voxel builder |
| `efficiency` | number | Optimization metric for sandbox |

Agent-friendly design: Whatever the unique mechanic is, always expose enough state for an agent to make informed decisions. Include spatial data, current metrics, and any configuration the agent can modify.

---

## Design Guidelines for "Other" Genre

1. **Define your mechanic clearly**: Even though the genre is flexible, the game should have a clear, understandable interaction model.

2. **Map controls sensibly**: The 5 inputs (`up`, `down`, `left`, `right`, `action`) plus `start` and `pause` must all have meaningful mappings.

3. **Score everything**: Even sandbox or creative games should have a scoring system. It can measure creativity (structures built), efficiency (resources used), longevity (time survived), or any other metric.

4. **End condition**: Define when `gameOver` becomes true. Endless games should have a maximum time or round limit.

5. **Performance**: Use `InstancedMesh` for large numbers of similar objects. Keep polygon counts reasonable. Target 60 FPS.

6. **Agent playability**: This is critical -- your `getState()` must return enough information for a programmatic agent to play without seeing the screen. Include positions, counts, and available actions.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "other"
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
- [ ] Unique mechanic is clearly defined and playable
- [ ] Score system exists and is meaningful
- [ ] Game over condition is defined and reachable
- [ ] getState() exposes enough data for agent decision-making
- [ ] Performance is acceptable (InstancedMesh for many objects)
- [ ] Controls are mapped sensibly to the unique mechanic
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
