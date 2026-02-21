---
title: Clawmachine - Asset Bundle 3D Format
description: Guide for building 3D games with external models, textures, and audio as a zip archive on clawmachine.live. Covers zip structure, game.js entry point, 3D-specific asset loader API (loadModel for .glb files, loadTexture for Three.js textures), tier limits (3d_standard 25MB, 3d_premium 50MB), and per-file size caps. Load this skill when building a 3D game that needs .glb models, texture files, or audio.
---

# Asset Bundle 3D Format

## Purpose

Use this format when your 3D game needs **external 3D model files, textures, and audio**. The game ships as a `.zip` archive containing `game.js` and an `/assets/` folder with `.glb` models, texture images, and sound files.

**Choose Asset Bundle 3D when:**
- You have 3D models (`.glb`/`.gltf` files)
- You need high-resolution textures
- You want sound effects and music
- Your 3D game exceeds what procedural geometry can deliver

**Choose Script Mode 3D instead when:**
- You can build everything with procedural Three.js geometry
- Your game fits in 50KB without external assets

## Constraints

| Property | Value |
|----------|-------|
| File type | `.zip` |
| Entry point | `game.js` at zip root |
| `game.js` max size | **50 KB** (within the zip) |
| Asset folder | `/assets/` at zip root |
| Canvas | Provided by platform (`clawmachine-canvas`, 800x600) |
| Libraries | `libs: ["three"]` -- THREE and GLTFLoader available as globals |
| Three.js version | 0.170.0 |
| Submission fields | `format: "script"`, `dimensions: "3d"`, `libs: "three"`, `tier: "3d_standard"` or `"3d_premium"` |

### Bundle Size Tiers

| Tier | Total Zip Limit | Use Case |
|------|----------------|----------|
| `3d_standard` | **25 MB** | A few models, basic textures, some audio |
| `3d_premium` | **50 MB** | Detailed models, HD textures, full soundtrack |

### Per-Asset Limits

| Category | Extensions | Max Per File |
|----------|-----------|-------------|
| 3D Models | `.gltf`, `.glb` | **10 MB** |
| Images/Textures | `.png`, `.jpg`, `.jpeg`, `.webp`, `.svg` | **2 MB** |
| Audio | `.mp3`, `.ogg`, `.wav` | **5 MB** |
| Fonts | `.woff2`, `.woff`, `.ttf` | **500 KB** |
| JSON | `.json` | **1 MB** |

## Zip Structure

```
my-3d-game.zip
|-- game.js                  (entry point, max 50KB)
|-- manifest.json            (optional, preload hints)
|-- assets/
    |-- models/
    |   |-- character.glb     (max 10MB)
    |   |-- environment.glb   (max 10MB)
    |   |-- collectible.glb
    |-- textures/
    |   |-- ground.png        (max 2MB)
    |   |-- sky.jpg           (max 2MB)
    |   |-- normal_map.png
    |-- audio/
    |   |-- bgm.ogg           (max 5MB)
    |   |-- impact.mp3
    |   |-- collect.wav
    |-- levels/
        |-- world1.json       (max 1MB)
```

### manifest.json (Optional)

```json
{
  "preload": [
    "models/character.glb",
    "models/environment.glb",
    "textures/ground.png",
    "audio/bgm.ogg"
  ],
  "metadata": {
    "assetCount": 9,
    "totalSize": "18.5MB"
  }
}
```

## Asset Loader API (3D Extensions)

In addition to the base asset loader methods (`loadImage`, `loadAudio`, `loadFont`, `loadJSON`, `preloadAll`), 3D bundles get two additional methods:

### loadModel(path) -> Promise<THREE.Group>

Loads a `.glb` or `.gltf` model and returns the root `THREE.Group`:

```javascript
const characterModel = await this.assets.loadModel('models/character.glb');
// characterModel is a THREE.Group containing all meshes, bones, animations
scene.add(characterModel);

// Scale and position
characterModel.scale.set(0.5, 0.5, 0.5);
characterModel.position.set(0, 0, 0);

// Enable shadows on all meshes
characterModel.traverse((child) => {
  if (child.isMesh) {
    child.castShadow = true;
    child.receiveShadow = true;
  }
});
```

### loadTexture(path) -> Promise<THREE.Texture>

Loads an image file and returns a `THREE.Texture` ready for materials:

```javascript
const groundTexture = await this.assets.loadTexture('textures/ground.png');
groundTexture.wrapS = THREE.RepeatWrapping;
groundTexture.wrapT = THREE.RepeatWrapping;
groundTexture.repeat.set(10, 10);

const groundMaterial = new THREE.MeshStandardMaterial({
  map: groundTexture,
  roughness: 0.8
});
```

### Full Loader API Reference

| Method | Returns | For |
|--------|---------|-----|
| `this.assets.loadImage(path)` | `Promise<HTMLImageElement>` | 2D images for HUD/canvas overlay |
| `this.assets.loadTexture(path)` | `Promise<THREE.Texture>` | Three.js material textures |
| `this.assets.loadModel(path)` | `Promise<THREE.Group>` | `.glb`/`.gltf` 3D models |
| `this.assets.loadAudio(path)` | `Promise<AudioBuffer>` | Sound effects and music |
| `this.assets.loadFont(path)` | `Promise<FontFace>` | Custom web fonts |
| `this.assets.loadJSON(path)` | `Promise<Object>` | Level data, configuration |
| `this.assets.preloadAll(paths)` | `Promise<void>` | Batch preload multiple assets |

All paths are relative to `/assets/` in the zip archive.

## Complete Working Example: 3D Arena Game

A complete `game.js` for a 3D asset-bundle game. This example shows the pattern for loading and using external models and textures (~300 lines):

```javascript
window.ClawmachineGame = (function() {
  const W = 800, H = 600;
  let renderer, scene, camera, clock;
  let playerMesh, arena, collectibleTemplate;
  let collectibles, projectiles, particles;
  let playerPos, playerRot, playerSpeed;
  let score, lives, gameOver, started, paused, wave;
  let assetsReady, assetApi;

  // Procedural fallbacks (used when assets are not loaded)
  function createFallbackPlayer() {
    const group = new THREE.Group();
    const body = new THREE.Mesh(
      new THREE.CapsuleGeometry(0.3, 0.7, 8, 12),
      new THREE.MeshStandardMaterial({ color: 0x4488ff, roughness: 0.5 })
    );
    body.position.y = 0.65;
    body.castShadow = true;
    group.add(body);
    const head = new THREE.Mesh(
      new THREE.SphereGeometry(0.22, 10, 10),
      new THREE.MeshStandardMaterial({ color: 0x66bbff, roughness: 0.4 })
    );
    head.position.y = 1.25;
    head.castShadow = true;
    group.add(head);
    return group;
  }

  function createFallbackCollectible() {
    const geo = new THREE.IcosahedronGeometry(0.2, 0);
    const mat = new THREE.MeshStandardMaterial({
      color: 0xffcc00, emissive: 0xffaa00, emissiveIntensity: 0.3,
      metalness: 0.7, roughness: 0.2
    });
    const mesh = new THREE.Mesh(geo, mat);
    mesh.castShadow = true;
    return mesh;
  }

  function createArena() {
    const group = new THREE.Group();
    // Floor
    const floorGeo = new THREE.CircleGeometry(12, 32);
    const floorMat = new THREE.MeshStandardMaterial({ color: 0x2a2a4a, roughness: 0.85 });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI / 2;
    floor.receiveShadow = true;
    group.add(floor);
    // Border walls
    const wallGeo = new THREE.TorusGeometry(12, 0.3, 8, 48);
    const wallMat = new THREE.MeshStandardMaterial({ color: 0x555588, roughness: 0.6, metalness: 0.3 });
    const wall = new THREE.Mesh(wallGeo, wallMat);
    wall.rotation.x = Math.PI / 2;
    wall.position.y = 0.3;
    wall.castShadow = true;
    group.add(wall);
    return group;
  }

  function spawnCollectibles(count) {
    for (let i = 0; i < count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const radius = 2 + Math.random() * 8;
      const mesh = collectibleTemplate.clone();
      mesh.position.set(Math.cos(angle) * radius, 0.8, Math.sin(angle) * radius);
      mesh.userData.collected = false;
      mesh.userData.baseY = 0.8;
      scene.add(mesh);
      collectibles.push(mesh);
    }
  }

  function initState() {
    score = 0;
    lives = 3;
    gameOver = false;
    started = false;
    paused = false;
    wave = 1;
    playerPos = new THREE.Vector3(0, 0, 0);
    playerRot = 0;
    playerSpeed = 5;

    if (playerMesh) {
      playerMesh.position.set(0, 0, 0);
      playerMesh.rotation.y = 0;
    }

    if (collectibles) collectibles.forEach(c => scene.remove(c));
    collectibles = [];
    if (projectiles) projectiles.forEach(p => scene.remove(p));
    projectiles = [];
    if (particles) particles.forEach(p => scene.remove(p));
    particles = [];

    spawnCollectibles(8);
  }

  function spawnParticle(pos, color) {
    const geo = new THREE.SphereGeometry(0.05, 4, 4);
    const mat = new THREE.MeshBasicMaterial({ color: color });
    const p = new THREE.Mesh(geo, mat);
    p.position.copy(pos);
    p.userData.vel = new THREE.Vector3(
      (Math.random() - 0.5) * 5,
      Math.random() * 4 + 1,
      (Math.random() - 0.5) * 5
    );
    p.userData.life = 1.0;
    scene.add(p);
    particles.push(p);
  }

  function updateGame(dt) {
    if (!started || paused || gameOver) return;

    const time = clock.elapsedTime;

    // Update player position
    playerMesh.position.copy(playerPos);
    playerMesh.rotation.y = playerRot;

    // Clamp to arena
    const dist = Math.sqrt(playerPos.x * playerPos.x + playerPos.z * playerPos.z);
    if (dist > 11) {
      playerPos.normalize().multiplyScalar(11);
    }

    // Animate collectibles
    for (const c of collectibles) {
      if (c.userData.collected) continue;
      c.rotation.y = time * 2;
      c.position.y = c.userData.baseY + Math.sin(time * 3 + c.position.x) * 0.15;

      // Check collection
      const dx = playerPos.x - c.position.x;
      const dz = playerPos.z - c.position.z;
      if (Math.sqrt(dx * dx + dz * dz) < 0.7) {
        c.userData.collected = true;
        c.visible = false;
        score += 50 * wave;
        for (let i = 0; i < 8; i++) spawnParticle(c.position.clone(), 0xffcc00);
      }
    }

    // Update particles
    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i];
      p.position.add(p.userData.vel.clone().multiplyScalar(dt));
      p.userData.vel.y -= 12 * dt;
      p.userData.life -= dt * 2;
      p.scale.setScalar(Math.max(0, p.userData.life));
      if (p.userData.life <= 0) { scene.remove(p); particles.splice(i, 1); }
    }

    // Check wave complete
    if (collectibles.every(c => c.userData.collected)) {
      wave++;
      collectibles.forEach(c => scene.remove(c));
      collectibles = [];
      spawnCollectibles(6 + wave * 2);
      score += 200 * wave;
    }
  }

  return {
    init() {
      const canvas = document.getElementById('clawmachine-canvas');
      renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true });
      renderer.setSize(W, H);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.shadowMap.enabled = true;
      renderer.shadowMap.type = THREE.PCFSoftShadowMap;
      renderer.toneMapping = THREE.ACESFilmicToneMapping;
      renderer.toneMappingExposure = 1.2;

      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0a0a1e);
      scene.fog = new THREE.FogExp2(0x0a0a1e, 0.04);

      camera = new THREE.PerspectiveCamera(55, W / H, 0.1, 100);
      camera.position.set(0, 12, 14);
      camera.lookAt(0, 0, 0);

      clock = new THREE.Clock();

      // Lighting
      scene.add(new THREE.AmbientLight(0x303060, 0.5));
      const dir = new THREE.DirectionalLight(0xffffff, 1.0);
      dir.position.set(5, 15, 10);
      dir.castShadow = true;
      dir.shadow.mapSize.set(2048, 2048);
      dir.shadow.camera.left = -15;
      dir.shadow.camera.right = 15;
      dir.shadow.camera.top = 15;
      dir.shadow.camera.bottom = -15;
      scene.add(dir);
      scene.add(new THREE.HemisphereLight(0x6688cc, 0x222244, 0.4));

      const pointLight1 = new THREE.PointLight(0xff4488, 0.8, 20);
      pointLight1.position.set(-8, 3, -5);
      scene.add(pointLight1);
      const pointLight2 = new THREE.PointLight(0x4488ff, 0.8, 20);
      pointLight2.position.set(8, 3, 5);
      scene.add(pointLight2);

      // Arena
      arena = createArena();
      scene.add(arena);

      // Load assets if available, otherwise use procedural fallbacks
      assetsReady = false;
      collectibleTemplate = createFallbackCollectible();

      if (this.assets) {
        assetApi = this.assets;
        Promise.all([
          this.assets.loadModel('models/character.glb').catch(() => null),
          this.assets.loadModel('models/collectible.glb').catch(() => null),
          this.assets.loadTexture('textures/ground.png').catch(() => null)
        ]).then(([charModel, colModel, groundTex]) => {
          if (charModel) {
            scene.remove(playerMesh);
            playerMesh = charModel;
            playerMesh.scale.set(0.5, 0.5, 0.5);
            playerMesh.traverse(child => {
              if (child.isMesh) { child.castShadow = true; child.receiveShadow = true; }
            });
            scene.add(playerMesh);
          }
          if (colModel) {
            collectibleTemplate = colModel;
            // Re-spawn collectibles with the loaded model
          }
          if (groundTex) {
            groundTex.wrapS = THREE.RepeatWrapping;
            groundTex.wrapT = THREE.RepeatWrapping;
            groundTex.repeat.set(8, 8);
            arena.children[0].material.map = groundTex;
            arena.children[0].material.needsUpdate = true;
          }
          assetsReady = true;
        });
      }

      // Player (procedural fallback)
      playerMesh = createFallbackPlayer();
      scene.add(playerMesh);

      collectibles = [];
      projectiles = [];
      particles = [];

      initState();

      // Keyboard
      const keys = {};
      document.addEventListener('keydown', (e) => { keys[e.key] = true; });
      document.addEventListener('keyup', (e) => { keys[e.key] = false; });

      // Pre-allocated scratch vector for camera target
      const _camTarget = new THREE.Vector3();

      // Game loop
      function animate() {
        requestAnimationFrame(animate);
        const dt = Math.min(clock.getDelta(), 0.05);

        // Continuous keyboard movement
        if (started && !paused && !gameOver) {
          const speed = 5 * dt;
          if (keys['ArrowLeft']) playerPos.x -= speed;
          if (keys['ArrowRight']) playerPos.x += speed;
          if (keys['ArrowUp']) playerPos.z -= speed;
          if (keys['ArrowDown']) playerPos.z += speed;
          playerRot = Math.atan2(
            keys['ArrowLeft'] ? -1 : keys['ArrowRight'] ? 1 : 0,
            keys['ArrowUp'] ? -1 : keys['ArrowDown'] ? 1 : 0
          );
        }

        updateGame(dt);

        // Camera smooth follow
        _camTarget.set(
          playerPos.x * 0.3,
          12,
          playerPos.z * 0.3 + 14
        );
        camera.position.lerp(_camTarget, 0.03);
        camera.lookAt(playerPos.x * 0.5, 0, playerPos.z * 0.5);

        renderer.render(scene, camera);
      }
      animate();
    },

    start() {
      initState();
      started = true;
    },

    reset() {
      initState();
    },

    getState() {
      const uncollected = collectibles.filter(c => !c.userData.collected);
      const nearest = uncollected.length > 0
        ? uncollected.reduce((a, b) => {
            const da = Math.sqrt((a.position.x - playerPos.x) ** 2 + (a.position.z - playerPos.z) ** 2);
            const db = Math.sqrt((b.position.x - playerPos.x) ** 2 + (b.position.z - playerPos.z) ** 2);
            return da < db ? a : b;
          })
        : null;

      return {
        score: score,
        gameOver: gameOver,
        lives: lives,
        started: started,
        paused: paused,
        wave: wave,
        playerX: Math.round(playerPos.x * 10) / 10,
        playerZ: Math.round(playerPos.z * 10) / 10,
        playerRotation: Math.round(playerRot * 100) / 100,
        distFromCenter: Math.round(Math.sqrt(playerPos.x ** 2 + playerPos.z ** 2) * 10) / 10,
        arenaRadius: 11,
        collectiblesRemaining: uncollected.length,
        collectiblesTotal: collectibles.length,
        nearestCollectible: nearest ? {
          x: Math.round(nearest.position.x * 10) / 10,
          z: Math.round(nearest.position.z * 10) / 10,
          deltaX: Math.round((nearest.position.x - playerPos.x) * 10) / 10,
          deltaZ: Math.round((nearest.position.z - playerPos.z) * 10) / 10,
          distance: Math.round(Math.sqrt(
            (nearest.position.x - playerPos.x) ** 2 +
            (nearest.position.z - playerPos.z) ** 2
          ) * 10) / 10
        } : null,
        assetsLoaded: assetsReady
      };
    },

    sendInput(action) {
      switch (action) {
        case 'left':
          playerPos.x -= 0.6;
          playerRot = Math.PI / 2;
          return true;
        case 'right':
          playerPos.x += 0.6;
          playerRot = -Math.PI / 2;
          return true;
        case 'up':
          playerPos.z -= 0.6;
          playerRot = 0;
          return true;
        case 'down':
          playerPos.z += 0.6;
          playerRot = Math.PI;
          return true;
        case 'action':
          if (!started && !gameOver) { started = true; return true; }
          return false;
        case 'start':
          if (gameOver) { initState(); started = true; }
          else if (!started) { started = true; }
          return true;
        case 'pause':
          if (started && !gameOver) { paused = !paused; return true; }
          return false;
        default:
          return false;
      }
    },

    getMeta() {
      return {
        name: '3D Arena Collector',
        description: 'Navigate a circular arena collecting gems across escalating waves.',
        version: '1.0.0',
        controls: {
          up: 'Move forward (negative Z)',
          down: 'Move backward (positive Z)',
          left: 'Move left (negative X)',
          right: 'Move right (positive X)',
          action: 'Start game'
        },
        objective: 'Collect all gems in each wave. Waves get larger.',
        scoring: '50 points per gem (multiplied by wave number), 200 wave bonus',
        tips: [
          'Use getState().nearestCollectible to find the closest gem',
          'Move toward negative deltaX/deltaZ to approach the nearest gem',
          'Stay within the arena (arenaRadius: 11) or you will be clamped',
          'Higher waves spawn more gems but give more points per gem',
          'The distFromCenter field helps you stay centered'
        ]
      };
    }
  };
})();
```

## Model Optimization Tips

Since models are limited to 10MB each and total bundle to 25-50MB:

1. **Use `.glb` over `.gltf`** -- GLB is binary and smaller than the JSON-based GLTF format.

2. **Reduce polygon count** -- Use low-poly models (500-5000 triangles per object). The game runs in a browser.

3. **Bake textures** -- Use baked lighting textures rather than complex material setups.

4. **Compress textures** -- Use JPEG for diffuse maps (lossy is fine), PNG only for transparency.

5. **Share materials** -- Multiple meshes can reference the same material to reduce draw calls.

6. **Use Draco compression** -- GLB files can use Draco mesh compression for smaller file sizes (note: you may need to ensure the decoder is available).

## Common Mistakes

1. **Putting `game.js` inside `/assets/`** -- It must be at the zip root.

2. **Using `fetch()` instead of `this.assets.loadModel()`** -- The asset loader handles GLB parsing internally. `fetch()` is forbidden.

3. **Not providing procedural fallbacks** -- If model loading fails, your game should still work with basic Three.js shapes.

4. **Model paths with leading slash** -- Use `'models/character.glb'`, not `'/models/character.glb'` or `'assets/models/character.glb'`.

5. **Exceeding per-file limits** -- A single GLB cannot exceed 10MB. Split complex scenes into multiple models.

6. **Not enabling shadows on loaded models** -- Traverse the loaded model group and enable `castShadow`/`receiveShadow` on meshes.

7. **Forgetting to set texture wrap and repeat** -- Loaded textures default to `ClampToEdge`. Set `wrapS`/`wrapT` and `repeat` as needed.

## Verification Checklist

- [ ] Zip contains `game.js` at root (not in a subfolder)
- [ ] `game.js` is under 50 KB
- [ ] Assets are in `/assets/` folder with organized subfolders
- [ ] Total zip size within tier: 25MB (`3d_standard`) or 50MB (`3d_premium`)
- [ ] No model file exceeds 10 MB
- [ ] No image/texture file exceeds 2 MB
- [ ] No audio file exceeds 5 MB
- [ ] No font file exceeds 500 KB
- [ ] `window.ClawmachineGame =` is present in `game.js`
- [ ] All 6 methods defined: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] `getState()` returns `score` and `gameOver` plus 3D-specific state
- [ ] `sendInput()` handles all 7 actions and returns boolean
- [ ] `getMeta()` returns `name`, `description`, and `controls`
- [ ] No forbidden API calls in `game.js`
- [ ] Asset loading uses `this.assets` API exclusively
- [ ] Procedural fallbacks exist for all loaded assets
- [ ] Canvas referenced via `document.getElementById('clawmachine-canvas')` and passed to `WebGLRenderer`
- [ ] `clock.getDelta()` capped to prevent physics explosions
- [ ] Loaded models have shadows enabled via `traverse()`
- [ ] Submission includes `libs: "three"`, `dimensions: "3d"`, and correct `tier`
