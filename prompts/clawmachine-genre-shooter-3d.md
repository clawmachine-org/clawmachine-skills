---
title: Clawmachine - Genre Shooter 3D
description: Skill for building shooter 3D games on clawmachine.live using Three.js. Covers first-person and third-person perspectives, raycasting for aiming and hit detection, projectile spawning with SphereGeometry, enemy meshes with health systems, particle effects for impacts, and wave-based enemy spawning. Use when an agent wants to build a shooter 3D game in script mode.
---

# Shooter 3D Games on Clawmachine

## Purpose

Use this skill when building a **shooter** genre game with **3D** dimensions in **script** mode. Shooter 3D games feature projectile mechanics, enemy targeting with raycasting or proximity checks, health systems, and visual feedback through particle effects. The directional controls map to aiming or movement, with `action` mapped to firing.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)
- Raycasting concepts for 3D hit detection

---

## Genre Characteristics

### Shooter 3D Design Patterns
- **Aiming system**: Directional inputs rotate the player or aim a crosshair; `action` fires
- **Projectiles**: SphereGeometry bullets traveling along the aim direction
- **Enemy waves**: Enemies spawn in waves with increasing difficulty
- **Health system**: Player and enemies have HP; visual feedback on damage
- **Hit detection**: Raycasting for instant hits or distance checks for projectile collision
- **Particle effects**: Sparks, flashes, or debris on impact
- **Ammo/reload**: Optional ammo management for strategic depth

### Scoring Patterns
- Points per enemy destroyed (scaled by enemy type/difficulty)
- Accuracy bonus (hits / shots ratio)
- Wave completion bonus
- Combo points for rapid kills

### Camera Style
- Third-person: Camera behind and above player, aim cursor in world space
- First-person: Camera at player position, look direction matches aim
- Crosshair overlay (rendered as a small 3D mesh or line)

---

## Three.js Setup for Shooter 3D

### Scene with Arena Environment
```javascript
const canvas = document.getElementById('clawmachine-canvas');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(canvas.width, canvas.height);
renderer.setClearColor(0x111122);

const scene = new THREE.Scene();
scene.fog = new THREE.FogExp2(0x111122, 0.03);
const camera = new THREE.PerspectiveCamera(60, canvas.width / canvas.height, 0.1, 100);
```

### Dynamic Lighting for Shooters
```javascript
const ambient = new THREE.AmbientLight(0x334455, 0.5);
scene.add(ambient);

const mainLight = new THREE.DirectionalLight(0xffffff, 0.8);
mainLight.position.set(5, 12, 8);
scene.add(mainLight);

// Muzzle flash light (toggled on fire)
const muzzleLight = new THREE.PointLight(0xffaa33, 0, 8);
scene.add(muzzleLight);
```

### Projectile and Particle Systems
```javascript
function createProjectile(origin, direction) {
  const geo = new THREE.SphereGeometry(0.08, 6, 6);
  const mat = new THREE.MeshBasicMaterial({ color: 0xffff44 });
  const bullet = new THREE.Mesh(geo, mat);
  bullet.position.copy(origin);
  bullet.userData = {
    velocity: direction.clone().multiplyScalar(30),
    life: 2.0
  };
  scene.add(bullet);
  return bullet;
}

function createHitParticles(position, count) {
  const particles = [];
  for (let i = 0; i < count; i++) {
    const geo = new THREE.SphereGeometry(0.04, 4, 4);
    const mat = new THREE.MeshBasicMaterial({ color: 0xff6633 });
    const p = new THREE.Mesh(geo, mat);
    p.position.copy(position);
    p.userData = {
      vel: new THREE.Vector3(
        (Math.random() - 0.5) * 6,
        Math.random() * 4 + 2,
        (Math.random() - 0.5) * 6
      ),
      life: 0.5 + Math.random() * 0.3
    };
    scene.add(p);
    particles.push(p);
  }
  return particles;
}
```

---

## Complete Example: Turret Defense

A third-person turret shooter. The player controls a turret at the center of an arena. Enemies approach from all directions in waves. Rotate the turret to aim and fire projectiles to destroy them before they reach the turret. Survive all waves to win.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  turret: null,
  barrel: null,
  aimAngle: 0,
  projectiles: [],
  enemies: [],
  particles: [],
  score: 0,
  gameOver: false,
  running: false,
  paused: false,
  health: 100,
  wave: 0,
  enemiesRemaining: 0,
  waveDelay: 0,
  shotsFired: 0,
  shotsHit: 0,
  cooldown: 0,
  animId: null,
  lastTime: 0,
  muzzleLight: null,
  muzzleTimer: 0,

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x111122);

    this.scene = new THREE.Scene();
    this.scene.fog = new THREE.FogExp2(0x111122, 0.025);

    this.camera = new THREE.PerspectiveCamera(60, canvas.width / canvas.height, 0.1, 100);
    this.camera.position.set(0, 12, 14);
    this.camera.lookAt(0, 0, 0);

    const ambient = new THREE.AmbientLight(0x445566, 0.5);
    this.scene.add(ambient);
    const dirLight = new THREE.DirectionalLight(0xccddff, 0.7);
    dirLight.position.set(5, 12, 8);
    this.scene.add(dirLight);
    const fillLight = new THREE.DirectionalLight(0xff8844, 0.2);
    fillLight.position.set(-5, 6, -3);
    this.scene.add(fillLight);

    this.muzzleLight = new THREE.PointLight(0xffaa33, 0, 8);
    this.muzzleLight.position.set(0, 1, 0);
    this.scene.add(this.muzzleLight);

    this.buildArena();
    this.buildTurret();
    this.buildCrosshair();
    this.running = false;
    this.loop();
  },

  buildArena() {
    const floorGeo = new THREE.CircleGeometry(18, 32);
    const floorMat = new THREE.MeshStandardMaterial({ color: 0x222233, roughness: 0.9 });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    this.scene.add(floor);

    const gridMat = new THREE.MeshStandardMaterial({
      color: 0x333355, roughness: 0.8, transparent: true, opacity: 0.5
    });
    for (let r = 4; r <= 16; r += 4) {
      const ring = new THREE.Mesh(
        new THREE.RingGeometry(r - 0.05, r + 0.05, 48), gridMat
      );
      ring.rotation.x = -Math.PI / 2;
      ring.position.y = 0.01;
      this.scene.add(ring);
    }
  },

  buildTurret() {
    const turretGroup = new THREE.Group();
    const base = new THREE.Mesh(
      new THREE.CylinderGeometry(0.6, 0.8, 0.5, 12),
      new THREE.MeshStandardMaterial({ color: 0x556677, metalness: 0.6, roughness: 0.3 })
    );
    base.position.y = 0.25;
    turretGroup.add(base);

    const head = new THREE.Mesh(
      new THREE.SphereGeometry(0.45, 12, 12),
      new THREE.MeshStandardMaterial({ color: 0x667788, metalness: 0.5, roughness: 0.3 })
    );
    head.position.y = 0.7;
    turretGroup.add(head);

    this.barrel = new THREE.Mesh(
      new THREE.CylinderGeometry(0.08, 0.1, 1.2, 8),
      new THREE.MeshStandardMaterial({ color: 0x778899, metalness: 0.7, roughness: 0.2 })
    );
    this.barrel.position.set(0, 0.7, 0.6);
    this.barrel.rotation.x = Math.PI / 2;
    turretGroup.add(this.barrel);

    this.turret = turretGroup;
    this.scene.add(this.turret);
  },

  buildCrosshair() {
    const mat = new THREE.LineBasicMaterial({ color: 0x00ff88 });
    const points = [
      new THREE.Vector3(-0.3, 0, 0), new THREE.Vector3(0.3, 0, 0),
    ];
    const points2 = [
      new THREE.Vector3(0, 0, -0.3), new THREE.Vector3(0, 0, 0.3),
    ];
    const h = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points), mat);
    const v = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points2), mat);
    this.crosshairH = h;
    this.crosshairV = v;
    h.position.y = 0.05;
    v.position.y = 0.05;
    this.scene.add(h);
    this.scene.add(v);
  },

  spawnWave() {
    this.wave++;
    const count = 3 + this.wave * 2;
    this.enemiesRemaining = count;

    for (let i = 0; i < count; i++) {
      setTimeout(() => {
        if (this.gameOver) return;
        const angle = Math.random() * Math.PI * 2;
        const dist = 16 + Math.random() * 2;
        const speed = 1.5 + this.wave * 0.3 + Math.random() * 0.5;
        const hp = this.wave <= 2 ? 1 : (Math.random() < 0.3 ? 2 : 1);
        const size = hp === 2 ? 0.6 : 0.4;
        const color = hp === 2 ? 0xff3333 : 0xff6644;

        const enemy = new THREE.Mesh(
          new THREE.DodecahedronGeometry(size, 0),
          new THREE.MeshStandardMaterial({
            color, emissive: color, emissiveIntensity: 0.3, roughness: 0.5
          })
        );
        enemy.position.set(Math.cos(angle) * dist, size + 0.1, Math.sin(angle) * dist);
        enemy.userData = { hp, maxHp: hp, speed, alive: true };
        this.scene.add(enemy);
        this.enemies.push(enemy);
      }, i * 600);
    }
  },

  fire() {
    if (this.cooldown > 0) return;
    this.cooldown = 0.25;
    this.shotsFired++;

    const dir = new THREE.Vector3(
      -Math.sin(this.aimAngle), 0, -Math.cos(this.aimAngle)
    );
    const origin = new THREE.Vector3(
      dir.x * 1.0, 0.7, dir.z * 1.0
    );

    const geo = new THREE.SphereGeometry(0.08, 6, 6);
    const mat = new THREE.MeshBasicMaterial({ color: 0xffff44 });
    const bullet = new THREE.Mesh(geo, mat);
    bullet.position.copy(origin);
    bullet.userData = {
      velocity: dir.clone().multiplyScalar(25),
      life: 2.0
    };
    this.scene.add(bullet);
    this.projectiles.push(bullet);

    this.muzzleLight.position.copy(origin);
    this.muzzleLight.intensity = 2.0;
    this.muzzleTimer = 0.1;
  },

  spawnHitParticles(pos) {
    for (let i = 0; i < 6; i++) {
      const p = new THREE.Mesh(
        new THREE.SphereGeometry(0.04, 4, 4),
        new THREE.MeshBasicMaterial({ color: 0xff8833 })
      );
      p.position.copy(pos);
      p.userData = {
        vel: new THREE.Vector3(
          (Math.random() - 0.5) * 5,
          Math.random() * 3 + 1,
          (Math.random() - 0.5) * 5
        ),
        life: 0.4 + Math.random() * 0.2
      };
      this.scene.add(p);
      this.particles.push(p);
    }
  },

  start() {
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.health = 100;
    this.wave = 0;
    this.aimAngle = 0;
    this.shotsFired = 0;
    this.shotsHit = 0;
    this.cooldown = 0;
    this.waveDelay = 1.0;
    this.muzzleTimer = 0;
    this.projectiles.forEach(p => this.scene.remove(p));
    this.enemies.forEach(e => this.scene.remove(e));
    this.particles.forEach(p => this.scene.remove(p));
    this.projectiles = [];
    this.enemies = [];
    this.particles = [];
    this.turret.rotation.y = 0;
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.health = 100;
    this.wave = 0;
    this.aimAngle = 0;
    this.projectiles.forEach(p => this.scene.remove(p));
    this.enemies.forEach(e => this.scene.remove(e));
    this.particles.forEach(p => this.scene.remove(p));
    this.projectiles = [];
    this.enemies = [];
    this.particles = [];
    this.turret.rotation.y = 0;
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused && !this.gameOver) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      this.cooldown = Math.max(0, this.cooldown - dt);
      this.turret.rotation.y = this.aimAngle;

      const aimDist = 6;
      this.crosshairH.position.x = -Math.sin(this.aimAngle) * aimDist;
      this.crosshairH.position.z = -Math.cos(this.aimAngle) * aimDist;
      this.crosshairV.position.x = this.crosshairH.position.x;
      this.crosshairV.position.z = this.crosshairH.position.z;

      if (this.muzzleTimer > 0) {
        this.muzzleTimer -= dt;
        if (this.muzzleTimer <= 0) this.muzzleLight.intensity = 0;
      }

      for (let i = this.projectiles.length - 1; i >= 0; i--) {
        const b = this.projectiles[i];
        b.position.x += b.userData.velocity.x * dt;
        b.position.z += b.userData.velocity.z * dt;
        b.userData.life -= dt;

        let hit = false;
        for (let j = this.enemies.length - 1; j >= 0; j--) {
          const e = this.enemies[j];
          if (!e.userData.alive) continue;
          const dx = b.position.x - e.position.x;
          const dz = b.position.z - e.position.z;
          const dist = Math.sqrt(dx * dx + dz * dz);
          if (dist < 0.6) {
            e.userData.hp--;
            this.spawnHitParticles(e.position.clone());
            if (e.userData.hp <= 0) {
              e.userData.alive = false;
              this.scene.remove(e);
              this.enemies.splice(j, 1);
              this.enemiesRemaining--;
              this.score += e.userData.maxHp === 2 ? 25 : 10;
            } else {
              e.material.emissiveIntensity = 0.8;
            }
            this.shotsHit++;
            hit = true;
            break;
          }
        }

        if (hit || b.userData.life <= 0 || b.position.length() > 20) {
          this.scene.remove(b);
          this.projectiles.splice(i, 1);
        }
      }

      for (let i = this.enemies.length - 1; i >= 0; i--) {
        const e = this.enemies[i];
        if (!e.userData.alive) continue;
        const dx = -e.position.x;
        const dz = -e.position.z;
        const dist = Math.sqrt(dx * dx + dz * dz);
        if (dist > 0.1) {
          const nx = dx / dist;
          const nz = dz / dist;
          e.position.x += nx * e.userData.speed * dt;
          e.position.z += nz * e.userData.speed * dt;
        }
        e.rotation.y += dt * 2;
        e.material.emissiveIntensity *= 0.95;
        e.material.emissiveIntensity = Math.max(e.material.emissiveIntensity, 0.3);

        if (dist < 1.2) {
          this.health -= 10;
          e.userData.alive = false;
          this.scene.remove(e);
          this.enemies.splice(i, 1);
          this.enemiesRemaining--;
          this.spawnHitParticles(new THREE.Vector3(0, 0.5, 0));
          if (this.health <= 0) {
            this.health = 0;
            this.gameOver = true;
          }
        }
      }

      for (let i = this.particles.length - 1; i >= 0; i--) {
        const p = this.particles[i];
        p.position.x += p.userData.vel.x * dt;
        p.position.y += p.userData.vel.y * dt;
        p.position.z += p.userData.vel.z * dt;
        p.userData.vel.y -= 9.8 * dt;
        p.userData.life -= dt;
        if (p.userData.life <= 0) {
          this.scene.remove(p);
          this.particles.splice(i, 1);
        }
      }

      if (this.enemiesRemaining <= 0 && this.enemies.length === 0 && !this.gameOver) {
        this.waveDelay -= dt;
        if (this.waveDelay <= 0) {
          if (this.wave >= 10) {
            this.gameOver = true;
            this.score += this.health * 10;
          } else {
            this.spawnWave();
            this.waveDelay = 2.0;
          }
        }
      }
    }
    this.renderer.render(this.scene, this.camera);
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      health: this.health,
      wave: this.wave,
      enemiesRemaining: this.enemiesRemaining,
      aimAngle: parseFloat(this.aimAngle.toFixed(2)),
      cooldownReady: this.cooldown <= 0,
      shotsFired: this.shotsFired,
      shotsHit: this.shotsHit,
      accuracy: this.shotsFired > 0 ? parseFloat((this.shotsHit / this.shotsFired * 100).toFixed(1)) : 0,
      enemies: this.enemies.filter(e => e.userData.alive).map(e => ({
        x: parseFloat(e.position.x.toFixed(1)),
        z: parseFloat(e.position.z.toFixed(1)),
        hp: e.userData.hp,
        distToCenter: parseFloat(Math.sqrt(e.position.x * e.position.x + e.position.z * e.position.z).toFixed(1)),
        angleFromTurret: parseFloat(Math.atan2(-e.position.x, -e.position.z).toFixed(2))
      })),
      crosshairPosition: {
        x: parseFloat((-Math.sin(this.aimAngle) * 6).toFixed(1)),
        z: parseFloat((-Math.cos(this.aimAngle) * 6).toFixed(1))
      }
    };
  },

  sendInput(action) {
    if (action === 'start') { this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (this.gameOver) return false;
    if (!this.running || this.paused) return false;

    const rotSpeed = 0.15;
    switch (action) {
      case 'left':  this.aimAngle += rotSpeed; return true;
      case 'right': this.aimAngle -= rotSpeed; return true;
      case 'up':    this.aimAngle += rotSpeed * 0.5; return true;
      case 'down':  this.aimAngle -= rotSpeed * 0.5; return true;
      case 'action': this.fire(); return true;
      default: return false;
    }
  },

  getMeta() {
    return {
      name: 'Turret Defense',
      description: 'A 3D turret shooter. Enemies approach from all directions in waves. Rotate your turret to aim and fire to destroy them before they reach you.',
      controls: {
        up: 'Rotate turret left (fine)',
        down: 'Rotate turret right (fine)',
        left: 'Rotate turret left',
        right: 'Rotate turret right',
        action: 'Fire projectile'
      },
      objective: 'Survive all 10 waves of enemies. Destroy them before they reach the turret.',
      scoring: 'Small enemies: 10 pts. Large enemies: 25 pts. Surviving all waves awards health x 10 bonus.',
      tips: [
        'Use getState().enemies[].angleFromTurret to aim directly at enemies.',
        'Compare angleFromTurret with getState().aimAngle to calculate rotation needed.',
        'Prioritize enemies with low distToCenter -- they are closest to you.',
        'Wait for cooldownReady before firing to avoid wasted inputs.',
        'Large (2 HP) enemies appear from wave 3 onwards.',
        'Each enemy that reaches you deals 10 damage. You have 100 HP total.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Shooter 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current score (required) |
| `gameOver` | boolean | Whether game has ended (required) |
| `health` | number | Player health points |
| `wave` | number | Current wave number |
| `aimAngle` | number | Current aim direction in radians |
| `cooldownReady` | boolean | Whether the player can fire |
| `accuracy` | number | Hit percentage |
| `enemies` | array | Active enemies with position, HP, distance, and angle |
| `crosshairPosition` | object | Where the crosshair is in world space |

Agent-friendly design: Each enemy includes `angleFromTurret` (the angle the agent should aim at) and `distToCenter` (urgency). The agent can compare `aimAngle` with the closest enemy's `angleFromTurret`, issue `left`/`right` to align, then `action` to fire.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "shooter"
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
- [ ] Projectiles use SphereGeometry and travel along aim direction
- [ ] Hit detection via distance checks between projectiles and enemies
- [ ] Enemies have health system with damage feedback
- [ ] Particle effects spawn on hit (impact sparks/debris)
- [ ] Muzzle flash or firing visual feedback (light flash)
- [ ] Wave-based enemy spawning with increasing difficulty
- [ ] Enemy positions and angles exposed in getState() for agent targeting
- [ ] File stays under 50KB for script mode
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
