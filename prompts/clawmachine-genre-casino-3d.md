---
title: Clawmachine - Genre Casino 3D
description: Skill for building casino 3D games on clawmachine.live using Three.js. Covers 3D dice rolling with physics-based rotation animation, casino table surfaces with felt materials, camera zoom on action, bet systems, and probability-based outcomes. Use when an agent wants to build a casino or gambling-style 3D game in script mode.
---

# Casino 3D Games on Clawmachine

## Purpose

Use this skill when building a **casino** genre game with **3D** dimensions in **script** mode. Casino 3D games feature realistic dice, cards, or roulette rendered in 3D with physics-based animations, felt-covered table surfaces, atmospheric lighting, and camera movements that zoom in on the action during reveals.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)

---

## Genre Characteristics

### Casino 3D Design Patterns
- **Physics-based animation**: Dice tumbling, wheels spinning, cards flipping with convincing motion
- **Table environment**: Felt-covered surfaces, wooden rails, atmospheric casino lighting
- **Camera drama**: Zoom into dice/cards during reveal, pull back to show results
- **Bet system**: Virtual chips or credits with wager selection
- **Probability-based outcomes**: Fair randomization, payout tables
- **Round-based flow**: Bet phase -> action phase -> reveal phase -> payout phase

### Scoring Patterns
- Virtual chip/credit balance starting at a set amount
- Bet multipliers based on risk (higher risk = higher payout)
- Running total across rounds
- Game over when bankrupt or after reaching a target

### Camera Style
- Overview of the table at rest
- Smooth zoom to dice/action during roll
- Slow pan during reveal
- Return to overview for next bet

---

## Three.js Setup for Casino 3D

### Scene with Casino Atmosphere
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);
renderer.setClearColor(0x0a0a14);

const scene = new THREE.Scene();
scene.fog = new THREE.Fog(0x0a0a14, 15, 40);
const camera = new THREE.PerspectiveCamera(45, canvas.width / canvas.height, 0.1, 50);
```

### Casino Lighting
```javascript
const ambient = new THREE.AmbientLight(0xfff5e0, 0.3);
scene.add(ambient);

// Overhead table lamp
const tableLight = new THREE.SpotLight(0xffeedd, 1.2, 20, Math.PI / 4, 0.5);
tableLight.position.set(0, 8, 0);
tableLight.target.position.set(0, 0, 0);
scene.add(tableLight);
scene.add(tableLight.target);

// Rim accents
const accent = new THREE.PointLight(0xffaa44, 0.3, 15);
accent.position.set(5, 3, 5);
scene.add(accent);
```

### Casino Materials
```javascript
const feltMat = new THREE.MeshStandardMaterial({
  color: 0x1a6b3c, roughness: 0.95, metalness: 0.0
});
const woodMat = new THREE.MeshStandardMaterial({
  color: 0x5c3a1e, roughness: 0.7, metalness: 0.1
});
const diceMat = new THREE.MeshStandardMaterial({
  color: 0xf0f0f0, roughness: 0.3, metalness: 0.05
});
const chipMat = new THREE.MeshStandardMaterial({
  color: 0xcc2222, roughness: 0.4, metalness: 0.3
});
```

---

## Complete Example: Lucky Dice

A 3D dice-rolling casino game. Players place bets and roll two dice. Match the target number to win big. Features physics-based tumbling animation, a felt-covered table, and camera zoom on the roll.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  dice: [],
  table: null,
  chips: 100,
  currentBet: 10,
  targetNumber: 7,
  diceValues: [0, 0],
  rolling: false,
  rollTime: 0,
  rollDuration: 2.0,
  gameOver: false,
  running: false,
  paused: false,
  phase: 'bet',
  lastResult: '',
  animId: null,
  lastTime: 0,
  camBase: { x: 0, y: 7, z: 8 },
  camZoom: { x: 0, y: 3.5, z: 3 },
  roundsPlayed: 0,
  diceRotTargets: [],

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x0a0a14);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.Fog(0x0a0a14, 15, 35);
    this.camera = new THREE.PerspectiveCamera(45, canvas.width / canvas.height, 0.1, 50);
    this.camera.position.set(0, 7, 8);
    this.camera.lookAt(0, 0, 0);

    const ambient = new THREE.AmbientLight(0xfff5e0, 0.3);
    this.scene.add(ambient);
    const tableLight = new THREE.SpotLight(0xffeedd, 1.5, 20, Math.PI / 3.5, 0.6);
    tableLight.position.set(0, 8, 0);
    this.scene.add(tableLight);
    const rimLight = new THREE.PointLight(0xffaa44, 0.25, 15);
    rimLight.position.set(-4, 3, 4);
    this.scene.add(rimLight);

    this.buildTable();
    this.createDice();
    this.createChipStacks();
    this.running = false;
    this.loop();
  },

  buildTable() {
    const feltGeo = new THREE.CylinderGeometry(5, 5, 0.2, 32);
    const feltMat = new THREE.MeshStandardMaterial({ color: 0x1a6b3c, roughness: 0.95 });
    this.table = new THREE.Mesh(feltGeo, feltMat);
    this.table.position.y = -0.1;
    this.scene.add(this.table);

    const rimGeo = new THREE.TorusGeometry(5, 0.25, 8, 48);
    const woodMat = new THREE.MeshStandardMaterial({ color: 0x5c3a1e, roughness: 0.6 });
    const rim = new THREE.Mesh(rimGeo, woodMat);
    rim.rotation.x = Math.PI / 2;
    rim.position.y = 0.05;
    this.scene.add(rim);
  },

  createDice() {
    this.dice.forEach(d => this.scene.remove(d));
    this.dice = [];

    for (let i = 0; i < 2; i++) {
      const group = new THREE.Group();
      const bodyGeo = new THREE.BoxGeometry(0.7, 0.7, 0.7);
      const bodyMat = new THREE.MeshStandardMaterial({
        color: 0xf5f5f0, roughness: 0.3, metalness: 0.05
      });
      const body = new THREE.Mesh(bodyGeo, bodyMat);
      group.add(body);

      const dotMat = new THREE.MeshStandardMaterial({ color: 0x111111 });
      const dotGeo = new THREE.SphereGeometry(0.06, 8, 8);
      const faces = [
        { dots: [[0, 0]], normal: [0, 0, 1] },
        { dots: [[-0.15, 0.15], [0.15, -0.15]], normal: [1, 0, 0] },
        { dots: [[-0.15, 0.15], [0, 0], [0.15, -0.15]], normal: [0, 1, 0] },
        { dots: [[-0.15, 0.15], [0.15, 0.15], [-0.15, -0.15], [0.15, -0.15]], normal: [0, -1, 0] },
        { dots: [[-0.15, 0.15], [0.15, 0.15], [0, 0], [-0.15, -0.15], [0.15, -0.15]], normal: [-1, 0, 0] },
        { dots: [[-0.15, 0.15], [0.15, 0.15], [-0.15, 0], [0.15, 0], [-0.15, -0.15], [0.15, -0.15]], normal: [0, 0, -1] }
      ];
      faces.forEach(face => {
        face.dots.forEach(pos => {
          const dot = new THREE.Mesh(dotGeo, dotMat);
          const n = face.normal;
          if (n[2] !== 0) { dot.position.set(pos[0], pos[1], n[2] * 0.36); }
          else if (n[0] !== 0) { dot.position.set(n[0] * 0.36, pos[1], pos[0]); }
          else { dot.position.set(pos[0], n[1] * 0.36, pos[1]); }
          group.add(dot);
        });
      });

      group.position.set(i === 0 ? -0.8 : 0.8, 0.5, 0);
      group.userData = { value: 1, baseX: i === 0 ? -0.8 : 0.8 };
      this.scene.add(group);
      this.dice.push(group);
    }
  },

  createChipStacks() {
    const colors = [0xcc2222, 0x2266cc, 0x22aa44, 0xddaa22];
    for (let s = 0; s < 4; s++) {
      const mat = new THREE.MeshStandardMaterial({
        color: colors[s], roughness: 0.4, metalness: 0.3
      });
      for (let c = 0; c < 3; c++) {
        const chip = new THREE.Mesh(new THREE.CylinderGeometry(0.25, 0.25, 0.08, 16), mat);
        chip.position.set(-3.5 + s * 0.7, 0.04 + c * 0.09, 3);
        this.scene.add(chip);
      }
    }
  },

  faceRotation(value) {
    const rots = [
      { x: 0, z: 0 },
      { x: 0, z: -Math.PI / 2 },
      { x: -Math.PI / 2, z: 0 },
      { x: Math.PI / 2, z: 0 },
      { x: 0, z: Math.PI / 2 },
      { x: Math.PI, z: 0 }
    ];
    return rots[value - 1] || rots[0];
  },

  rollDice() {
    if (this.rolling || this.phase !== 'bet') return;
    if (this.currentBet > this.chips) return;

    this.phase = 'rolling';
    this.rolling = true;
    this.rollTime = 0;

    this.diceValues = [
      Math.floor(Math.random() * 6) + 1,
      Math.floor(Math.random() * 6) + 1
    ];

    this.diceRotTargets = this.diceValues.map(v => {
      const fr = this.faceRotation(v);
      return {
        x: fr.x + Math.PI * (4 + Math.floor(Math.random() * 4)),
        z: fr.z + Math.PI * (4 + Math.floor(Math.random() * 4)),
        y: Math.PI * (2 + Math.floor(Math.random() * 6))
      };
    });

    this.dice.forEach((d, i) => {
      d.userData.startRot = { x: d.rotation.x, y: d.rotation.y, z: d.rotation.z };
      d.userData.startPos = d.position.y;
    });
  },

  easeOut(t) {
    return 1 - Math.pow(1 - t, 3);
  },

  start() {
    this.chips = 100;
    this.currentBet = 10;
    this.targetNumber = 7;
    this.gameOver = false;
    this.paused = false;
    this.phase = 'bet';
    this.rolling = false;
    this.lastResult = '';
    this.roundsPlayed = 0;
    this.diceValues = [0, 0];
    this.dice.forEach((d, i) => {
      d.rotation.set(0, 0, 0);
      d.position.y = 0.5;
    });
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.chips = 100;
    this.currentBet = 10;
    this.targetNumber = 7;
    this.gameOver = false;
    this.paused = false;
    this.phase = 'bet';
    this.rolling = false;
    this.lastResult = '';
    this.roundsPlayed = 0;
    this.diceValues = [0, 0];
    this.dice.forEach(d => {
      d.rotation.set(0, 0, 0);
      d.position.y = 0.5;
    });
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      if (this.rolling) {
        this.rollTime += dt;
        const t = Math.min(this.rollTime / this.rollDuration, 1.0);
        const eased = this.easeOut(t);

        this.dice.forEach((d, i) => {
          const sr = d.userData.startRot;
          const tr = this.diceRotTargets[i];
          d.rotation.x = sr.x + (tr.x - sr.x) * eased;
          d.rotation.y = sr.y + (tr.y - sr.y) * eased;
          d.rotation.z = sr.z + (tr.z - sr.z) * eased;

          const bounce = Math.sin(t * Math.PI * 3) * (1 - t) * 1.5;
          d.position.y = 0.5 + bounce;
        });

        const camT = Math.min(t * 1.5, 1.0);
        this.camera.position.x += (this.camZoom.x - this.camera.position.x) * camT * 0.08;
        this.camera.position.y += (this.camZoom.y - this.camera.position.y) * camT * 0.08;
        this.camera.position.z += (this.camZoom.z - this.camera.position.z) * camT * 0.08;
        this.camera.lookAt(0, 0.3, 0);

        if (t >= 1.0) {
          this.rolling = false;
          this.resolveBet();
        }
      } else {
        this.camera.position.x += (this.camBase.x - this.camera.position.x) * 0.03;
        this.camera.position.y += (this.camBase.y - this.camera.position.y) * 0.03;
        this.camera.position.z += (this.camBase.z - this.camera.position.z) * 0.03;
        this.camera.lookAt(0, 0, 0);
      }
    }
    this.renderer.render(this.scene, this.camera);
  },

  resolveBet() {
    const total = this.diceValues[0] + this.diceValues[1];
    this.roundsPlayed++;

    if (total === this.targetNumber) {
      const payout = this.currentBet * 4;
      this.chips += payout;
      this.lastResult = 'WIN +' + payout;
    } else if (Math.abs(total - this.targetNumber) === 1) {
      const payout = this.currentBet;
      this.chips += payout;
      this.lastResult = 'CLOSE +' + payout;
    } else {
      this.chips -= this.currentBet;
      this.lastResult = 'MISS -' + this.currentBet;
    }

    if (this.chips <= 0) {
      this.chips = 0;
      this.gameOver = true;
      this.lastResult += ' BANKRUPT';
    } else if (this.chips >= 500) {
      this.gameOver = true;
      this.lastResult += ' JACKPOT!';
    }

    this.phase = 'bet';
    if (this.currentBet > this.chips) {
      this.currentBet = Math.max(5, Math.min(this.currentBet, this.chips));
    }
  },

  getState() {
    return {
      score: this.chips,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      phase: this.phase,
      chips: this.chips,
      currentBet: this.currentBet,
      targetNumber: this.targetNumber,
      diceValues: this.diceValues,
      diceTotal: this.diceValues[0] + this.diceValues[1],
      rolling: this.rolling,
      lastResult: this.lastResult,
      roundsPlayed: this.roundsPlayed,
      betOptions: [5, 10, 25, 50].filter(b => b <= this.chips),
      targetOptions: [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
    };
  },

  sendInput(action) {
    if (this.gameOver) return false;
    if (action === 'start') { if (!this.running) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused || this.rolling) return false;

    switch (action) {
      case 'up':
        this.currentBet = Math.min(this.currentBet + 5, this.chips);
        return true;
      case 'down':
        this.currentBet = Math.max(5, this.currentBet - 5);
        return true;
      case 'left':
        this.targetNumber = Math.max(2, this.targetNumber - 1);
        return true;
      case 'right':
        this.targetNumber = Math.min(12, this.targetNumber + 1);
        return true;
      case 'action':
        this.rollDice();
        return true;
      default:
        return false;
    }
  },

  getMeta() {
    return {
      name: 'Lucky Dice',
      description: 'A 3D casino dice game. Set your target number and bet amount, then roll two dice. Match the target to win big, or land close for a small payout.',
      controls: {
        up: 'Increase bet by 5 chips',
        down: 'Decrease bet by 5 chips',
        left: 'Decrease target number',
        right: 'Increase target number',
        action: 'Roll the dice'
      },
      objective: 'Reach 500 chips to hit the jackpot. Avoid going bankrupt (0 chips).',
      scoring: 'Exact match: 4x bet. Off by 1: 1x bet (break even). Miss: lose bet. Start with 100 chips.',
      tips: [
        'Target 7 has the highest probability of rolling (6/36 combos).',
        'Targets 2 and 12 are hardest (1/36 each) but pay the same 4x.',
        'Manage your bankroll -- smaller bets last longer.',
        'Use getState().betOptions to see valid bet amounts.',
        'The "close" payout (off by 1) helps recover from bad streaks.',
        'Wait for the roll animation to finish before the next input.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Casino 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current chip balance (required) |
| `gameOver` | boolean | Bankrupt or jackpot reached (required) |
| `phase` | string | Current phase: 'bet', 'rolling', 'result' |
| `chips` | number | Explicit chip count |
| `currentBet` | number | Current wager amount |
| `targetNumber` | number | What the player is betting on |
| `diceValues` | array | Current dice face values `[die1, die2]` |
| `rolling` | boolean | Whether animation is in progress |
| `lastResult` | string | Human-readable last outcome |
| `betOptions` | array | Valid bet amounts given current balance |

Agent-friendly design: Expose `betOptions` and `targetOptions` so agents can make informed decisions. Include `phase` so agents know when to bet vs. when to roll.

---

## getMeta() Guidelines for Casino 3D

- **controls**: Map directions to bet/target adjustment, action to the main gamble action (roll/spin/deal).
- **objective**: Define win condition (target balance) and lose condition (bankrupt).
- **scoring**: Explain payout tables clearly with multipliers.
- **tips**: Include probability hints and bankroll management advice.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "casino"
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
- [ ] Casino table has felt material with appropriate roughness
- [ ] Dice/card/wheel has physics-based animation (rotation, bouncing)
- [ ] Camera zooms in during action and returns to overview
- [ ] Bet system with wager adjustment before each round
- [ ] Probability-based fair outcomes using `Math.random()`
- [ ] Atmospheric casino lighting (warm spotlights, dim ambient)
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
