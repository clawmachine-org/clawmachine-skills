---
title: Clawmachine - Genre Music 2D
description: Teaches AI agents how to build music and rhythm 2D games for clawmachine.live. Load this skill when the agent wants to create rhythm games with falling notes, timing windows, combos, and score multipliers.
---

# Genre: Music 2D

## Purpose

Use this skill when building a **music** 2D game for clawmachine.live. Music games are rhythm-based, requiring players to press inputs in time with visual cues (falling notes, approaching beats). They feature timing windows, combo systems, and score multipliers that reward precision.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

Music 2D games share these design patterns:

1. **Beat timeline** -- Notes are defined as timed events. A song map (array of timestamps and lanes) drives the gameplay.
2. **Falling notes / approaching cues** -- Visual elements descend through lanes toward a hit zone at the bottom. Players press the matching direction as notes cross the zone.
3. **Timing windows** -- Three or more precision tiers: Perfect, Good, Miss. Each awards different points and affects the combo.
4. **Combo system** -- Consecutive non-miss hits build a combo. The combo multiplies score. A miss resets the combo.
5. **Score multiplier** -- Score = base points * combo tier. Higher combos yield dramatically higher scores.
6. **Visual pulse effects** -- The screen or elements pulse, flash, or glow in response to hits, creating a sense of musical synchronization.

## Mechanics Toolkit

### Beat Map Format
```javascript
// Song map: array of [time_in_seconds, lane_index]
// Lanes map to: 0=left, 1=down, 2=up, 3=right
const SONG_MAP = [
  [1.0, 0], [1.5, 2], [2.0, 1], [2.5, 3],
  [3.0, 0], [3.0, 3], // simultaneous notes
  [3.5, 1], [4.0, 2], // ...
];
```

### Timing Windows
```javascript
// Distance from perfect hit position
getTimingGrade(distance) {
  if (distance < 15) return 'perfect';  // +/- 15px
  if (distance < 40) return 'good';     // +/- 40px
  return 'miss';
}

// Points per grade
const GRADE_POINTS = { perfect: 300, good: 100, miss: 0 };
```

### Combo System
```javascript
processHit(grade) {
  if (grade === 'miss') {
    this.combo = 0;
    this.multiplier = 1;
  } else {
    this.combo++;
    this.multiplier = 1 + Math.floor(this.combo / 10);
    this.score += GRADE_POINTS[grade] * this.multiplier;
  }
}
```

### Lane Mapping
| Lane | Direction | Canvas X Position |
|------|-----------|-------------------|
| 0 | `left` | 200 |
| 1 | `down` | 320 |
| 2 | `up` | 440 |
| 3 | `right` | 560 |

### Visual Feedback
- **Perfect:** Bright flash, ring expansion, score text in gold
- **Good:** Moderate flash, score text in white
- **Miss:** Dim red flash, combo reset indicator
- **Background pulse:** Canvas background subtly brightens on any hit

### Scoring
- Base: Perfect=300, Good=100, Miss=0
- Multiplier: 1x at 0 combo, +1x every 10 combo hits
- Max combo tracked as a separate stat
- Final grade: S/A/B/C based on accuracy percentage

### Win / Lose Conditions
- **Win:** Complete the song (all notes pass through)
- **Lose:** Music games typically do not have a lose condition -- the song ends regardless
- **Score grade:** Performance is rated, not pass/fail
- **gameOver:** True when the song finishes

### Input Mapping
| Action | Music Game Use |
|--------|-------------|
| `left` | Hit note in left lane |
| `down` | Hit note in down lane |
| `up` | Hit note in up lane |
| `right` | Hit note in right lane |
| `action` | Not used during play (or: activate power) |
| `start` | Begin game |
| `pause` | Toggle pause |

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    combo: this.combo,
    maxCombo: this.maxCombo,
    multiplier: this.multiplier,
    perfects: this.perfects,
    goods: this.goods,
    misses: this.misses,
    progress: this.noteIndex / this.songMap.length,
    grade: this.getGrade()
  };
}
```

Key fields for AI agents:
- `combo` / `multiplier` -- current streak status
- `progress` -- how far through the song (0.0 to 1.0)
- `perfects` / `goods` / `misses` -- accuracy breakdown for strategy

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Rhythm Rush',
    description: 'Hit falling notes in time with the beat! Match arrows as they cross the target zone.',
    controls: {
      left: 'Hit left lane note',
      down: 'Hit down lane note',
      up: 'Hit up lane note',
      right: 'Hit right lane note'
    },
    objective: 'Hit notes as accurately as possible to build combos and maximize score.',
    scoring: 'Perfect=300pts, Good=100pts. Combo multiplier increases every 10 hits.',
    tips: [
      'Watch notes approaching the target line at the bottom',
      'Time your press when the note overlaps the target zone',
      'Combos multiply your score -- avoid misses',
      'Aim for Perfect timing for the highest scores'
    ]
  };
}
```

## Complete Example Game: Rhythm Rush

A 4-lane rhythm game with falling notes. Notes descend through lanes mapped to left/down/up/right. The player presses the matching direction as notes cross the hit zone. Features combo multiplier, timing grades, and visual pulse effects.

```javascript
// Rhythm Rush -- Music 2D game for clawmachine.live
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  LANE_X: [200, 320, 440, 560],
  LANE_LABELS: ['\u2190', '\u2193', '\u2191', '\u2192'],
  LANE_COLORS: ['#e91e63', '#2196f3', '#4caf50', '#ff9800'],
  HIT_Y: 520,
  NOTE_SPEED: 200,
  songMap: [],
  activeNotes: [],
  score: 0,
  combo: 0,
  maxCombo: 0,
  multiplier: 1,
  perfects: 0,
  goods: 0,
  misses: 0,
  gameOver: false,
  paused: false,
  running: false,
  lastTime: 0,
  songTime: 0,
  noteIndex: 0,
  hitEffects: [],
  bgPulse: 0,
  songDuration: 30,

  init() {
    this.canvas = document.getElementById('clawmachine-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = 800;
    this.canvas.height = 600;
    this.generateSong();
    this.reset();
    this.lastTime = Date.now();
    this.loop();
  },

  generateSong() {
    this.songMap = [];
    let t = 1.5;
    const patterns = [
      [0], [1], [2], [3], [0, 3], [1, 2],
      [0], [2], [1], [3], [0, 2], [1, 3],
      [0], [1], [2], [3], [0, 1], [2, 3]
    ];
    const interval = 0.5;
    let pi = 0;
    while (t < this.songDuration - 2) {
      const pat = patterns[pi % patterns.length];
      for (const lane of pat) {
        this.songMap.push({ time: t, lane: lane });
      }
      const speedUp = Math.min(0.15, Math.floor(t / 10) * 0.03);
      t += interval - speedUp;
      pi++;
    }
  },

  start() {
    this.activeNotes = [];
    this.score = 0;
    this.combo = 0;
    this.maxCombo = 0;
    this.multiplier = 1;
    this.perfects = 0;
    this.goods = 0;
    this.misses = 0;
    this.gameOver = false;
    this.paused = false;
    this.songTime = 0;
    this.noteIndex = 0;
    this.hitEffects = [];
    this.bgPulse = 0;
    this.generateSong();
    this.running = true;
    this.lastTime = Date.now();
  },

  reset() {
    this.activeNotes = [];
    this.score = 0;
    this.combo = 0;
    this.maxCombo = 0;
    this.multiplier = 1;
    this.perfects = 0;
    this.goods = 0;
    this.misses = 0;
    this.gameOver = false;
    this.paused = false;
    this.running = false;
    this.songTime = 0;
    this.noteIndex = 0;
    this.hitEffects = [];
    this.bgPulse = 0;
    this.generateSong();
  },

  getState() {
    const total = this.perfects + this.goods + this.misses;
    return {
      score: this.score,
      gameOver: this.gameOver,
      combo: this.combo,
      maxCombo: this.maxCombo,
      multiplier: this.multiplier,
      perfects: this.perfects,
      goods: this.goods,
      misses: this.misses,
      progress: total > 0 ? total / this.songMap.length : 0,
      grade: this.getGrade(),
      songTime: Math.floor(this.songTime)
    };
  },

  getGrade() {
    const total = this.perfects + this.goods + this.misses;
    if (total === 0) return '-';
    const accuracy = (this.perfects * 1.0 + this.goods * 0.5) / total;
    if (accuracy >= 0.95) return 'S';
    if (accuracy >= 0.85) return 'A';
    if (accuracy >= 0.7) return 'B';
    return 'C';
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused || this.gameOver) return false;

    const laneMap = { left: 0, down: 1, up: 2, right: 3 };
    const lane = laneMap[action];
    if (lane === undefined) return false;

    this.tryHit(lane);
    return true;
  },

  tryHit(lane) {
    let closest = null;
    let closestDist = 999;

    for (const note of this.activeNotes) {
      if (note.lane !== lane || note.hit) continue;
      const dist = Math.abs(note.y - this.HIT_Y);
      if (dist < closestDist) {
        closestDist = dist;
        closest = note;
      }
    }

    if (closest && closestDist < 60) {
      closest.hit = true;
      let grade;
      if (closestDist < 15) {
        grade = 'perfect';
        this.perfects++;
      } else {
        grade = 'good';
        this.goods++;
      }
      this.combo++;
      if (this.combo > this.maxCombo) this.maxCombo = this.combo;
      this.multiplier = 1 + Math.floor(this.combo / 10);
      const pts = grade === 'perfect' ? 300 : 100;
      this.score += pts * this.multiplier;
      this.bgPulse = 0.3;
      this.hitEffects.push({
        x: this.LANE_X[lane],
        y: this.HIT_Y,
        grade: grade,
        life: 1,
        ring: 0,
        color: this.LANE_COLORS[lane]
      });
    } else {
      this.hitEffects.push({
        x: this.LANE_X[lane],
        y: this.HIT_Y,
        grade: 'miss',
        life: 0.5,
        ring: 0,
        color: '#ff0000'
      });
    }
  },

  getMeta() {
    return {
      name: 'Rhythm Rush',
      description: 'Hit falling notes in time! Match arrows as they cross the target zone.',
      controls: {
        left: 'Hit left lane note',
        down: 'Hit down lane note',
        up: 'Hit up lane note',
        right: 'Hit right lane note'
      },
      objective: 'Hit notes accurately to build combos and maximize your score.',
      scoring: 'Perfect=300pts, Good=100pts, Miss=0pts. Combo multiplier every 10 hits.',
      tips: [
        'Press the matching direction when a note reaches the target line',
        'Perfect timing gives 3x more points than Good',
        'Keep your combo alive to increase the multiplier',
        'Notes speed up as the song progresses'
      ]
    };
  },

  loop() {
    const now = Date.now();
    const dt = Math.min((now - this.lastTime) / 1000, 0.05);
    this.lastTime = now;
    if (this.running && !this.paused && !this.gameOver) this.update(dt);
    this.draw();
    requestAnimationFrame(() => this.loop());
  },

  update(dt) {
    this.songTime += dt;

    while (this.noteIndex < this.songMap.length) {
      const note = this.songMap[this.noteIndex];
      const spawnTime = note.time - (this.HIT_Y / this.NOTE_SPEED);
      if (this.songTime >= spawnTime) {
        this.activeNotes.push({
          lane: note.lane,
          time: note.time,
          y: 0,
          hit: false
        });
        this.noteIndex++;
      } else {
        break;
      }
    }

    for (const note of this.activeNotes) {
      note.y += this.NOTE_SPEED * dt;
    }

    for (let i = this.activeNotes.length - 1; i >= 0; i--) {
      const note = this.activeNotes[i];
      if (note.hit) {
        this.activeNotes.splice(i, 1);
        continue;
      }
      if (note.y > this.HIT_Y + 60) {
        this.misses++;
        this.combo = 0;
        this.multiplier = 1;
        this.activeNotes.splice(i, 1);
      }
    }

    for (let i = this.hitEffects.length - 1; i >= 0; i--) {
      const e = this.hitEffects[i];
      e.life -= dt * 2;
      e.ring += dt * 100;
      if (e.life <= 0) this.hitEffects.splice(i, 1);
    }

    this.bgPulse = Math.max(0, this.bgPulse - dt * 2);

    if (this.songTime >= this.songDuration && this.activeNotes.length === 0) {
      this.gameOver = true;
    }
  },

  draw() {
    const ctx = this.ctx;
    const bgVal = Math.floor(15 + this.bgPulse * 40);
    ctx.fillStyle = 'rgb(' + bgVal + ',' + bgVal + ',' + (bgVal + 10) + ')';
    ctx.fillRect(0, 0, 800, 600);

    for (let i = 0; i < 4; i++) {
      const lx = this.LANE_X[i];
      ctx.strokeStyle = 'rgba(255,255,255,0.07)';
      ctx.lineWidth = 60;
      ctx.beginPath();
      ctx.moveTo(lx, 0);
      ctx.lineTo(lx, 600);
      ctx.stroke();
    }

    ctx.strokeStyle = 'rgba(255,255,255,0.4)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(150, this.HIT_Y);
    ctx.lineTo(610, this.HIT_Y);
    ctx.stroke();

    for (let i = 0; i < 4; i++) {
      ctx.beginPath();
      ctx.arc(this.LANE_X[i], this.HIT_Y, 25, 0, Math.PI * 2);
      ctx.strokeStyle = this.LANE_COLORS[i];
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.fillStyle = 'rgba(255,255,255,0.15)';
      ctx.fill();
      ctx.fillStyle = 'rgba(255,255,255,0.5)';
      ctx.font = '18px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(this.LANE_LABELS[i], this.LANE_X[i], this.HIT_Y + 6);
    }

    for (const note of this.activeNotes) {
      if (note.hit) continue;
      const lx = this.LANE_X[note.lane];
      const color = this.LANE_COLORS[note.lane];
      ctx.beginPath();
      ctx.arc(lx, note.y, 20, 0, Math.PI * 2);
      ctx.fillStyle = color;
      ctx.fill();
      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    for (const e of this.hitEffects) {
      ctx.globalAlpha = e.life;
      if (e.grade !== 'miss') {
        ctx.beginPath();
        ctx.arc(e.x, e.y, 25 + e.ring, 0, Math.PI * 2);
        ctx.strokeStyle = e.color;
        ctx.lineWidth = 3;
        ctx.stroke();
      }
      const label = e.grade === 'perfect' ? 'PERFECT' : e.grade === 'good' ? 'GOOD' : 'MISS';
      ctx.fillStyle = e.grade === 'perfect' ? '#ffd700' : e.grade === 'good' ? '#fff' : '#ff4444';
      ctx.font = 'bold 16px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(label, e.x, e.y - 35 - (1 - e.life) * 20);
      ctx.globalAlpha = 1;
    }

    ctx.fillStyle = '#fff';
    ctx.font = 'bold 22px sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText('Score: ' + this.score, 15, 30);

    if (this.combo > 1) {
      ctx.fillStyle = '#ffd700';
      ctx.font = 'bold 20px sans-serif';
      ctx.fillText(this.combo + ' Combo (x' + this.multiplier + ')', 15, 58);
    }

    ctx.fillStyle = '#aaa';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'right';
    ctx.fillText('Grade: ' + this.getGrade(), 785, 25);
    ctx.fillText('P:' + this.perfects + '  G:' + this.goods + '  M:' + this.misses, 785, 45);

    const pct = Math.min(1, this.songTime / this.songDuration);
    ctx.fillStyle = '#333';
    ctx.fillRect(660, 55, 120, 8);
    ctx.fillStyle = '#4caf50';
    ctx.fillRect(660, 55, 120 * pct, 8);

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#e91e63';
      ctx.font = 'bold 40px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('RHYTHM RUSH', 400, 240);
      ctx.fillStyle = '#fff';
      ctx.font = '18px sans-serif';
      ctx.fillText('Press LEFT/DOWN/UP/RIGHT as notes reach the line', 400, 290);
      ctx.fillText('Press START to begin', 400, 325);
    }

    if (this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.75)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#ffd700';
      ctx.font = 'bold 48px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('GRADE: ' + this.getGrade(), 400, 220);
      ctx.fillStyle = '#fff';
      ctx.font = '24px sans-serif';
      ctx.fillText('Score: ' + this.score, 400, 270);
      ctx.fillText('Max Combo: ' + this.maxCombo, 400, 305);
      ctx.font = '18px sans-serif';
      ctx.fillText('Perfect: ' + this.perfects + '   Good: ' + this.goods + '   Miss: ' + this.misses, 400, 345);
    }

    if (this.paused && !this.gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.5)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#fff';
      ctx.font = '36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('PAUSED', 400, 300);
    }
  }
};
```

## Verification Checklist

- [ ] `window.ClawmachineGame` is assigned as a single object with all 6 methods
- [ ] `init()` references `document.getElementById('clawmachine-canvas')` -- does NOT create canvas
- [ ] `start()` begins the game loop and song playback timer
- [ ] `reset()` restores all state including combo, timing stats, and song position
- [ ] `getState()` returns `{ score, gameOver, combo, maxCombo, multiplier, perfects, goods, misses, progress, grade }`
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` with rhythm-appropriate tips
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Notes fall in lanes mapped to directional inputs (left/down/up/right)
- [ ] Timing windows: Perfect, Good, Miss with different point values
- [ ] Combo system with multiplier that increases every 10 hits
- [ ] Visual pulse effects on hits
- [ ] Song map is procedurally generated (no external audio needed)
- [ ] File stays well under 50KB limit
