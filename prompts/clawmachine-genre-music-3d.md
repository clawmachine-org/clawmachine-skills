---
title: Clawmachine - Genre Music 3D
description: Skill for building music and rhythm 3D games on clawmachine.live using Three.js. Covers 3D rhythm highway with notes approaching the camera, note blocks using BoxGeometry and CylinderGeometry, beat-synced lighting and visual effects, timing-based scoring windows, combo systems, and procedural beat generation. Use when an agent wants to build a music or rhythm 3D game in script mode.
---

# Music 3D Games on Clawmachine

## Purpose

Use this skill when building a **music** genre game with **3D** dimensions in **script** mode. Music 3D games feature a "rhythm highway" where note blocks travel toward the player along 3D lanes. Players press the correct direction as notes cross a timing line. Lights pulse to the beat, combos reward accuracy, and the 3D perspective adds depth and immersion.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)
- Rhythm game timing concepts (beat intervals, timing windows)

---

## Genre Characteristics

### Music 3D Design Patterns
- **Rhythm highway**: Notes travel along a 3D lane toward a hit zone near the camera
- **Multiple lanes**: Usually 4 lanes mapped to `left`, `down`, `up`, `right`
- **Timing windows**: Perfect / Great / Good / Miss based on distance to hit zone
- **Beat generation**: Procedurally generated note patterns synced to a BPM value
- **Combo system**: Consecutive hits multiply score; any miss resets combo
- **Visual feedback**: Lane flashes, note explosions, camera shake on misses
- **Pulsing environment**: Lights, colors, or geometry scale in sync with the beat

### Scoring Patterns
- Perfect hit: highest points + combo continues
- Great hit: medium points + combo continues
- Good hit: small points + combo continues
- Miss: zero points, combo resets
- End-of-song accuracy percentage bonus

### Camera Style
- Fixed behind/above the hit zone, looking down the highway
- Slight sway or pulse on beat
- Flash effects on note hits

---

## Three.js Setup for Music 3D

### Rhythm Highway Scene
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);
renderer.setClearColor(0x0a0010);

const scene = new THREE.Scene();
scene.fog = new THREE.Fog(0x0a0010, 10, 35);

const camera = new THREE.PerspectiveCamera(65, canvas.width / canvas.height, 0.1, 50);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, -5);
```

### Beat-Synced Lighting
```javascript
const ambient = new THREE.AmbientLight(0x220033, 0.4);
scene.add(ambient);

// Lane lights that pulse
const laneLights = [];
const laneColors = [0xff3366, 0x33ff66, 0x3366ff, 0xffcc33];
laneColors.forEach((color, i) => {
  const light = new THREE.PointLight(color, 0.3, 8);
  light.position.set(-1.5 + i, 0.5, 0);
  scene.add(light);
  laneLights.push(light);
});
```

---

## Complete Example: Neon Beats

A 3D rhythm highway game. Notes approach the player along 4 colored lanes. Press the matching direction as notes cross the hit line. Build combos for higher scores. The song plays for 60 seconds with procedurally generated beats.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  notes: [],
  laneTargets: [],
  laneLights: [],
  score: 0,
  combo: 0,
  maxCombo: 0,
  gameOver: false,
  running: false,
  paused: false,
  bpm: 120,
  songTime: 0,
  songDuration: 60,
  lastTime: 0,
  animId: null,
  beatIndex: 0,
  nextBeatTime: 0,
  hitResults: { perfect: 0, great: 0, good: 0, miss: 0 },
  laneFlash: [0, 0, 0, 0],
  beatPulse: 0,
  noteSpeed: 8,
  hitZoneZ: 8,
  spawnZ: -20,
  beatPattern: [],
  patternIndex: 0,

  laneColors: [0xff3366, 0x33ff66, 0x3366ff, 0xffcc33],
  laneNames: ['left', 'down', 'up', 'right'],
  laneX: [-1.5, -0.5, 0.5, 1.5],

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x0a0010);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.Fog(0x0a0010, 12, 35);

    this.camera = new THREE.PerspectiveCamera(65, canvas.width / canvas.height, 0.1, 50);
    this.camera.position.set(0, 5, 12);
    this.camera.lookAt(0, 0, -2);

    const ambient = new THREE.AmbientLight(0x220033, 0.3);
    this.scene.add(ambient);
    const topLight = new THREE.DirectionalLight(0x6644aa, 0.4);
    topLight.position.set(0, 10, 5);
    this.scene.add(topLight);

    this.buildHighway();
    this.buildHitZone();
    this.buildLaneLights();
    this.running = false;
    this.loop();
  },

  buildHighway() {
    for (let i = 0; i < 4; i++) {
      const geo = new THREE.PlaneGeometry(0.9, 30);
      const mat = new THREE.MeshStandardMaterial({
        color: 0x1a1a2e, roughness: 0.9, transparent: true, opacity: 0.7
      });
      const lane = new THREE.Mesh(geo, mat);
      lane.rotation.x = -Math.PI / 2;
      lane.position.set(this.laneX[i], 0, -7);
      this.scene.add(lane);

      const edgeMat = new THREE.MeshBasicMaterial({
        color: this.laneColors[i], transparent: true, opacity: 0.2
      });
      const edgeL = new THREE.Mesh(new THREE.PlaneGeometry(0.02, 30), edgeMat);
      edgeL.rotation.x = -Math.PI / 2;
      edgeL.position.set(this.laneX[i] - 0.45, 0.01, -7);
      this.scene.add(edgeL);
      const edgeR = new THREE.Mesh(new THREE.PlaneGeometry(0.02, 30), edgeMat);
      edgeR.rotation.x = -Math.PI / 2;
      edgeR.position.set(this.laneX[i] + 0.45, 0.01, -7);
      this.scene.add(edgeR);
    }
  },

  buildHitZone() {
    this.laneTargets = [];
    for (let i = 0; i < 4; i++) {
      const geo = new THREE.CylinderGeometry(0.35, 0.35, 0.1, 12);
      const mat = new THREE.MeshStandardMaterial({
        color: this.laneColors[i], emissive: this.laneColors[i],
        emissiveIntensity: 0.2, transparent: true, opacity: 0.6
      });
      const target = new THREE.Mesh(geo, mat);
      target.position.set(this.laneX[i], 0.1, this.hitZoneZ);
      this.scene.add(target);
      this.laneTargets.push(target);
    }

    const lineGeo = new THREE.PlaneGeometry(5, 0.08);
    const lineMat = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.5 });
    const hitLine = new THREE.Mesh(lineGeo, lineMat);
    hitLine.rotation.x = -Math.PI / 2;
    hitLine.position.set(0, 0.02, this.hitZoneZ);
    this.scene.add(hitLine);
  },

  buildLaneLights() {
    this.laneLights = [];
    for (let i = 0; i < 4; i++) {
      const light = new THREE.PointLight(this.laneColors[i], 0.2, 6);
      light.position.set(this.laneX[i], 1.5, this.hitZoneZ);
      this.scene.add(light);
      this.laneLights.push(light);
    }
  },

  generateBeatPattern() {
    this.beatPattern = [];
    const beatInterval = 60 / this.bpm;
    const totalBeats = Math.floor(this.songDuration / beatInterval);

    for (let b = 4; b < totalBeats; b++) {
      const time = b * beatInterval;
      const r = Math.random();
      if (r < 0.6) {
        const lane = Math.floor(Math.random() * 4);
        this.beatPattern.push({ time, lane });
      } else if (r < 0.8) {
        const lane1 = Math.floor(Math.random() * 4);
        let lane2 = (lane1 + 1 + Math.floor(Math.random() * 3)) % 4;
        this.beatPattern.push({ time, lane: lane1 });
        this.beatPattern.push({ time: time + beatInterval * 0.5, lane: lane2 });
      }
    }
    this.beatPattern.sort((a, b) => a.time - b.time);
  },

  spawnNote(lane) {
    const geo = new THREE.BoxGeometry(0.7, 0.3, 0.5);
    const color = this.laneColors[lane];
    const mat = new THREE.MeshStandardMaterial({
      color, emissive: color, emissiveIntensity: 0.4,
      metalness: 0.3, roughness: 0.4
    });
    const note = new THREE.Mesh(geo, mat);
    note.position.set(this.laneX[lane], 0.25, this.spawnZ);
    note.userData = { lane, hit: false, missed: false, spawnTime: this.songTime };
    this.scene.add(note);
    this.notes.push(note);
  },

  spawnHitEffect(lane, quality) {
    const colors = { perfect: 0xffff00, great: 0x00ff88, good: 0x44aaff };
    const color = colors[quality] || 0xffffff;
    for (let i = 0; i < 8; i++) {
      const p = new THREE.Mesh(
        new THREE.SphereGeometry(0.05, 4, 4),
        new THREE.MeshBasicMaterial({ color })
      );
      p.position.set(this.laneX[lane], 0.3, this.hitZoneZ);
      p.userData = {
        vel: new THREE.Vector3(
          (Math.random() - 0.5) * 4,
          Math.random() * 3 + 1,
          (Math.random() - 0.5) * 2
        ),
        life: 0.4
      };
      this.scene.add(p);
      this.notes.push(p);
    }
  },

  tryHitLane(lane) {
    let bestNote = null;
    let bestDist = Infinity;

    for (const note of this.notes) {
      if (note.userData.lane !== lane || note.userData.hit || note.userData.missed) continue;
      if (!note.userData.spawnTime && note.userData.spawnTime !== 0) continue;
      const dist = Math.abs(note.position.z - this.hitZoneZ);
      if (dist < bestDist) {
        bestDist = dist;
        bestNote = note;
      }
    }

    if (!bestNote) return;

    let quality = null;
    if (bestDist < 0.5) { quality = 'perfect'; }
    else if (bestDist < 1.2) { quality = 'great'; }
    else if (bestDist < 2.0) { quality = 'good'; }

    if (quality) {
      bestNote.userData.hit = true;
      this.scene.remove(bestNote);
      this.combo++;
      if (this.combo > this.maxCombo) this.maxCombo = this.combo;
      this.hitResults[quality]++;

      const base = quality === 'perfect' ? 100 : quality === 'great' ? 60 : 30;
      const mult = Math.min(1 + Math.floor(this.combo / 10) * 0.5, 4);
      this.score += Math.floor(base * mult);

      this.laneFlash[lane] = 1.0;
      this.spawnHitEffect(lane, quality);
    }
  },

  start() {
    this.score = 0;
    this.combo = 0;
    this.maxCombo = 0;
    this.gameOver = false;
    this.paused = false;
    this.songTime = 0;
    this.patternIndex = 0;
    this.hitResults = { perfect: 0, great: 0, good: 0, miss: 0 };
    this.laneFlash = [0, 0, 0, 0];
    this.beatPulse = 0;
    this.notes.forEach(n => this.scene.remove(n));
    this.notes = [];
    this.generateBeatPattern();
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.score = 0;
    this.combo = 0;
    this.maxCombo = 0;
    this.gameOver = false;
    this.paused = false;
    this.songTime = 0;
    this.notes.forEach(n => this.scene.remove(n));
    this.notes = [];
    this.hitResults = { perfect: 0, great: 0, good: 0, miss: 0 };
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused && !this.gameOver) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      this.songTime += dt;

      while (this.patternIndex < this.beatPattern.length) {
        const beat = this.beatPattern[this.patternIndex];
        const travelTime = (this.hitZoneZ - this.spawnZ) / this.noteSpeed;
        if (beat.time - travelTime <= this.songTime) {
          this.spawnNote(beat.lane);
          this.patternIndex++;
        } else {
          break;
        }
      }

      for (let i = this.notes.length - 1; i >= 0; i--) {
        const n = this.notes[i];
        if (n.userData.vel) {
          n.position.x += n.userData.vel.x * dt;
          n.position.y += n.userData.vel.y * dt;
          n.position.z += n.userData.vel.z * dt;
          n.userData.vel.y -= 9.8 * dt;
          n.userData.life -= dt;
          if (n.userData.life <= 0) {
            this.scene.remove(n);
            this.notes.splice(i, 1);
          }
          continue;
        }

        if (n.userData.hit) {
          this.scene.remove(n);
          this.notes.splice(i, 1);
          continue;
        }

        n.position.z += this.noteSpeed * dt;

        if (n.position.z > this.hitZoneZ + 2.5 && !n.userData.missed) {
          n.userData.missed = true;
          this.hitResults.miss++;
          this.combo = 0;
          this.scene.remove(n);
          this.notes.splice(i, 1);
        }
      }

      const beatInterval = 60 / this.bpm;
      this.beatPulse = Math.max(0, this.beatPulse - dt * 4);
      if (this.songTime % beatInterval < dt) {
        this.beatPulse = 1.0;
      }

      for (let i = 0; i < 4; i++) {
        this.laneFlash[i] = Math.max(0, this.laneFlash[i] - dt * 4);
        this.laneLights[i].intensity = 0.2 + this.laneFlash[i] * 2.0 + this.beatPulse * 0.3;
        this.laneTargets[i].material.emissiveIntensity = 0.2 + this.laneFlash[i] * 0.8;
        this.laneTargets[i].scale.y = 1.0 + this.laneFlash[i] * 0.5;
      }

      if (this.songTime >= this.songDuration && this.notes.length === 0) {
        this.gameOver = true;
        const totalNotes = this.hitResults.perfect + this.hitResults.great +
          this.hitResults.good + this.hitResults.miss;
        if (totalNotes > 0) {
          const accuracy = (this.hitResults.perfect + this.hitResults.great + this.hitResults.good) / totalNotes;
          this.score += Math.floor(accuracy * 500);
        }
      }

      this.camera.position.y = 5 + this.beatPulse * 0.15;
    }
    this.renderer.render(this.scene, this.camera);
  },

  getState() {
    const totalNotes = this.hitResults.perfect + this.hitResults.great +
      this.hitResults.good + this.hitResults.miss;
    return {
      score: this.score,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      songTime: parseFloat(this.songTime.toFixed(1)),
      songDuration: this.songDuration,
      combo: this.combo,
      maxCombo: this.maxCombo,
      hitResults: { ...this.hitResults },
      accuracy: totalNotes > 0 ? parseFloat(
        ((this.hitResults.perfect + this.hitResults.great + this.hitResults.good) / totalNotes * 100).toFixed(1)
      ) : 100,
      bpm: this.bpm,
      upcomingNotes: this.notes
        .filter(n => n.userData.lane !== undefined && !n.userData.hit && !n.userData.missed)
        .sort((a, b) => a.position.z - b.position.z)
        .slice(-5)
        .map(n => ({
          lane: this.laneNames[n.userData.lane],
          laneIndex: n.userData.lane,
          distToHitZone: parseFloat((this.hitZoneZ - n.position.z).toFixed(1))
        }))
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (this.gameOver) return false;
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused) return false;

    const laneMap = { left: 0, down: 1, up: 2, right: 3 };
    if (action in laneMap) {
      this.tryHitLane(laneMap[action]);
      return true;
    }
    if (action === 'action') return true;
    return false;
  },

  getMeta() {
    return {
      name: 'Neon Beats',
      description: 'A 3D rhythm highway game. Notes flow toward you along 4 colored lanes. Press the matching direction as they cross the hit line to score points and build combos.',
      controls: {
        up: 'Hit note in lane 3 (blue)',
        down: 'Hit note in lane 2 (green)',
        left: 'Hit note in lane 1 (red/pink)',
        right: 'Hit note in lane 4 (yellow)',
        action: 'No action (all hits via directional inputs)'
      },
      objective: 'Hit as many notes as possible during the 60-second song. Achieve high accuracy and combo streaks.',
      scoring: 'Perfect: 100 pts, Great: 60 pts, Good: 30 pts, Miss: 0 pts. Combo multiplier up to 4x every 10 hits. Accuracy bonus at end.',
      tips: [
        'Check getState().upcomingNotes to see the next notes approaching.',
        'When distToHitZone is near 0, that note is at the hit line -- press its lane direction.',
        'Build combos for multiplied scores -- one miss resets the multiplier.',
        'The song plays at 120 BPM with procedurally generated patterns.',
        'Lane mapping: left=pink, down=green, up=blue, right=yellow.',
        'Accuracy bonus at song end rewards consistent play over risky combos.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Music 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current score (required) |
| `gameOver` | boolean | Whether song has ended (required) |
| `songTime` | number | Elapsed time in seconds |
| `songDuration` | number | Total song length |
| `combo` | number | Current consecutive hit streak |
| `maxCombo` | number | Best combo achieved |
| `hitResults` | object | Counts per timing quality `{ perfect, great, good, miss }` |
| `accuracy` | number | Overall hit percentage |
| `upcomingNotes` | array | Next notes approaching with lane name and distance to hit zone |

Agent-friendly design: `upcomingNotes` is the critical field for AI agents. Each note includes `lane` (the action to press) and `distToHitZone` (how far the note is from the scoring line). An agent should watch for `distToHitZone` approaching 0 and send the corresponding lane direction.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "music"
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
- [ ] Notes travel along 3D highway lanes toward hit zone
- [ ] Timing windows: perfect, great, good, miss with distinct thresholds
- [ ] Combo system that resets on miss
- [ ] Lights pulse to beat (BPM-synced visual effects)
- [ ] Hit effects: lane flash + particle burst on successful hit
- [ ] upcomingNotes in getState() for agent play
- [ ] No audio fetch -- rhythm is purely visual (no forbidden APIs)
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
