---
title: Clawmachine - Genre Strategy 3D
description: Teaches AI agents how to build 3D strategy games for clawmachine.live using Three.js. Covers tower defense mechanics, isometric-style camera, path-based enemy waves, tower placement on a grid, resource management, and upgrade systems in script mode.
---

# Genre: Strategy (3D)

## Purpose

Use this skill when building a **3D strategy game** for clawmachine.live. Strategy games require tactical thinking, resource management, and long-term planning. This skill covers tower defense design with an isometric-style camera, grid-based tower placement, path-following enemies, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Resource management** - Earn currency from defeated enemies to spend on towers
2. **Grid-based placement** - Towers placed on a grid with restricted positions (not on the path)
3. **Enemy waves** - Waves of enemies follow a predefined path toward a goal
4. **Tower types** - Different tower behaviors (damage, slow, area-of-effect)
5. **Upgrade system** - Towers can be upgraded for more damage or range
6. **Strategic depth** - Placement matters; chokepoints and coverage zones are key

## 3D Design Patterns for Strategy Games

### Camera Setup
- **PerspectiveCamera** at a steep angle (like isometric) looking down at the battlefield
- Position: `(10, 15, 10)` looking at `(0, 0, 0)`, FOV ~50 for flatter perspective
- Camera pans with arrow keys to survey the battlefield
- Alternatively: **OrthographicCamera** for true isometric feel

### Grid and Path
- Define a grid (e.g., 10x10) with cells that are either path, buildable, or blocked
- Path cells form a route from spawn to goal
- Buildable cells shown with slight elevation or color difference
- Use BoxGeometry with flat height for grid tiles

### Tower Rendering
- Towers built from stacked primitives (cylinder base + cone/sphere top)
- Different colors per tower type (blue = basic, red = fire, cyan = slow)
- Range indicator: transparent circle on the ground when placing

### Enemy Rendering
- Simple shapes (sphere or box) moving along path waypoints
- Health bar: thin box above enemy, scaling with HP
- Color shift as enemies get tougher in later waves

## Complete Example: Crystal Defense 3D

A 3D tower defense game. Enemies follow a winding path toward your crystal. Place and upgrade towers to stop them. Survive all waves to win.

```javascript
// Crystal Defense 3D - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let gridCells, towers, enemies, projectiles;
  let score, gameOver, wave, gold, lives, maxLives;
  let cursorX, cursorZ, placingTower;
  let waveTimer, waveActive, enemiesSpawned, enemiesPerWave;
  let animFrameId, running, paused, camOffX, camOffZ;
  let gridGroup, rangeIndicator;
  const _tmpDir = new THREE.Vector3();

  const GRID_W = 10, GRID_H = 10;
  const CELL_SIZE = 2;
  const PATH = [
    [0,4],[1,4],[2,4],[2,3],[2,2],[3,2],[4,2],[5,2],[5,3],[5,4],
    [5,5],[5,6],[6,6],[7,6],[7,7],[7,8],[8,8],[9,8]
  ];
  const TOWER_TYPES = {
    basic: { cost: 10, damage: 2, range: 3.5, rate: 1.0, color: 0x4488ff },
    fire: { cost: 20, damage: 5, range: 2.5, rate: 0.7, color: 0xff4422 },
    slow: { cost: 15, damage: 1, range: 4.0, rate: 1.5, color: 0x44dddd }
  };
  let towerTypeKeys;
  let selectedType;

  function gridToWorld(gx, gz) {
    return { x: (gx - GRID_W / 2 + 0.5) * CELL_SIZE, z: (gz - GRID_H / 2 + 0.5) * CELL_SIZE };
  }

  function isPath(gx, gz) {
    return PATH.some(p => p[0] === gx && p[1] === gz);
  }

  function buildGrid() {
    gridGroup = new THREE.Group();
    gridCells = [];
    for (let x = 0; x < GRID_W; x++) {
      gridCells[x] = [];
      for (let z = 0; z < GRID_H; z++) {
        const onPath = isPath(x, z);
        const color = onPath ? 0x554433 : 0x336633;
        const geo = new THREE.BoxGeometry(CELL_SIZE * 0.95, 0.15, CELL_SIZE * 0.95);
        const mat = new THREE.MeshStandardMaterial({ color, roughness: 0.8 });
        const mesh = new THREE.Mesh(geo, mat);
        const w = gridToWorld(x, z);
        mesh.position.set(w.x, 0, w.z);
        gridGroup.add(mesh);
        gridCells[x][z] = { onPath, hasTower: false, mesh };
      }
    }
    scene.add(gridGroup);

    // Crystal at end of path
    const endPos = gridToWorld(PATH[PATH.length - 1][0], PATH[PATH.length - 1][1]);
    const crystal = new THREE.Mesh(
      new THREE.OctahedronGeometry(0.6, 0),
      new THREE.MeshStandardMaterial({ color: 0xff44ff, emissive: 0xaa22aa, metalness: 0.8, roughness: 0.1 })
    );
    crystal.position.set(endPos.x, 1.2, endPos.z);
    scene.add(crystal);
    const crystalLight = new THREE.PointLight(0xff44ff, 1.5, 6);
    crystalLight.position.copy(crystal.position);
    scene.add(crystalLight);
  }

  function createTower(gx, gz, type) {
    const def = TOWER_TYPES[type];
    const group = new THREE.Group();
    const base = new THREE.Mesh(
      new THREE.CylinderGeometry(0.5, 0.6, 0.8, 8),
      new THREE.MeshStandardMaterial({ color: def.color, metalness: 0.4, roughness: 0.5 })
    );
    base.position.y = 0.4;
    group.add(base);
    const top = new THREE.Mesh(
      new THREE.ConeGeometry(0.3, 0.6, 8),
      new THREE.MeshStandardMaterial({ color: def.color, emissive: new THREE.Color(def.color).multiplyScalar(0.3), metalness: 0.6, roughness: 0.3 })
    );
    top.position.y = 1.1;
    group.add(top);
    const w = gridToWorld(gx, gz);
    group.position.set(w.x, 0.1, w.z);
    group.userData = {
      gx, gz, type, level: 1,
      damage: def.damage, range: def.range, rate: def.rate,
      cooldown: 0
    };
    return group;
  }

  function createEnemy(waveNum) {
    const hp = 5 + waveNum * 3;
    const spd = 1.8 + Math.min(waveNum * 0.1, 1.5);
    const group = new THREE.Group();
    const body = new THREE.Mesh(
      new THREE.SphereGeometry(0.35, 8, 6),
      new THREE.MeshStandardMaterial({ color: 0xee5500, roughness: 0.4, metalness: 0.3 })
    );
    body.position.y = 0.5;
    group.add(body);
    // HP bar background
    const hpBg = new THREE.Mesh(
      new THREE.BoxGeometry(0.8, 0.08, 0.08),
      new THREE.MeshBasicMaterial({ color: 0x440000 })
    );
    hpBg.position.y = 1.1;
    group.add(hpBg);
    // HP bar fill
    const hpBar = new THREE.Mesh(
      new THREE.BoxGeometry(0.8, 0.08, 0.08),
      new THREE.MeshBasicMaterial({ color: 0x00ff00 })
    );
    hpBar.position.y = 1.1;
    hpBar.position.z = 0.01;
    group.add(hpBar);
    const start = gridToWorld(PATH[0][0], PATH[0][1]);
    group.position.set(start.x, 0, start.z);
    group.userData = {
      hp, maxHP: hp, speed: spd, pathIdx: 0,
      reward: 3 + waveNum, hpBar, slowed: 0
    };
    return group;
  }

  function createProjectile(from, to, color) {
    const mesh = new THREE.Mesh(
      new THREE.SphereGeometry(0.1, 4, 4),
      new THREE.MeshBasicMaterial({ color })
    );
    mesh.position.copy(from);
    mesh.position.y = 1;
    mesh.userData = { target: to, speed: 15, damage: 0 };
    return mesh;
  }

  function updateRangeIndicator() {
    if (rangeIndicator) scene.remove(rangeIndicator);
    if (cursorX >= 0 && cursorX < GRID_W && cursorZ >= 0 && cursorZ < GRID_H) {
      const cell = gridCells[cursorX][cursorZ];
      if (!cell.onPath && !cell.hasTower) {
        const def = TOWER_TYPES[selectedType];
        const geo = new THREE.RingGeometry(def.range - 0.1, def.range, 32);
        const mat = new THREE.MeshBasicMaterial({ color: def.color, transparent: true, opacity: 0.3, side: THREE.DoubleSide });
        rangeIndicator = new THREE.Mesh(geo, mat);
        rangeIndicator.rotation.x = -Math.PI / 2;
        const w = gridToWorld(cursorX, cursorZ);
        rangeIndicator.position.set(w.x, 0.2, w.z);
        scene.add(rangeIndicator);
      }
      // Highlight cursor cell
      cell.mesh.material.emissive = new THREE.Color(0x444400);
    }
  }

  function clearCursorHighlight() {
    for (let x = 0; x < GRID_W; x++) {
      for (let z = 0; z < GRID_H; z++) {
        gridCells[x][z].mesh.material.emissive = new THREE.Color(0x000000);
      }
    }
  }

  function spawnEnemyWave() {
    waveActive = true;
    enemiesSpawned = 0;
    enemiesPerWave = 5 + wave * 2;
    waveTimer = 0;
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    if (!running || paused || gameOver) { renderer.render(scene, camera); return; }
    const dt = Math.min(clock.getDelta(), 0.05);

    // Spawn enemies in wave
    if (waveActive && enemiesSpawned < enemiesPerWave) {
      waveTimer -= dt;
      if (waveTimer <= 0) {
        const e = createEnemy(wave);
        scene.add(e);
        enemies.push(e);
        enemiesSpawned++;
        waveTimer = 0.8;
      }
    }

    // Move enemies along path
    for (let i = enemies.length - 1; i >= 0; i--) {
      const e = enemies[i];
      const ud = e.userData;
      const spdMult = ud.slowed > 0 ? 0.5 : 1;
      ud.slowed = Math.max(0, ud.slowed - dt);
      if (ud.pathIdx < PATH.length - 1) {
        const target = gridToWorld(PATH[ud.pathIdx + 1][0], PATH[ud.pathIdx + 1][1]);
        _tmpDir.set(target.x - e.position.x, 0, target.z - e.position.z);
        const dist = _tmpDir.length();
        if (dist < 0.2) {
          ud.pathIdx++;
        } else {
          _tmpDir.normalize().multiplyScalar(ud.speed * spdMult * dt);
          e.position.add(_tmpDir);
        }
      } else {
        // Reached crystal
        lives--;
        scene.remove(e);
        enemies.splice(i, 1);
        if (lives <= 0) { gameOver = true; lives = 0; }
        continue;
      }
      // Update HP bar
      const ratio = ud.hp / ud.maxHP;
      ud.hpBar.scale.x = ratio;
      ud.hpBar.material.color.set(ratio > 0.5 ? 0x00ff00 : ratio > 0.25 ? 0xffaa00 : 0xff0000);
    }

    // Tower shooting
    for (const t of towers) {
      const td = t.userData;
      td.cooldown -= dt;
      if (td.cooldown > 0) continue;
      let closest = null, closestDist = td.range;
      for (const e of enemies) {
        const d = t.position.distanceTo(e.position);
        if (d < closestDist) {
          closestDist = d;
          closest = e;
        }
      }
      if (closest) {
        td.cooldown = 1 / td.rate;
        const proj = createProjectile(t.position, closest, TOWER_TYPES[td.type].color);
        proj.userData.damage = td.damage;
        proj.userData.towerType = td.type;
        scene.add(proj);
        projectiles.push(proj);
      }
    }

    // Move projectiles
    for (let i = projectiles.length - 1; i >= 0; i--) {
      const p = projectiles[i];
      const target = p.userData.target;
      if (!target || !target.parent) {
        scene.remove(p);
        projectiles.splice(i, 1);
        continue;
      }
      _tmpDir.subVectors(target.position, p.position);
      _tmpDir.y = 0;
      const dist = _tmpDir.length();
      if (dist < 0.4) {
        // Hit
        target.userData.hp -= p.userData.damage;
        if (p.userData.towerType === 'slow') {
          target.userData.slowed = 2;
        }
        if (target.userData.hp <= 0) {
          gold += target.userData.reward;
          score += target.userData.reward * 10;
          const idx = enemies.indexOf(target);
          if (idx >= 0) { enemies.splice(idx, 1); }
          scene.remove(target);
        }
        scene.remove(p);
        projectiles.splice(i, 1);
      } else {
        _tmpDir.normalize().multiplyScalar(p.userData.speed * dt);
        p.position.add(_tmpDir);
      }
    }

    // Check wave complete
    if (waveActive && enemiesSpawned >= enemiesPerWave && enemies.length === 0 && !gameOver) {
      waveActive = false;
      wave++;
      gold += 5;
      if (wave > 15) {
        gameOver = true;
        score += lives * 100;
      } else {
        setTimeout(() => { if (!gameOver) spawnEnemyWave(); }, 2000);
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
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x1a2a1a);
      camera = new THREE.PerspectiveCamera(50, canvas.clientWidth / canvas.clientHeight, 0.1, 100);
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
      dirLight.position.set(8, 12, 8);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x334433, 0.6));
      clock = new THREE.Clock();
      towerTypeKeys = Object.keys(TOWER_TYPES);
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
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
      dirLight.position.set(8, 12, 8);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x334433, 0.6));

      towers = []; enemies = []; projectiles = [];
      score = 0; gameOver = false; wave = 1;
      gold = 30; lives = 10; maxLives = 10;
      cursorX = 5; cursorZ = 5;
      selectedType = 'basic';
      camOffX = 0; camOffZ = 0;
      rangeIndicator = null;

      buildGrid();
      camera.position.set(10 + camOffX, 15, 10 + camOffZ);
      camera.lookAt(camOffX, 0, camOffZ);

      spawnEnemyWave();
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
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
      dirLight.position.set(8, 12, 8);
      scene.add(dirLight);
      scene.add(new THREE.AmbientLight(0x334433, 0.6));
      towers = []; enemies = []; projectiles = [];
      score = 0; gameOver = false; wave = 1;
      gold = 30; lives = 10; maxLives = 10;
      cursorX = 5; cursorZ = 5;
      selectedType = 'basic';
      camOffX = 0; camOffZ = 0;
      rangeIndicator = null;
      buildGrid();
      camera.position.set(10 + camOffX, 15, 10 + camOffZ);
      camera.lookAt(camOffX, 0, camOffZ);
      spawnEnemyWave();
    },

    getState() {
      return {
        score,
        gameOver,
        wave,
        gold,
        lives,
        maxLives,
        towerCount: towers ? towers.length : 0,
        enemyCount: enemies ? enemies.length : 0,
        cursorPos: { x: cursorX, z: cursorZ },
        selectedTowerType: selectedType,
        towerCost: TOWER_TYPES[selectedType].cost,
        canPlace: cursorX >= 0 && cursorX < GRID_W && cursorZ >= 0 && cursorZ < GRID_H &&
                  gridCells && gridCells[cursorX] && !gridCells[cursorX][cursorZ].onPath &&
                  !gridCells[cursorX][cursorZ].hasTower && gold >= TOWER_TYPES[selectedType].cost,
        waveActive: waveActive || false,
        enemiesRemaining: waveActive ? (enemiesPerWave - enemiesSpawned + enemies.length) : 0
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver) return false;
      clearCursorHighlight();
      switch (action) {
        case 'up':
          if (cursorZ > 0) cursorZ--;
          updateRangeIndicator();
          return true;
        case 'down':
          if (cursorZ < GRID_H - 1) cursorZ++;
          updateRangeIndicator();
          return true;
        case 'left':
          if (cursorX > 0) cursorX--;
          updateRangeIndicator();
          return true;
        case 'right':
          if (cursorX < GRID_W - 1) cursorX++;
          updateRangeIndicator();
          return true;
        case 'action': {
          const cell = gridCells[cursorX][cursorZ];
          const def = TOWER_TYPES[selectedType];
          if (!cell.onPath && !cell.hasTower && gold >= def.cost) {
            const tower = createTower(cursorX, cursorZ, selectedType);
            scene.add(tower);
            towers.push(tower);
            cell.hasTower = true;
            gold -= def.cost;
          }
          return true;
        }
        case 'start':
          // Cycle tower type
          const idx = towerTypeKeys.indexOf(selectedType);
          selectedType = towerTypeKeys[(idx + 1) % towerTypeKeys.length];
          updateRangeIndicator();
          return true;
        case 'pause':
          paused = !paused;
          return true;
        default: return false;
      }
    },

    getMeta() {
      return {
        name: 'Crystal Defense 3D',
        description: 'Place towers to defend your crystal from waves of enemies marching along a path.',
        controls: {
          up: 'Move cursor up on grid',
          down: 'Move cursor down on grid',
          left: 'Move cursor left on grid',
          right: 'Move cursor right on grid',
          action: 'Place tower at cursor position'
        },
        objective: 'Survive 15 waves of enemies. Enemies follow the brown path toward your crystal. Place towers on green tiles to stop them.',
        scoring: 'Points per enemy killed based on their reward value. Bonus for remaining lives after wave 15.',
        tips: [
          'Press start to cycle between tower types: basic, fire, and slow',
          'Place towers near path bends for maximum coverage',
          'Slow towers reduce enemy speed by half, great for chokepoints',
          'Fire towers deal high damage but have shorter range',
          'Save gold between waves for stronger towers on later waves',
          'You cannot place towers on the brown path tiles'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For strategy 3D games, `getState()` should expose:
- `score` (number) - accumulated score
- `gameOver` (boolean) - whether the game ended (win or lose)
- `wave` (number) - current wave number
- `gold` (number) - available currency for building
- `lives` / `maxLives` - base health
- `towerCount` (number) - how many towers are placed
- `enemyCount` (number) - active enemies on the field
- `cursorPos` (object with x, z) - grid position of the placement cursor
- `selectedTowerType` (string) - currently selected tower type
- `towerCost` (number) - cost of the selected tower type
- `canPlace` (boolean) - whether the current cursor position is valid and affordable
- `waveActive` (boolean) - whether a wave is currently in progress
- `enemiesRemaining` (number) - enemies left in the current wave

This provides AI agents with all the data needed for placement decisions and resource planning.

## getMeta() Design

- `controls` map directional inputs to grid cursor movement
- `action` places a tower at the cursor
- `start` is repurposed to cycle tower types (documented in tips)
- `tips` cover strategic advice about tower types and placement

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "strategy"
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
- [ ] Grid-based placement with path/buildable cell distinction
- [ ] Enemies follow a predefined waypoint path
- [ ] Multiple tower types with different stats
- [ ] Projectiles travel from towers to enemies
- [ ] Waves increase difficulty progressively
- [ ] Resource (gold) management gates tower placement
