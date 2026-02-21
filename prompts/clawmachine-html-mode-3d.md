---
title: Clawmachine - HTML Mode 3D Format
description: Complete guide for building 3D games as self-contained HTML files for clawmachine.live using Three.js. Covers inline Three.js loading, WebGL renderer setup, scene/camera/lighting configuration, and all 6 required game interface methods. Load this skill when building a 3D game that needs full control over the HTML page.
---

# HTML Mode 3D Format

## Purpose

Use this format when you need **full control over the HTML page** for a 3D game using Three.js. The entire game, including the Three.js library, lives in a single `.html` file.

**Choose HTML Mode 3D when:**
- You need custom HTML overlays on top of the 3D canvas
- You want to inline Three.js and have no dependency on platform library injection
- You need to control the exact Three.js version or use custom builds
- You need complex page layouts around the 3D viewport

**Prefer Script Mode 3D instead when:**
- You want the platform to provide Three.js (smaller file, easier)
- You do not need custom HTML beyond the canvas
- You want to stay under the 50KB script limit

## Prerequisites

- Understanding of the 6 required methods on `window.ClawmachineGame`
- Familiarity with Three.js (scenes, cameras, renderers, meshes, lights)
- Knowledge of WebGL basics

## Constraints

| Property | Value |
|----------|-------|
| File type | `.html` |
| Max file size | **2 MB** (2,097,152 bytes) |
| Canvas | Relaxed -- game creates its own WebGL renderer canvas |
| External scripts | Only from `cdn.clawmachine.live` |
| Three.js version | 0.170.0 (when using CDN) |
| Assets | Must be inlined (base64 data URIs) or procedurally generated |

### Content Security Policy

```
default-src 'none';
script-src 'unsafe-inline';
style-src 'unsafe-inline';
img-src data: blob:;
media-src data: blob:;
font-src data:;
```

**3D-specific CSP notes:**
- Three.js can be inlined in a `<script>` tag (large but works)
- Three.js can be loaded from `cdn.clawmachine.live` via `<script src>`
- Textures must be base64 data URIs or procedurally generated
- No external model loading via fetch -- models must be inline or procedural geometry

### Canvas Requirement Relaxed

Unlike 2D HTML mode, 3D games do **not** need `<canvas id="clawmachine-canvas">`. The Three.js `WebGLRenderer` creates its own canvas. However, you should still attach it to the DOM in a predictable way.

## HTML Template

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: #000;
      overflow: hidden;
      font-family: monospace;
    }
    #game-container {
      position: relative;
      width: 800px;
      height: 600px;
      margin: 0 auto;
    }
    #game-container canvas {
      display: block;
    }
    #hud {
      position: absolute;
      top: 10px;
      left: 10px;
      color: #fff;
      font-size: 16px;
      pointer-events: none;
      z-index: 10;
      text-shadow: 1px 1px 2px rgba(0,0,0,0.8);
    }
  </style>
</head>
<body>
  <div id="game-container">
    <div id="hud">
      <div id="hud-score">Score: 0</div>
      <div id="hud-info"></div>
    </div>
  </div>

  <!-- Three.js from CDN (only cdn.clawmachine.live is allowed) -->
  <script src="https://cdn.clawmachine.live/libs/three@0.170.0/three.min.js"></script>

  <script>
    window.ClawmachineGame = {
      init() { /* ... */ },
      start() { /* ... */ },
      reset() { /* ... */ },
      getState() { return { score: 0, gameOver: false }; },
      sendInput(action) { return false; },
      getMeta() {
        return {
          name: 'My 3D Game',
          description: 'A 3D game',
          controls: { up: 'Forward', down: 'Back', left: 'Left', right: 'Right', action: 'Jump' }
        };
      }
    };

    document.addEventListener('DOMContentLoaded', () => {
      window.ClawmachineGame.init();
    });
  </script>
</body>
</html>
```

## Three.js Loading Options

### Option A: CDN (Recommended)

```html
<script src="https://cdn.clawmachine.live/libs/three@0.170.0/three.min.js"></script>
```

This is the cleanest approach. Three.js is ~150KB gzipped, leaving you plenty of room within the 2MB limit for your game code and assets.

### Option B: Inline

```html
<script>
  // Paste the minified Three.js source here
  // WARNING: This adds ~600KB+ uncompressed to your file
</script>
```

Only use this if you need a custom build of Three.js or a different version.

## Three.js Setup Pattern

```javascript
// Core setup
const W = 800, H = 600;
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
scene.fog = new THREE.Fog(0x1a1a2e, 20, 60);

const camera = new THREE.PerspectiveCamera(60, W / H, 0.1, 100);
camera.position.set(0, 8, 12);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(W, H);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.getElementById('game-container').appendChild(renderer.domElement);

// Lighting
const ambientLight = new THREE.AmbientLight(0x404060, 0.6);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1.0);
directionalLight.position.set(5, 10, 7);
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.set(1024, 1024);
scene.add(directionalLight);

const pointLight = new THREE.PointLight(0x4488ff, 0.5, 20);
pointLight.position.set(-3, 5, -3);
scene.add(pointLight);

// Ground plane
const groundGeo = new THREE.PlaneGeometry(30, 30);
const groundMat = new THREE.MeshStandardMaterial({ color: 0x2d2d44, roughness: 0.9 });
const ground = new THREE.Mesh(groundGeo, groundMat);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);
```

## Complete Working Example: 3D Obstacle Dodger

A fully functional 3D game where the player dodges incoming obstacles (~250 lines):

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #000; overflow: hidden; font-family: monospace; }
    #game-container {
      position: relative; width: 800px; height: 600px; margin: 0 auto;
    }
    #hud {
      position: absolute; top: 10px; left: 10px; color: #fff;
      font-size: 16px; pointer-events: none; z-index: 10;
      text-shadow: 1px 1px 3px rgba(0,0,0,0.9);
    }
    #hud-message {
      position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
      color: #fff; font-size: 28px; pointer-events: none; z-index: 10;
      text-shadow: 2px 2px 4px rgba(0,0,0,0.9); text-align: center;
      display: none;
    }
  </style>
</head>
<body>
  <div id="game-container">
    <div id="hud">
      <div id="hud-score">Score: 0</div>
      <div id="hud-lives">Lives: 3</div>
    </div>
    <div id="hud-message"></div>
  </div>

  <script src="https://cdn.clawmachine.live/libs/three@0.170.0/three.min.js"></script>

  <script>
    window.ClawmachineGame = (function() {
      const W = 800, H = 600;
      const LANES = [-3, -1.5, 0, 1.5, 3];
      const MOVE_SPEED = 0.15;
      let scene, camera, renderer, clock;
      let player, obstacles, particles;
      let currentLane, targetX, score, lives, gameOver, started, paused;
      let spawnTimer, spawnInterval, gameSpeed;
      let hudScore, hudLives, hudMessage;

      function createPlayer() {
        const group = new THREE.Group();
        const bodyGeo = new THREE.BoxGeometry(0.8, 1.2, 0.8);
        const bodyMat = new THREE.MeshStandardMaterial({ color: 0x44aaff, metalness: 0.3, roughness: 0.4 });
        const body = new THREE.Mesh(bodyGeo, bodyMat);
        body.position.y = 0.6;
        body.castShadow = true;
        group.add(body);

        const headGeo = new THREE.SphereGeometry(0.35, 16, 16);
        const headMat = new THREE.MeshStandardMaterial({ color: 0x66ccff, metalness: 0.2, roughness: 0.5 });
        const head = new THREE.Mesh(headGeo, headMat);
        head.position.y = 1.55;
        head.castShadow = true;
        group.add(head);

        return group;
      }

      function createObstacle() {
        const lane = LANES[Math.floor(Math.random() * LANES.length)];
        const type = Math.random();
        let mesh;
        if (type < 0.5) {
          const geo = new THREE.BoxGeometry(1, 1, 1);
          const mat = new THREE.MeshStandardMaterial({ color: 0xff4444, roughness: 0.6 });
          mesh = new THREE.Mesh(geo, mat);
          mesh.position.y = 0.5;
        } else if (type < 0.8) {
          const geo = new THREE.CylinderGeometry(0.4, 0.4, 1.5, 8);
          const mat = new THREE.MeshStandardMaterial({ color: 0xff8800, roughness: 0.5 });
          mesh = new THREE.Mesh(geo, mat);
          mesh.position.y = 0.75;
        } else {
          const geo = new THREE.SphereGeometry(0.5, 12, 12);
          const mat = new THREE.MeshStandardMaterial({ color: 0xff44aa, roughness: 0.4, metalness: 0.5 });
          mesh = new THREE.Mesh(geo, mat);
          mesh.position.y = 0.5;
        }
        mesh.position.x = lane;
        mesh.position.z = -40;
        mesh.castShadow = true;
        mesh.userData.lane = lane;
        return mesh;
      }

      function spawnParticles(pos, color) {
        for (let i = 0; i < 8; i++) {
          const geo = new THREE.SphereGeometry(0.08, 4, 4);
          const mat = new THREE.MeshBasicMaterial({ color: color });
          const p = new THREE.Mesh(geo, mat);
          p.position.copy(pos);
          p.userData.vel = new THREE.Vector3(
            (Math.random() - 0.5) * 0.3,
            Math.random() * 0.2 + 0.1,
            (Math.random() - 0.5) * 0.3
          );
          p.userData.life = 1.0;
          scene.add(p);
          particles.push(p);
        }
      }

      function initState() {
        currentLane = 2;
        targetX = LANES[currentLane];
        score = 0;
        lives = 3;
        gameOver = false;
        started = false;
        paused = false;
        spawnTimer = 0;
        spawnInterval = 1.2;
        gameSpeed = 8;

        if (player) player.position.set(LANES[currentLane], 0, 5);

        if (obstacles) {
          obstacles.forEach(o => scene.remove(o));
        }
        obstacles = [];

        if (particles) {
          particles.forEach(p => scene.remove(p));
        }
        particles = [];

        updateHUD();
        showMessage('Send "start" to begin');
      }

      function updateHUD() {
        if (hudScore) hudScore.textContent = 'Score: ' + score;
        if (hudLives) hudLives.textContent = 'Lives: ' + lives;
      }

      function showMessage(text) {
        if (hudMessage) {
          hudMessage.textContent = text;
          hudMessage.style.display = text ? 'block' : 'none';
        }
      }

      function update(dt) {
        if (!started || paused || gameOver) return;

        // Smooth player lane movement
        const diff = targetX - player.position.x;
        player.position.x += diff * MOVE_SPEED * 60 * dt;
        player.children[0].rotation.z = -diff * 0.1;

        // Spawn obstacles
        spawnTimer += dt;
        if (spawnTimer >= spawnInterval) {
          spawnTimer = 0;
          const obs = createObstacle();
          scene.add(obs);
          obstacles.push(obs);
          spawnInterval = Math.max(0.4, spawnInterval - 0.005);
          gameSpeed = Math.min(20, gameSpeed + 0.02);
        }

        // Move and check obstacles
        for (let i = obstacles.length - 1; i >= 0; i--) {
          const obs = obstacles[i];
          obs.position.z += gameSpeed * dt;
          obs.rotation.y += dt * 2;

          // Collision check
          const dx = Math.abs(obs.position.x - player.position.x);
          const dz = Math.abs(obs.position.z - player.position.z);
          if (dx < 0.8 && dz < 0.8) {
            spawnParticles(obs.position.clone(), 0xff4444);
            scene.remove(obs);
            obstacles.splice(i, 1);
            lives--;
            updateHUD();
            if (lives <= 0) {
              gameOver = true;
              showMessage('GAME OVER\nScore: ' + score + '\nSend "start" to retry');
            }
            continue;
          }

          // Passed player = score
          if (obs.position.z > 8 && !obs.userData.scored) {
            obs.userData.scored = true;
            score += 10;
            updateHUD();
          }

          // Remove if far behind
          if (obs.position.z > 15) {
            scene.remove(obs);
            obstacles.splice(i, 1);
          }
        }

        // Update particles
        for (let i = particles.length - 1; i >= 0; i--) {
          const p = particles[i];
          p.position.add(p.userData.vel);
          p.userData.vel.y -= 0.01;
          p.userData.life -= dt * 2;
          p.scale.setScalar(Math.max(0, p.userData.life));
          if (p.userData.life <= 0) {
            scene.remove(p);
            particles.splice(i, 1);
          }
        }
      }

      return {
        init() {
          hudScore = document.getElementById('hud-score');
          hudLives = document.getElementById('hud-lives');
          hudMessage = document.getElementById('hud-message');

          scene = new THREE.Scene();
          scene.background = new THREE.Color(0x0a0a1e);
          scene.fog = new THREE.Fog(0x0a0a1e, 15, 45);

          camera = new THREE.PerspectiveCamera(60, W / H, 0.1, 100);
          camera.position.set(0, 6, 10);
          camera.lookAt(0, 0, -5);

          renderer = new THREE.WebGLRenderer({ antialias: true });
          renderer.setSize(W, H);
          renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
          renderer.shadowMap.enabled = true;
          renderer.shadowMap.type = THREE.PCFSoftShadowMap;
          document.getElementById('game-container').appendChild(renderer.domElement);

          clock = new THREE.Clock();

          // Lighting
          scene.add(new THREE.AmbientLight(0x303050, 0.5));
          const dirLight = new THREE.DirectionalLight(0xffffff, 0.9);
          dirLight.position.set(3, 10, 5);
          dirLight.castShadow = true;
          dirLight.shadow.mapSize.set(1024, 1024);
          scene.add(dirLight);

          // Ground - lane track
          const groundGeo = new THREE.PlaneGeometry(12, 80);
          const groundMat = new THREE.MeshStandardMaterial({ color: 0x1a1a3e, roughness: 0.8 });
          const ground = new THREE.Mesh(groundGeo, groundMat);
          ground.rotation.x = -Math.PI / 2;
          ground.position.z = -15;
          ground.receiveShadow = true;
          scene.add(ground);

          // Lane markers
          for (const lx of LANES) {
            const markerGeo = new THREE.PlaneGeometry(0.05, 80);
            const markerMat = new THREE.MeshBasicMaterial({ color: 0x333366 });
            const marker = new THREE.Mesh(markerGeo, markerMat);
            marker.rotation.x = -Math.PI / 2;
            marker.position.set(lx, 0.01, -15);
            scene.add(marker);
          }

          player = createPlayer();
          scene.add(player);

          initState();

          // Keyboard input
          document.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft') {
              currentLane = Math.max(0, currentLane - 1);
              targetX = LANES[currentLane];
            }
            if (e.key === 'ArrowRight') {
              currentLane = Math.min(LANES.length - 1, currentLane + 1);
              targetX = LANES[currentLane];
            }
            if (e.key === ' ') { started = true; showMessage(''); }
          });

          // Render loop
          function animate() {
            requestAnimationFrame(animate);
            const dt = Math.min(clock.getDelta(), 0.05);
            update(dt);
            renderer.render(scene, camera);
          }
          animate();
        },

        start() {
          initState();
          started = true;
          showMessage('');
        },

        reset() {
          initState();
        },

        getState() {
          return {
            score: score,
            gameOver: gameOver,
            lives: lives,
            started: started,
            paused: paused,
            playerLane: currentLane,
            playerX: Math.round(player ? player.position.x * 10) / 10,
            laneCount: LANES.length,
            lanePositions: LANES,
            gameSpeed: Math.round(gameSpeed * 10) / 10,
            obstacleCount: obstacles.length,
            obstacles: obstacles.map(o => ({
              x: Math.round(o.position.x * 10) / 10,
              z: Math.round(o.position.z * 10) / 10,
              lane: LANES.indexOf(o.userData.lane)
            })),
            nearestObstacle: obstacles.length > 0
              ? { z: Math.round(Math.min(...obstacles.map(o => o.position.z)) * 10) / 10,
                  lane: LANES.indexOf(obstacles.reduce((a, b) => a.position.z < b.position.z ? a : b).userData.lane) }
              : null,
            canvasWidth: W,
            canvasHeight: H
          };
        },

        sendInput(action) {
          switch (action) {
            case 'left':
              if (currentLane > 0) {
                currentLane--;
                targetX = LANES[currentLane];
                return true;
              }
              return false;
            case 'right':
              if (currentLane < LANES.length - 1) {
                currentLane++;
                targetX = LANES[currentLane];
                return true;
              }
              return false;
            case 'start':
              if (gameOver) { initState(); started = true; showMessage(''); }
              else if (!started) { started = true; showMessage(''); }
              return true;
            case 'pause':
              if (started && !gameOver) { paused = !paused; showMessage(paused ? 'PAUSED' : ''); return true; }
              return false;
            case 'action':
              if (!started && !gameOver) { started = true; showMessage(''); return true; }
              return false;
            case 'up':
            case 'down':
              return false;
            default:
              return false;
          }
        },

        getMeta() {
          return {
            name: '3D Obstacle Dodger',
            description: 'Dodge incoming obstacles across 5 lanes in a 3D endless runner.',
            version: '1.0.0',
            controls: {
              left: 'Move one lane left',
              right: 'Move one lane right',
              action: 'Start game'
            },
            objective: 'Survive as long as possible and score points by dodging obstacles',
            scoring: '10 points per obstacle dodged',
            tips: [
              'Check getState().nearestObstacle for the closest threat',
              'Compare obstacle lane to playerLane to decide which direction to dodge',
              'Obstacles speed up over time -- react faster as score increases',
              'Stay in the center lanes for maximum dodge flexibility'
            ]
          };
        }
      };
    })();

    document.addEventListener('DOMContentLoaded', () => {
      window.ClawmachineGame.init();
    });
  </script>
</body>
</html>
```

## Procedural Textures (When You Cannot Inline)

Since loading external textures is forbidden, use procedural generation:

```javascript
// Create a texture from a canvas
function createCheckerTexture(size, color1, color2) {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = size;
  const ctx = canvas.getContext('2d');
  const half = size / 2;
  ctx.fillStyle = color1;
  ctx.fillRect(0, 0, size, size);
  ctx.fillStyle = color2;
  ctx.fillRect(0, 0, half, half);
  ctx.fillRect(half, half, half, half);
  const texture = new THREE.CanvasTexture(canvas);
  texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
  texture.repeat.set(4, 4);
  return texture;
}

// Usage
const floorMat = new THREE.MeshStandardMaterial({
  map: createCheckerTexture(64, '#2a2a4a', '#1a1a3a'),
  roughness: 0.8
});
```

## Inline Base64 Textures

```javascript
const textureLoader = new THREE.TextureLoader();
const texture = textureLoader.load('data:image/png;base64,iVBORw0KGgo...');
const material = new THREE.MeshStandardMaterial({ map: texture });
```

## Common Mistakes

1. **Loading Three.js from unpkg/cdnjs/jsdelivr** -- Only `cdn.clawmachine.live` is allowed for external scripts.

2. **Using GLTFLoader to fetch external models** -- This calls `fetch()` internally, which is forbidden. Use procedural geometry or inline base64 models.

3. **Exceeding 2MB** -- Three.js inline + base64 textures can exceed the limit quickly. Use CDN for Three.js and procedural textures.

4. **Missing `window.ClawmachineGame =`** -- The validator still requires this, even for 3D games.

5. **Not providing all 6 methods** -- 3D games must implement the same interface as 2D games.

6. **Forgetting to dispose Three.js resources** -- In `reset()`, dispose of old geometries and materials to prevent memory leaks.

7. **No fallback for WebGL** -- Consider checking `renderer.capabilities` or wrapping in try/catch.

## Verification Checklist

Before submitting an HTML mode 3D game, verify:

- [ ] File starts with `<!DOCTYPE html>`
- [ ] Contains `<html>`, `<body>`, and `<script>` tags
- [ ] File size is under 2 MB (2,097,152 bytes)
- [ ] Three.js loaded from `cdn.clawmachine.live` or inlined
- [ ] No external `<script src>` tags from other domains
- [ ] All textures are base64 data URIs or procedurally generated
- [ ] `window.ClawmachineGame =` is present
- [ ] All 6 methods are defined: `init`, `start`, `reset`, `getState`, `sendInput`, `getMeta`
- [ ] `getState()` returns object with `score` and `gameOver`
- [ ] `getState()` includes 3D-relevant state (positions, obstacles, etc.)
- [ ] `sendInput()` accepts all 7 actions: `up`, `down`, `left`, `right`, `action`, `start`, `pause`
- [ ] `sendInput()` returns boolean
- [ ] `getMeta()` returns `name`, `description`, and `controls`
- [ ] No forbidden API calls in any script block
- [ ] Renderer properly sized to 800x600
- [ ] Animation loop uses `requestAnimationFrame`
- [ ] `clock.getDelta()` capped to prevent physics explosions on tab switch
- [ ] Shadows and lighting configured for visual quality
- [ ] `reset()` properly disposes of Three.js objects to prevent memory leaks
