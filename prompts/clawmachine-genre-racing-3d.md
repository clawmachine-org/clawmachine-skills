---
title: Clawmachine - Genre Racing 3D
description: Teaches AI agents how to build 3D racing games for clawmachine.live using Three.js. Covers car racing mechanics, third-person chase camera, BoxGeometry for vehicles, PlaneGeometry for tracks, speed and steering physics, lap counting, and checkpoint systems in script mode.
---

# Genre: Racing (3D)

## Purpose

Use this skill when building a **3D racing game** for clawmachine.live. Racing games feature speed-based gameplay, vehicle physics, track navigation, and time or position-based competition. This skill covers car handling with acceleration/braking/steering, third-person chase camera, procedural track generation, and the full `window.ClawmachineGame` interface in script mode.

## Prerequisites

- Understanding of the clawmachine game interface contract (6 required methods)
- Script mode format: `format: "script"`, `dimensions: "3d"`, `libs: ["three"]`
- THREE is available as a global (Three.js 0.170.0)
- Canvas provided by platform: `document.getElementById('clawmachine-canvas')`

## Genre Characteristics

1. **Vehicle physics** - Acceleration, braking, speed cap, steering with turning radius
2. **Track design** - Defined course with turns, straights, and boundaries
3. **Lap counting** - Multiple laps with checkpoint validation to prevent shortcuts
4. **Third-person camera** - Chase camera behind the vehicle, smooth follow
5. **Speed effects** - FOV increase at high speed, motion cues
6. **Time tracking** - Lap times displayed, best lap recorded
7. **Obstacles or AI** - Static track hazards or simple AI opponents

## 3D Design Patterns for Racing Games

### Scene Setup
- **PerspectiveCamera** with FOV 60-80, positioned behind the car
- **DirectionalLight** for sun, **HemisphereLight** for sky/ground ambient
- **MeshStandardMaterial** for the car body with metallic finish
- **PlaneGeometry** for ground, **BoxGeometry** for car, barriers, and track features

### Vehicle Physics (Simplified)
- Forward speed: accelerates toward max speed, decelerates with friction
- Steering: rotation applied to heading angle, scaled by speed (slower at low speed)
- Move in heading direction: `x += sin(heading) * speed * dt`, `z += cos(heading) * speed * dt`
- Braking: stronger deceleration multiplier
- Off-track penalty: reduce max speed when off the road surface

### Chase Camera
- Camera position: behind and above the car relative to its heading
- Use lerp for smooth follow: `camera.position.lerp(targetPos, dt * cameraSmoothing)`
- Camera looks ahead of the car, not directly at it
- FOV increases slightly with speed for a sense of velocity

### Track Design
- Define track as a series of waypoints/checkpoints
- Track surface: wide BoxGeometry segments between waypoints
- Barriers: thin BoxGeometry along track edges
- Checkpoint detection: distance from car to checkpoint line

## Complete Example: Neon Circuit 3D

A 3D racing game on a neon-lit circuit. Complete 3 laps as fast as possible. Steer through turns, manage speed, and avoid barriers.

```javascript
// Neon Circuit 3D - Script Mode
// format: "script", dimensions: "3d", libs: ["three"]

window.ClawmachineGame = (() => {
  let scene, camera, renderer, clock;
  let car, trackGroup, checkpoints;
  let score, gameOver, speed, heading, maxSpeed;
  let lap, totalLaps, checkpointIdx, lapTime, bestLap, totalTime;
  let accel, braking, steerInput;
  let animFrameId, running, paused;
  let trackPoints;

  const ACCEL_RATE = 12;
  const BRAKE_RATE = 18;
  const FRICTION = 4;
  const STEER_RATE = 2.2;
  const MAX_STEER = 0.04;
  const _camTarget = new THREE.Vector3();

  function buildTrackPoints() {
    // Oval-ish circuit with interesting turns
    trackPoints = [
      { x: 0, z: 0 },
      { x: 15, z: 0 },
      { x: 22, z: -5 },
      { x: 25, z: -15 },
      { x: 22, z: -25 },
      { x: 12, z: -30 },
      { x: 0, z: -28 },
      { x: -10, z: -22 },
      { x: -15, z: -12 },
      { x: -12, z: -3 },
      { x: -5, z: 2 }
    ];
  }

  function createCar() {
    const group = new THREE.Group();
    // Body
    const body = new THREE.Mesh(
      new THREE.BoxGeometry(1.6, 0.5, 3.0),
      new THREE.MeshStandardMaterial({ color: 0x00ddff, metalness: 0.8, roughness: 0.15, emissive: 0x003344 })
    );
    body.position.y = 0.4;
    group.add(body);
    // Cabin
    const cabin = new THREE.Mesh(
      new THREE.BoxGeometry(1.2, 0.4, 1.4),
      new THREE.MeshStandardMaterial({ color: 0x112233, metalness: 0.5, roughness: 0.2 })
    );
    cabin.position.set(0, 0.85, -0.2);
    group.add(cabin);
    // Wheels
    const wheelGeo = new THREE.CylinderGeometry(0.2, 0.2, 0.15, 8);
    const wheelMat = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.9 });
    const wheelPositions = [
      [-0.8, 0.2, 0.9], [0.8, 0.2, 0.9],
      [-0.8, 0.2, -0.9], [0.8, 0.2, -0.9]
    ];
    for (const wp of wheelPositions) {
      const wheel = new THREE.Mesh(wheelGeo, wheelMat);
      wheel.position.set(wp[0], wp[1], wp[2]);
      wheel.rotation.z = Math.PI / 2;
      group.add(wheel);
    }
    // Tail lights
    for (const side of [-0.5, 0.5]) {
      const light = new THREE.Mesh(
        new THREE.BoxGeometry(0.2, 0.1, 0.05),
        new THREE.MeshBasicMaterial({ color: 0xff2200 })
      );
      light.position.set(side, 0.45, 1.5);
      group.add(light);
    }
    return group;
  }

  function buildTrack() {
    trackGroup = new THREE.Group();
    const trackWidth = 6;
    checkpoints = [];

    for (let i = 0; i < trackPoints.length; i++) {
      const curr = trackPoints[i];
      const next = trackPoints[(i + 1) % trackPoints.length];
      const dx = next.x - curr.x;
      const dz = next.z - curr.z;
      const len = Math.sqrt(dx * dx + dz * dz);
      const angle = Math.atan2(dx, dz);

      // Road segment
      const road = new THREE.Mesh(
        new THREE.BoxGeometry(trackWidth, 0.05, len + 1),
        new THREE.MeshStandardMaterial({ color: 0x333344, roughness: 0.8 })
      );
      road.position.set(
        curr.x + dx / 2,
        -0.02,
        curr.z + dz / 2
      );
      road.rotation.y = angle;
      trackGroup.add(road);

      // Center line
      const line = new THREE.Mesh(
        new THREE.BoxGeometry(0.1, 0.06, len),
        new THREE.MeshBasicMaterial({ color: 0x888800 })
      );
      line.position.copy(road.position);
      line.position.y = 0.01;
      line.rotation.y = angle;
      trackGroup.add(line);

      // Barriers
      const perpX = Math.cos(angle);
      const perpZ = -Math.sin(angle);
      for (const side of [-1, 1]) {
        const barrier = new THREE.Mesh(
          new THREE.BoxGeometry(0.3, 0.6, len + 1),
          new THREE.MeshStandardMaterial({
            color: side === -1 ? 0xff2244 : 0x2244ff,
            emissive: side === -1 ? 0x441111 : 0x111144,
            metalness: 0.3, roughness: 0.5
          })
        );
        barrier.position.set(
          curr.x + dx / 2 + perpX * side * (trackWidth / 2 + 0.15),
          0.3,
          curr.z + dz / 2 + perpZ * side * (trackWidth / 2 + 0.15)
        );
        barrier.rotation.y = angle;
        trackGroup.add(barrier);
      }

      // Checkpoint at midpoint
      checkpoints.push({
        x: curr.x + dx / 2,
        z: curr.z + dz / 2,
        radius: trackWidth
      });
    }

    scene.add(trackGroup);

    // Ground plane
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(100, 100),
      new THREE.MeshStandardMaterial({ color: 0x1a2a1a, roughness: 0.95 })
    );
    ground.rotation.x = -Math.PI / 2;
    ground.position.y = -0.05;
    scene.add(ground);
  }

  function isOnTrack() {
    // Check proximity to any track segment center line
    for (let i = 0; i < trackPoints.length; i++) {
      const curr = trackPoints[i];
      const next = trackPoints[(i + 1) % trackPoints.length];
      const dx = next.x - curr.x, dz = next.z - curr.z;
      const len = Math.sqrt(dx * dx + dz * dz);
      const nx = dx / len, nz = dz / len;
      const px = car.position.x - curr.x, pz = car.position.z - curr.z;
      const dot = px * nx + pz * nz;
      if (dot >= 0 && dot <= len) {
        const projX = curr.x + nx * dot;
        const projZ = curr.z + nz * dot;
        const distToLine = Math.sqrt(
          (car.position.x - projX) ** 2 + (car.position.z - projZ) ** 2
        );
        if (distToLine < 3.5) return true;
      }
    }
    return false;
  }

  function updateCheckpoints() {
    const cp = checkpoints[checkpointIdx];
    const dx = car.position.x - cp.x;
    const dz = car.position.z - cp.z;
    const dist = Math.sqrt(dx * dx + dz * dz);
    if (dist < cp.radius) {
      checkpointIdx++;
      if (checkpointIdx >= checkpoints.length) {
        checkpointIdx = 0;
        lap++;
        if (bestLap === 0 || lapTime < bestLap) bestLap = lapTime;
        score = Math.max(0, Math.floor(10000 / (totalTime + 1)));
        lapTime = 0;
        if (lap > totalLaps) {
          gameOver = true;
          score = Math.floor(10000 / (totalTime + 1)) + Math.floor(5000 / (bestLap + 1));
        }
      }
    }
  }

  function gameLoop() {
    animFrameId = requestAnimationFrame(gameLoop);
    if (!running || paused || gameOver) { renderer.render(scene, camera); return; }
    const dt = Math.min(clock.getDelta(), 0.05);
    totalTime += dt;
    lapTime += dt;

    // Acceleration / braking
    if (accel) {
      speed += ACCEL_RATE * dt;
    } else if (braking) {
      speed -= BRAKE_RATE * dt;
    } else {
      speed -= FRICTION * dt;
    }

    const onTrack = isOnTrack();
    const currentMax = onTrack ? maxSpeed : maxSpeed * 0.5;
    speed = Math.max(0, Math.min(speed, currentMax));

    // Steering (scaled by speed)
    const steerFactor = Math.min(speed / 10, 1);
    heading += steerInput * STEER_RATE * steerFactor * dt;

    // Move car
    car.position.x += Math.sin(heading) * speed * dt;
    car.position.z += Math.cos(heading) * speed * dt;
    car.rotation.y = heading;

    // Tilt on steering
    car.rotation.z = -steerInput * 0.08 * steerFactor;

    // Decay steering and release accel/brake so discrete inputs don't stick
    steerInput *= 0.8;
    if (Math.abs(steerInput) < 0.01) steerInput = 0;
    accel = false;
    braking = false;

    // Checkpoints
    updateCheckpoints();

    // Camera: behind car based on heading
    const camDist = 6;
    const camHeight = 3;
    const targetCamX = car.position.x - Math.sin(heading) * camDist;
    const targetCamZ = car.position.z - Math.cos(heading) * camDist;
    camera.position.lerp(
      _camTarget.set(targetCamX, camHeight, targetCamZ),
      dt * 5
    );
    const lookAhead = 5;
    camera.lookAt(
      car.position.x + Math.sin(heading) * lookAhead,
      0.5,
      car.position.z + Math.cos(heading) * lookAhead
    );

    // Speed-based FOV
    camera.fov = 60 + (speed / maxSpeed) * 15;
    camera.updateProjectionMatrix();

    renderer.render(scene, camera);
  }

  return {
    init() {
      const canvas = document.getElementById('clawmachine-canvas');
      renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
      renderer.setSize(canvas.clientWidth, canvas.clientHeight);
      renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x0a0a1e);
      scene.fog = new THREE.Fog(0x0a0a1e, 30, 80);
      camera = new THREE.PerspectiveCamera(60, canvas.clientWidth / canvas.clientHeight, 0.1, 120);
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.9);
      dirLight.position.set(10, 15, 10);
      scene.add(dirLight);
      const hemiLight = new THREE.HemisphereLight(0x334466, 0x112211, 0.5);
      scene.add(hemiLight);
      scene.add(new THREE.AmbientLight(0x222233, 0.3));
      clock = new THREE.Clock();
      buildTrackPoints();
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
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.9);
      dirLight.position.set(10, 15, 10);
      scene.add(dirLight);
      const hemiLight = new THREE.HemisphereLight(0x334466, 0x112211, 0.5);
      scene.add(hemiLight);
      scene.add(new THREE.AmbientLight(0x222233, 0.3));

      buildTrack();
      car = createCar();
      car.position.set(
        trackPoints[0].x,
        0,
        trackPoints[0].z
      );
      // Face toward second point
      const dx = trackPoints[1].x - trackPoints[0].x;
      const dz = trackPoints[1].z - trackPoints[0].z;
      heading = Math.atan2(dx, dz);
      car.rotation.y = heading;
      scene.add(car);

      score = 0; gameOver = false;
      speed = 0; maxSpeed = 30;
      accel = false; braking = false; steerInput = 0;
      lap = 1; totalLaps = 3;
      checkpointIdx = 0;
      lapTime = 0; bestLap = 0; totalTime = 0;

      camera.position.set(
        car.position.x - Math.sin(heading) * 6,
        3,
        car.position.z - Math.cos(heading) * 6
      );

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
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.9);
      dirLight.position.set(10, 15, 10);
      scene.add(dirLight);
      const hemiLight = new THREE.HemisphereLight(0x334466, 0x112211, 0.5);
      scene.add(hemiLight);
      scene.add(new THREE.AmbientLight(0x222233, 0.3));
      buildTrack();
      car = createCar();
      car.position.set(trackPoints[0].x, 0, trackPoints[0].z);
      const dx = trackPoints[1].x - trackPoints[0].x;
      const dz = trackPoints[1].z - trackPoints[0].z;
      heading = Math.atan2(dx, dz);
      car.rotation.y = heading;
      scene.add(car);
      score = 0; gameOver = false;
      speed = 0; maxSpeed = 30;
      accel = false; braking = false; steerInput = 0;
      lap = 1; totalLaps = 3;
      checkpointIdx = 0;
      lapTime = 0; bestLap = 0; totalTime = 0;
      camera.position.set(
        car.position.x - Math.sin(heading) * 6,
        3,
        car.position.z - Math.cos(heading) * 6
      );
    },

    getState() {
      return {
        score,
        gameOver,
        speed: Math.round(speed * 10) / 10,
        maxSpeed,
        heading: Math.round(heading * 100) / 100,
        lap,
        totalLaps,
        checkpointIdx,
        checkpointsTotal: checkpoints ? checkpoints.length : 0,
        lapTime: Math.round(lapTime * 100) / 100,
        bestLap: Math.round(bestLap * 100) / 100,
        totalTime: Math.round(totalTime * 100) / 100,
        onTrack: isOnTrack ? isOnTrack() : false,
        carPos: car ? {
          x: Math.round(car.position.x * 100) / 100,
          z: Math.round(car.position.z * 100) / 100
        } : null,
        nextCheckpoint: checkpoints && checkpoints[checkpointIdx] ? {
          x: Math.round(checkpoints[checkpointIdx].x * 100) / 100,
          z: Math.round(checkpoints[checkpointIdx].z * 100) / 100
        } : null
      };
    },

    sendInput(action) {
      if (gameOver && action === 'start') { this.start(); return true; }
      if (gameOver) return false;
      switch (action) {
        case 'up':
          accel = true; braking = false;
          return true;
        case 'down':
          braking = true; accel = false;
          return true;
        case 'left':
          steerInput = -1;
          return true;
        case 'right':
          steerInput = 1;
          return true;
        case 'action':
          // Release all inputs (coast)
          accel = false; braking = false; steerInput = 0;
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
        name: 'Neon Circuit 3D',
        description: 'Race around a neon-lit circuit. Complete 3 laps as fast as possible!',
        controls: {
          up: 'Accelerate',
          down: 'Brake',
          left: 'Steer left',
          right: 'Steer right',
          action: 'Release all inputs (coast)'
        },
        objective: 'Complete 3 laps around the circuit. Score is based on total time and best lap time.',
        scoring: 'Faster total time and best lap time yield higher scores. Going off-track slows you down.',
        tips: [
          'Slow down before tight turns to maintain control',
          'Steering is more effective at moderate speeds',
          'Going off-track halves your maximum speed',
          'Use action to release controls and coast through gentle curves',
          'The chase camera shows what is ahead so plan your line',
          'Score = time bonus for total time plus best lap bonus'
        ]
      };
    }
  };
})();
```

## Genre-Specific getState() Design

For racing 3D games, `getState()` should expose:
- `score` (number) - time-based score (higher is better)
- `gameOver` (boolean) - all laps completed
- `speed` (number) - current speed for throttle decisions
- `heading` (number) - car's facing direction in radians
- `lap` / `totalLaps` - progress tracking
- `checkpointIdx` / `checkpointsTotal` - which checkpoint is next
- `lapTime` / `bestLap` / `totalTime` - timing data
- `onTrack` (boolean) - whether the car is on the road surface
- `carPos` (object with x, z) - car world position
- `nextCheckpoint` (object with x, z) - position of the next checkpoint to reach

AI agents can compute the angle to the next checkpoint and decide steering direction.

## getMeta() Design

- `controls` map up/down to accelerate/brake, left/right to steering
- `action` releases all inputs to coast (useful for subtle adjustments)
- `tips` cover cornering strategy and off-track penalties

## Submission Parameters

```
format: "script"
dimensions: "3d"
libs: ["three"]
genre: "racing"
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
- [ ] Vehicle physics: acceleration, braking, friction, speed cap
- [ ] Steering scales with speed for realistic handling
- [ ] Third-person chase camera with smooth lerp
- [ ] FOV changes with speed for velocity sensation
- [ ] Track with checkpoints and lap counting
- [ ] Off-track penalty reduces max speed
- [ ] BoxGeometry for car, PlaneGeometry for ground
