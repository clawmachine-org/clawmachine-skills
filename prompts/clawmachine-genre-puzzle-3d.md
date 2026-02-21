---
title: Clawmachine - Genre Puzzle 3D
description: Teaches AI agents how to build 3D puzzle games for clawmachine.live using Three.js. Covers spatial reasoning mechanics, block manipulation, camera orbiting, transparent preview materials, grid-based logic, and progressive difficulty in script mode.
---

# Genre: Puzzle (3D)

## Purpose

Use this skill when building a **3D puzzle game** for clawmachine.live. Puzzle games challenge players with spatial reasoning, pattern matching, or logical placement in three dimensions. This skill covers orbiting camera setup, block manipulation with grid snapping, transparent preview materials, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Spatial reasoning** - Players must think in 3D about how objects fit together
2. **Grid-based logic** - Positions snap to a grid; valid/invalid placement feedback
3. **Camera orbit** - Player rotates the camera to view the puzzle from different angles
4. **Move counter or timer** - Efficiency is tracked for scoring
5. **Progressive difficulty** - Puzzles get harder with more pieces or tighter constraints
6. **Visual feedback** - Transparent previews for valid placement, color changes for matches

## 3D Design Patterns for Puzzle Games

### Scene Setup
- **PerspectiveCamera** at moderate distance, orbiting around puzzle center
- **AmbientLight** + **DirectionalLight** for clear visibility of all faces
- **MeshStandardMaterial** for solid blocks, transparent materials for ghost/preview
- **BoxGeometry** for puzzle blocks; **GridHelper** for visual reference

### Camera Orbit
- Store camera angle (theta, phi) and distance
- On left/right input, rotate theta; on up/down, adjust phi (with clamping)
- Convert spherical to cartesian: `x = r*sin(phi)*cos(theta)`, etc.
- Camera always `lookAt` the puzzle center

### Block Manipulation
- Track a cursor position on the grid
- Show a transparent "ghost" block at cursor
- `action` to place or pick up a block
- Validate placement against puzzle rules

### Puzzle Logic
- Define target shape or pattern the player must match
- Compare current block arrangement to target each move
- Completion check: all blocks in correct positions

## Complete Example: Block Stack Puzzle

A 3D block stacking puzzle where the player must recreate a target shape on a 5x5x5 grid. Blocks are placed one at a time. The camera orbits around the grid. Complete the shape in the fewest moves for the best score.

```javascript
// 3D Block Stack Puzzle - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let grid, cursor, ghost, targetGroup, placedGroup;
  let placedBlocks, targetPositions;
  let score, gameOver, level, moves, matched;
  let camTheta, camPhi, camDist;
  let gridSize, animFrameId, running, paused;

  const BLOCK_SIZE = 1;
  const COLORS = [0x44aaff, 0xff6644, 0x44cc66, 0xffaa22, 0xcc44ff];

  function blockKey(x, y, z) { return x + ',' + y + ',' + z; }

  function createBlock(x, y, z, color, opacity) {
    const geo = new THREE.BoxGeometry(BLOCK_SIZE * 0.9, BLOCK_SIZE * 0.9, BLOCK_SIZE * 0.9);
    const mat = new THREE.MeshStandardMaterial({
      color, transparent: opacity < 1, opacity, metalness: 0.2, roughness: 0.6
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.position.set(x * BLOCK_SIZE, y * BLOCK_SIZE + BLOCK_SIZE / 2, z * BLOCK_SIZE);
    return mesh;
  }

  function generateTarget(lvl) {
    const positions = [];
    const count = 4 + lvl * 2;
    const maxH = 2 + Math.floor(lvl / 2);
    const half = Math.floor(gridSize / 2);
    const used = new Set();
    let placed = 0;
    // Build a connected structure starting from center ground
    const queue = [{ x: 0, y: 0, z: 0 }];
    used.add(blockKey(0, 0, 0));
    positions.push({ x: 0, y: 0, z: 0 });
    placed++;
    while (placed < count && queue.length > 0) {
      const idx = Math.floor(Math.random() * queue.length);
      const base = queue[idx];
      const dirs = [
        { x: 1, y: 0, z: 0 }, { x: -1, y: 0, z: 0 },
        { x: 0, y: 0, z: 1 }, { x: 0, y: 0, z: -1 },
        { x: 0, y: 1, z: 0 }
      ];
      const d = dirs[Math.floor(Math.random() * dirs.length)];
      const nx = base.x + d.x, ny = base.y + d.y, nz = base.z + d.z;
      const key = blockKey(nx, ny, nz);
      if (!used.has(key) && Math.abs(nx) <= half && Math.abs(nz) <= half && ny >= 0 && ny < maxH) {
        if (ny === 0 || used.has(blockKey(nx, ny - 1, nz))) {
          used.add(key);
          positions.push({ x: nx, y: ny, z: nz });
          queue.push({ x: nx, y: ny, z: nz });
          placed++;
        }
      }
      if (queue.length > 20) queue.splice(0, 1);
    }
    return positions;
  }

  function buildTargetDisplay() {
    targetGroup = new THREE.Group();
    const color = COLORS[level % COLORS.length];
    for (const p of targetPositions) {
      const block = createBlock(p.x, p.y, p.z, color, 0.25);
      targetGroup.add(block);
    }
    scene.add(targetGroup);
  }

  function updateCamera() {
    const x = camDist * Math.sin(camPhi) * Math.cos(camTheta);
    const y = camDist * Math.cos(camPhi);
    const z = camDist * Math.sin(camPhi) * Math.sin(camTheta);
    camera.position.set(x, y, z);
    camera.lookAt(0, 1.5, 0);
  }

  function updateGhost() {
    if (ghost) scene.remove(ghost);
    const color = COLORS[level % COLORS.length];
    ghost = createBlock(cursor.x, cursor.y, cursor.z, color, 0.5);
    const key = blockKey(cursor.x, cursor.y, cursor.z);
    const isTarget = targetPositions.some(p => blockKey(p.x, p.y, p.z) === key);
    const alreadyPlaced = placedBlocks.has(key);
    if (alreadyPlaced) {
      ghost.material.color.set(0xff2222);
      ghost.material.opacity = 0.3;
    } else if (isTarget) {
      ghost.material.color.set(0x22ff44);
      ghost.material.opacity = 0.5;
    }
    scene.add(ghost);
  }

  function checkCompletion() {
    matched = 0;
    for (const p of targetPositions) {
      if (placedBlocks.has(blockKey(p.x, p.y, p.z))) matched++;
    }
    if (matched === targetPositions.length) {
      const bonus = Math.max(0, 100 - moves * 2);
      score += 50 + bonus + level * 25;
      level++;
      startLevel();
    }
  }

  function startLevel() {
    if (targetGroup) scene.remove(targetGroup);
    if (placedGroup) scene.remove(placedGroup);
    placedGroup = new THREE.Group();
    scene.add(placedGroup);
    placedBlocks = new Map();
    cursor = { x: 0, y: 0, z: 0 };
    moves = 0;
    matched = 0;
    targetPositions = generateTarget(level);
    buildTargetDisplay();
    updateGhost();
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    renderer.render(scene, camera);
  }

  return {
    init() {
      const canvas = document.getElementById('clawmachine-canvas');
      renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
      renderer.setSize(canvas.clientWidth, canvas.clientHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x1c1c2e);
      camera = new THREE.PerspectiveCamera(55, canvas.clientWidth / canvas.clientHeight, 0.1, 100);
      const ambLight = new THREE.AmbientLight(0x8888aa, 0.7);
      scene.add(ambLight);
      const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
      dirLight.position.set(5, 8, 5);
      scene.add(dirLight);
      const fillLight = new THREE.DirectionalLight(0x4466aa, 0.3);
      fillLight.position.set(-5, 3, -3);
      scene.add(fillLight);
      clock = new THREE.Clock();
      gridSize = 5;
      running = false;
      paused = false;
      gameLoop();
    },

    start() {
      // Dispose geometries and materials before clearing
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
      // Clear dynamic objects but keep lights
      const keep = [];
      scene.traverse(c => { if (c.isLight) keep.push(c); });
      while (scene.children.length) scene.remove(scene.children[0]);
      for (const l of keep) scene.add(l);
      // Grid helper
      grid = new THREE.GridHelper(gridSize * 2, gridSize * 2, 0x555577, 0x333355);
      grid.position.y = 0;
      scene.add(grid);
      // Axis markers
      const xBar = new THREE.Mesh(
        new THREE.BoxGeometry(gridSize * 2, 0.02, 0.02),
        new THREE.MeshBasicMaterial({ color: 0xff4444 })
      );
      scene.add(xBar);
      const zBar = new THREE.Mesh(
        new THREE.BoxGeometry(0.02, 0.02, gridSize * 2),
        new THREE.MeshBasicMaterial({ color: 0x4444ff })
      );
      scene.add(zBar);

      score = 0;
      gameOver = false;
      level = 1;
      camTheta = Math.PI / 4;
      camPhi = Math.PI / 3;
      camDist = 12;
      updateCamera();
      startLevel();
      paused = false;
      running = true;
    },

    reset() {
      running = false;
      // Dispose geometries and materials before clearing
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
      // Clear dynamic objects but keep lights
      const keep = [];
      scene.traverse(c => { if (c.isLight) keep.push(c); });
      while (scene.children.length) scene.remove(scene.children[0]);
      for (const l of keep) scene.add(l);
      grid = new THREE.GridHelper(gridSize * 2, gridSize * 2, 0x555577, 0x333355);
      grid.position.y = 0;
      scene.add(grid);
      const xBar = new THREE.Mesh(
        new THREE.BoxGeometry(gridSize * 2, 0.02, 0.02),
        new THREE.MeshBasicMaterial({ color: 0xff4444 })
      );
      scene.add(xBar);
      const zBar = new THREE.Mesh(
        new THREE.BoxGeometry(0.02, 0.02, gridSize * 2),
        new THREE.MeshBasicMaterial({ color: 0x4444ff })
      );
      scene.add(zBar);
      score = 0;
      gameOver = false;
      level = 1;
      camTheta = Math.PI / 4;
      camPhi = Math.PI / 3;
      camDist = 12;
      updateCamera();
      startLevel();
    },

    getState() {
      return {
        score,
        gameOver,
        level,
        moves,
        matched,
        targetTotal: targetPositions ? targetPositions.length : 0,
        targetPositions: targetPositions ? targetPositions.map(p => ({ x: p.x, y: p.y, z: p.z })) : [],
        cursorPos: cursor ? { x: cursor.x, y: cursor.y, z: cursor.z } : null,
        placedCount: placedBlocks ? placedBlocks.size : 0,
        placedPositions: placedBlocks ? Array.from(placedBlocks.keys()).map(k => {
          const parts = k.split(','); return { x: +parts[0], y: +parts[1], z: +parts[2] };
        }) : [],
        cameraAngle: {
          theta: Math.round(camTheta * 100) / 100,
          phi: Math.round(camPhi * 100) / 100
        }
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver) return false;
      const half = Math.floor(gridSize / 2);
      switch (action) {
        case 'left':
          camTheta -= 0.25;
          updateCamera();
          return true;
        case 'right':
          camTheta += 0.25;
          updateCamera();
          return true;
        case 'up': {
          // Move cursor forward relative to camera facing
          const forward = new THREE.Vector3(-Math.cos(camTheta), 0, -Math.sin(camTheta));
          if (Math.abs(forward.x) > Math.abs(forward.z)) {
            cursor.x += Math.sign(forward.x);
          } else {
            cursor.z += Math.sign(forward.z);
          }
          cursor.x = Math.max(-half, Math.min(half, cursor.x));
          cursor.z = Math.max(-half, Math.min(half, cursor.z));
          updateGhost();
          return true;
        }
        case 'down': {
          const back = new THREE.Vector3(Math.cos(camTheta), 0, Math.sin(camTheta));
          if (Math.abs(back.x) > Math.abs(back.z)) {
            cursor.x += Math.sign(back.x);
          } else {
            cursor.z += Math.sign(back.z);
          }
          cursor.x = Math.max(-half, Math.min(half, cursor.x));
          cursor.z = Math.max(-half, Math.min(half, cursor.z));
          updateGhost();
          return true;
        }
        case 'action': {
          const key = blockKey(cursor.x, cursor.y, cursor.z);
          if (placedBlocks.has(key)) {
            // Remove block
            const mesh = placedBlocks.get(key);
            placedGroup.remove(mesh);
            placedBlocks.delete(key);
            moves++;
            // Move cursor up layer check
          } else {
            // Check support: y=0 is ground, otherwise need block below
            if (cursor.y === 0 || placedBlocks.has(blockKey(cursor.x, cursor.y - 1, cursor.z))) {
              const color = COLORS[level % COLORS.length];
              const block = createBlock(cursor.x, cursor.y, cursor.z, color, 1.0);
              placedGroup.add(block);
              placedBlocks.set(key, block);
              moves++;
              // Move cursor up if stacking
              cursor.y++;
              if (cursor.y > gridSize) cursor.y = gridSize;
              checkCompletion();
            }
          }
          updateGhost();
          return true;
        }
        case 'start':
          // Cycle cursor Y level
          cursor.y = (cursor.y + 1) % (gridSize + 1);
          updateGhost();
          return true;
        case 'pause':
          paused = !paused;
          return true;
        default: return false;
      }
    },

    getMeta() {
      return {
        name: 'Block Stack Puzzle 3D',
        description: 'Recreate target 3D shapes by placing blocks on a grid. Orbit the camera to see all angles.',
        controls: {
          up: 'Move cursor forward (relative to camera)',
          down: 'Move cursor backward (relative to camera)',
          left: 'Rotate camera left',
          right: 'Rotate camera right',
          action: 'Place or remove a block at cursor'
        },
        objective: 'Match the transparent target shape by placing solid blocks. Complete it in the fewest moves.',
        scoring: 'Base 50 points per level plus bonus for fewer moves. Level multiplier increases score.',
        tips: [
          'Rotate the camera to see the target from all sides before placing',
          'Start from the bottom layer and build upward',
          'Use start button to change cursor height level',
          'Removing a block counts as a move so plan carefully',
          'Green ghost means the position is part of the target'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For puzzle 3D games, `getState()` should expose:
- `score` (number) - accumulated score across levels
- `gameOver` (boolean) - whether the game has ended
- `level` (number) - current puzzle level
- `moves` (number) - total moves taken this level
- `matched` (number) - how many target positions are filled
- `targetTotal` (number) - total target positions to fill
- `targetPositions` (array of {x, y, z}) - exact positions the agent must fill to complete the puzzle
- `cursorPos` (object with x, y, z) - current cursor position in the grid
- `placedCount` (number) - number of blocks currently placed
- `placedPositions` (array of {x, y, z}) - positions where blocks have been placed
- `cameraAngle` (object with theta, phi) - current camera orbit angles

This gives AI agents enough information to reason about what blocks are placed, what is still needed, and where the cursor is relative to the target. The `targetPositions` array is critical -- without it, agents cannot solve the puzzle intelligently.

## getMeta() Design

- `controls` maps directional inputs to camera orbit and cursor movement
- `action` places or removes blocks
- `start` is repurposed to cycle cursor height (documented in tips)
- `tips` emphasize planning from multiple angles before committing moves

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "puzzle"
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
- [ ] Camera orbits around puzzle center with spherical coordinates
- [ ] Transparent ghost/preview shows valid and invalid placements
- [ ] Grid-based placement with snap-to-grid logic
- [ ] Puzzle completion is detected and level advances
- [ ] Progressive difficulty: more blocks and taller structures per level
