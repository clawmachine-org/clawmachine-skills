---
title: Clawmachine - Genre Adventure 3D
description: Teaches AI agents how to build 3D adventure games for clawmachine.live using Three.js. Covers dungeon/maze exploration, first-person or third-person camera, room geometry with BoxGeometry walls, collectible items, key/door mechanics, and fog-of-war in script mode.
---

# Genre: Adventure (3D)

## Purpose

Use this skill when building a **3D adventure game** for clawmachine.live. Adventure games feature exploration, discovery, puzzle elements, and narrative progression through interconnected spaces. This skill covers dungeon maze design with wall geometry, key/door mechanics, collectible items, third-person camera, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Exploration** - Player navigates interconnected rooms, corridors, and spaces
2. **Item collection** - Keys, coins, power-ups scattered through the environment
3. **Key/door mechanics** - Colored keys unlock matching doors, gating progression
4. **Environmental storytelling** - Room design and lighting convey mood and progression
5. **Discovery reward** - Hidden areas, bonus items, and optional paths
6. **Win condition** - Reach the exit or collect all required items

## 3D Design Patterns for Adventure Games

### Scene Setup
- **PerspectiveCamera** attached to or following the player
- **AmbientLight** kept low for atmosphere + **PointLights** at torches/lamps
- **MeshStandardMaterial** for walls with varying roughness for stone/metal
- **BoxGeometry** for walls, floors, ceilings, doors; **SphereGeometry** or small shapes for items

### Maze/Dungeon Generation
- Define a 2D grid where each cell is a room or corridor
- Walls placed on cell edges based on the grid
- Door cells are special walls that disappear when the player has the matching key
- Floor and ceiling per cell for enclosed dungeon feel

### Player Movement
- Grid-aligned movement (one cell at a time) or free movement with wall collision
- Wall collision: check if the target position overlaps a wall cell
- Smooth movement: lerp between current position and target position

### Camera Options
- **Third-person**: camera offset behind and above player, rotating with player facing
- **First-person**: camera at player eye height, rotating with facing direction
- **Top-down**: fixed overhead view showing the maze layout

### Lighting for Atmosphere
- Low ambient light for dark dungeon feel
- PointLights at specific locations (torches, item glow)
- Player carries a light (attached PointLight) for visibility
- Fog to limit visibility and create tension

## Complete Example: Dungeon Explorer 3D

A 3D dungeon crawler. Navigate a procedurally arranged dungeon, collect keys to open doors, gather treasure, and find the exit. Third-person camera follows the player through corridors.

```javascript
// Dungeon Explorer 3D - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let player, playerLight;
  let walls, items, doors;
  let score, gameOver, won, keysHeld, totalTreasure, treasureCollected;
  let playerGX, playerGZ, playerFacing;
  let targetPos, isMoving, moveProgress;
  let animFrameId, running, paused;
  let dungeonMap;
  let exitPos;

  const CELL = 3;
  const MAP_W = 9, MAP_H = 9;
  const WALL = 1, FLOOR = 0, DOOR_R = 2, DOOR_B = 3, KEY_R = 4, KEY_B = 5, TREASURE = 6, EXIT = 7, PLAYER_START = 8;
  const _camTarget = new THREE.Vector3();
  const FACING = [
    { dx: 0, dz: -1 }, // 0: north
    { dx: 1, dz: 0 },  // 1: east
    { dx: 0, dz: 1 },  // 2: south
    { dx: -1, dz: 0 }  // 3: west
  ];

  function generateDungeon() {
    // Hand-crafted dungeon layout for reliable gameplay
    // 0=floor, 1=wall, 2=red door, 3=blue door,
    // 4=red key, 5=blue key, 6=treasure, 7=exit, 8=player start
    return [
      [1,1,1,1,1,1,1,1,1],
      [1,8,0,0,1,6,0,6,1],
      [1,0,1,0,1,0,1,0,1],
      [1,0,1,0,2,0,1,4,1],
      [1,0,0,0,1,0,0,0,1],
      [1,1,3,1,1,0,1,1,1],
      [1,6,0,0,0,0,0,6,1],
      [1,0,1,6,1,5,1,7,1],
      [1,1,1,1,1,1,1,1,1]
    ];
  }

  function buildDungeon() {
    walls = [];
    items = [];
    doors = [];
    let startX = 1, startZ = 1;

    const floorMat = new THREE.MeshStandardMaterial({ color: 0x443322, roughness: 0.9 });
    const ceilMat = new THREE.MeshStandardMaterial({ color: 0x332211, roughness: 0.9 });
    const wallMat = new THREE.MeshStandardMaterial({ color: 0x555544, roughness: 0.85, metalness: 0.05 });

    for (let z = 0; z < MAP_H; z++) {
      for (let x = 0; x < MAP_W; x++) {
        const cell = dungeonMap[z][x];
        const wx = (x - MAP_W / 2) * CELL;
        const wz = (z - MAP_H / 2) * CELL;

        if (cell === WALL) {
          const wall = new THREE.Mesh(
            new THREE.BoxGeometry(CELL, CELL, CELL),
            wallMat
          );
          wall.position.set(wx, CELL / 2, wz);
          scene.add(wall);
          walls.push({ gx: x, gz: z, mesh: wall });
        } else {
          // Floor
          const floor = new THREE.Mesh(
            new THREE.BoxGeometry(CELL, 0.1, CELL),
            floorMat
          );
          floor.position.set(wx, 0, wz);
          scene.add(floor);
          // Ceiling
          const ceiling = new THREE.Mesh(
            new THREE.BoxGeometry(CELL, 0.1, CELL),
            ceilMat
          );
          ceiling.position.set(wx, CELL, wz);
          scene.add(ceiling);

          if (cell === PLAYER_START) {
            startX = x; startZ = z;
          } else if (cell === DOOR_R || cell === DOOR_B) {
            const color = cell === DOOR_R ? 0xcc2222 : 0x2244cc;
            const doorMesh = new THREE.Mesh(
              new THREE.BoxGeometry(CELL * 0.9, CELL * 0.9, 0.3),
              new THREE.MeshStandardMaterial({ color, metalness: 0.5, roughness: 0.3, transparent: true, opacity: 0.85 })
            );
            doorMesh.position.set(wx, CELL / 2, wz);
            scene.add(doorMesh);
            doors.push({ gx: x, gz: z, type: cell === DOOR_R ? 'red' : 'blue', mesh: doorMesh, open: false });
          } else if (cell === KEY_R || cell === KEY_B) {
            const color = cell === KEY_R ? 0xff4444 : 0x4488ff;
            const keyMesh = new THREE.Mesh(
              new THREE.OctahedronGeometry(0.3, 0),
              new THREE.MeshStandardMaterial({ color, emissive: new THREE.Color(color).multiplyScalar(0.4), metalness: 0.7, roughness: 0.2 })
            );
            keyMesh.position.set(wx, 1.0, wz);
            const keyLight = new THREE.PointLight(color, 0.5, 4);
            keyLight.position.set(wx, 1.2, wz);
            scene.add(keyLight);
            scene.add(keyMesh);
            items.push({ gx: x, gz: z, type: cell === KEY_R ? 'red_key' : 'blue_key', mesh: keyMesh, light: keyLight, collected: false });
          } else if (cell === TREASURE) {
            const tMesh = new THREE.Mesh(
              new THREE.DodecahedronGeometry(0.25, 0),
              new THREE.MeshStandardMaterial({ color: 0xffcc00, emissive: 0x665500, metalness: 0.8, roughness: 0.1 })
            );
            tMesh.position.set(wx, 0.8, wz);
            const tLight = new THREE.PointLight(0xffcc00, 0.3, 3);
            tLight.position.set(wx, 1.0, wz);
            scene.add(tLight);
            scene.add(tMesh);
            items.push({ gx: x, gz: z, type: 'treasure', mesh: tMesh, light: tLight, collected: false });
            totalTreasure++;
          } else if (cell === EXIT) {
            exitPos = { gx: x, gz: z };
            const exitMesh = new THREE.Mesh(
              new THREE.BoxGeometry(CELL * 0.8, CELL * 0.8, 0.2),
              new THREE.MeshStandardMaterial({ color: 0x00ff88, emissive: 0x00aa44, metalness: 0.3, roughness: 0.4, transparent: true, opacity: 0.6 })
            );
            exitMesh.position.set(wx, CELL / 2, wz);
            scene.add(exitMesh);
            const exitLight = new THREE.PointLight(0x00ff88, 1, 6);
            exitLight.position.set(wx, 1.5, wz);
            scene.add(exitLight);
          }

          // Occasional torch
          if (cell === FLOOR && Math.random() < 0.1) {
            const torch = new THREE.PointLight(0xff8844, 0.6, 5);
            torch.position.set(wx, 2.2, wz);
            scene.add(torch);
          }
        }
      }
    }
    return { startX, startZ };
  }

  function createPlayer(gx, gz) {
    const group = new THREE.Group();
    const body = new THREE.Mesh(
      new THREE.CapsuleGeometry(0.3, 0.6, 4, 8),
      new THREE.MeshStandardMaterial({ color: 0x44aaff, metalness: 0.4, roughness: 0.5 })
    );
    body.position.y = 0.9;
    group.add(body);
    const wx = (gx - MAP_W / 2) * CELL;
    const wz = (gz - MAP_H / 2) * CELL;
    group.position.set(wx, 0, wz);
    // Player-carried light
    playerLight = new THREE.PointLight(0xffddaa, 1.2, 10);
    playerLight.position.set(0, 2, 0);
    group.add(playerLight);
    return group;
  }

  function canMoveTo(gx, gz) {
    if (gx < 0 || gx >= MAP_W || gz < 0 || gz >= MAP_H) return false;
    const cell = dungeonMap[gz][gx];
    if (cell === WALL) return false;
    // Check doors
    for (const door of doors) {
      if (door.gx === gx && door.gz === gz && !door.open) {
        if (door.type === 'red' && keysHeld.red > 0) {
          keysHeld.red--;
          door.open = true;
          scene.remove(door.mesh);
          return true;
        } else if (door.type === 'blue' && keysHeld.blue > 0) {
          keysHeld.blue--;
          door.open = true;
          scene.remove(door.mesh);
          return true;
        }
        return false;
      }
    }
    return true;
  }

  function collectItems() {
    for (const item of items) {
      if (item.collected) continue;
      if (item.gx === playerGX && item.gz === playerGZ) {
        item.collected = true;
        scene.remove(item.mesh);
        if (item.light) scene.remove(item.light);
        if (item.type === 'red_key') {
          keysHeld.red++;
        } else if (item.type === 'blue_key') {
          keysHeld.blue++;
        } else if (item.type === 'treasure') {
          treasureCollected++;
          score += 100;
        }
      }
    }
    // Check exit
    if (exitPos && playerGX === exitPos.gx && playerGZ === exitPos.gz) {
      won = true;
      gameOver = true;
      score += 500 + treasureCollected * 50;
    }
  }

  function updateMovement(dt) {
    if (!isMoving) return;
    moveProgress += dt * 4;
    if (moveProgress >= 1) {
      moveProgress = 1;
      isMoving = false;
    }
    const sx = player.position.x;
    const sz = player.position.z;
    const tx = targetPos.x;
    const tz = targetPos.z;
    player.position.x = sx + (tx - sx) * Math.min(moveProgress * 1.5, 1);
    player.position.z = sz + (tz - sz) * Math.min(moveProgress * 1.5, 1);
    if (!isMoving) {
      player.position.x = tx;
      player.position.z = tz;
      collectItems();
    }
  }

  function updateCamera() {
    const facingDir = FACING[playerFacing];
    const camDist = 5;
    const camHeight = 4;
    _camTarget.set(
      player.position.x - facingDir.dx * camDist,
      camHeight,
      player.position.z - facingDir.dz * camDist
    );
    camera.position.lerp(_camTarget, 0.08);
    camera.lookAt(
      player.position.x + facingDir.dx * 2,
      1.5,
      player.position.z + facingDir.dz * 2
    );
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    if (running && !paused) {
      const dt = Math.min(clock.getDelta(), 0.05);
      if (!gameOver) {
        updateMovement(dt);
      }
      updateCamera();
      // Animate uncollected items
      const t = Date.now() * 0.003;
      for (const item of items) {
        if (!item.collected) {
          item.mesh.rotation.y = t;
          item.mesh.position.y = 0.8 + Math.sin(t * 2 + item.gx) * 0.1;
        }
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
      scene.background = new THREE.Color(0x0a0a0a);
      scene.fog = new THREE.Fog(0x0a0a0a, 3, 18);
      camera = new THREE.PerspectiveCamera(60, canvas.clientWidth / canvas.clientHeight, 0.1, 50);
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
      // Very dim ambient for dungeon feel
      scene.add(new THREE.AmbientLight(0x222222, 0.3));

      score = 0; gameOver = false; won = false;
      keysHeld = { red: 0, blue: 0 };
      totalTreasure = 0; treasureCollected = 0;
      playerFacing = 0;
      isMoving = false; moveProgress = 0;

      dungeonMap = generateDungeon();
      const start = buildDungeon();
      playerGX = start.startX;
      playerGZ = start.startZ;
      player = createPlayer(playerGX, playerGZ);
      scene.add(player);

      targetPos = new THREE.Vector3(player.position.x, 0, player.position.z);
      camera.position.set(player.position.x, 4, player.position.z + 5);
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
      scene.add(new THREE.AmbientLight(0x222222, 0.3));
      score = 0; gameOver = false; won = false;
      keysHeld = { red: 0, blue: 0 };
      totalTreasure = 0; treasureCollected = 0;
      playerFacing = 0;
      isMoving = false; moveProgress = 0;
      dungeonMap = generateDungeon();
      const start = buildDungeon();
      playerGX = start.startX;
      playerGZ = start.startZ;
      player = createPlayer(playerGX, playerGZ);
      scene.add(player);
      targetPos = new THREE.Vector3(player.position.x, 0, player.position.z);
      camera.position.set(player.position.x, 4, player.position.z + 5);
    },

    getState() {
      return {
        score,
        gameOver,
        won,
        playerPos: { gx: playerGX, gz: playerGZ },
        playerFacing,
        keysHeld: keysHeld ? { red: keysHeld.red, blue: keysHeld.blue } : { red: 0, blue: 0 },
        treasureCollected,
        totalTreasure,
        isMoving,
        doors: doors ? doors.filter(d => !d.open).map(d => ({
          gx: d.gx, gz: d.gz, type: d.type
        })) : [],
        itemsRemaining: items ? items.filter(i => !i.collected).map(i => ({
          gx: i.gx, gz: i.gz, type: i.type
        })) : [],
        exitPos: exitPos || null,
        mapWidth: MAP_W,
        mapHeight: MAP_H
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver || isMoving) return false;
      switch (action) {
        case 'up': {
          const dir = FACING[playerFacing];
          const nx = playerGX + dir.dx;
          const nz = playerGZ + dir.dz;
          if (canMoveTo(nx, nz)) {
            playerGX = nx;
            playerGZ = nz;
            targetPos = new THREE.Vector3(
              (nx - MAP_W / 2) * CELL, 0, (nz - MAP_H / 2) * CELL
            );
            isMoving = true;
            moveProgress = 0;
          }
          return true;
        }
        case 'down': {
          // Move backward
          const backIdx = (playerFacing + 2) % 4;
          const dir = FACING[backIdx];
          const nx = playerGX + dir.dx;
          const nz = playerGZ + dir.dz;
          if (canMoveTo(nx, nz)) {
            playerGX = nx;
            playerGZ = nz;
            targetPos = new THREE.Vector3(
              (nx - MAP_W / 2) * CELL, 0, (nz - MAP_H / 2) * CELL
            );
            isMoving = true;
            moveProgress = 0;
          }
          return true;
        }
        case 'left':
          playerFacing = (playerFacing + 3) % 4;
          return true;
        case 'right':
          playerFacing = (playerFacing + 1) % 4;
          return true;
        case 'action':
          // Interaction: try to open adjacent door or collect
          collectItems();
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
        name: 'Dungeon Explorer 3D',
        description: 'Explore a dark dungeon, collect keys and treasure, unlock doors, and find the exit!',
        controls: {
          up: 'Move forward in facing direction',
          down: 'Move backward',
          left: 'Turn left',
          right: 'Turn right',
          action: 'Interact with nearby items'
        },
        objective: 'Find the green exit portal. Collect colored keys to unlock matching doors. Gather treasure for bonus score.',
        scoring: '100 points per treasure. 500 points for reaching the exit. 50 bonus per treasure when exiting.',
        tips: [
          'Red keys open red doors, blue keys open blue doors',
          'Explore all corridors to find keys before heading to locked areas',
          'Items glow and rotate so they are easy to spot in the dark',
          'The exit glows green and is visible from a distance',
          'Your character carries a light so nearby walls are always visible',
          'Use getState() to see the full item and door map for pathfinding'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For adventure 3D games, `getState()` should expose:
- `score` (number) - accumulated score from treasure and completion
- `gameOver` (boolean) - whether the game ended
- `won` (boolean) - whether the player reached the exit (vs. just giving up)
- `playerPos` (object with gx, gz) - grid position in the dungeon
- `playerFacing` (number 0-3) - cardinal direction the player faces
- `keysHeld` (object with red, blue counts) - inventory state
- `treasureCollected` / `totalTreasure` - collection progress
- `isMoving` (boolean) - whether the player is mid-transition
- `doors` (array) - remaining locked doors with position and color
- `itemsRemaining` (array) - uncollected items with position and type
- `exitPos` (object with gx, gz) - location of the exit
- `mapWidth` / `mapHeight` - grid dimensions

This comprehensive state enables AI agents to build internal maps, plan routes to keys, and navigate efficiently.

## getMeta() Design

- `controls` map up/down to forward/backward movement, left/right to turning
- `action` triggers interaction with items and doors
- `tips` explain the key/door color-matching mechanic and exploration strategy

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "adventure"
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
- [ ] Dungeon built from grid with BoxGeometry walls, floors, ceilings
- [ ] Key/door mechanics: colored keys unlock matching doors
- [ ] Collectible items with visual feedback (glow, rotation)
- [ ] Third-person camera follows player with lerp
- [ ] Fog and low ambient light create dungeon atmosphere
- [ ] Player carries a PointLight for local illumination
- [ ] Exit detection triggers win condition with score bonus
