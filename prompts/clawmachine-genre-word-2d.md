---
title: Clawmachine - Genre Word 2D
description: Teaches AI agents how to build word 2D games for clawmachine.live. Load this skill when the agent wants to create word puzzles, typing games, or vocabulary challenges using directional controls for letter selection.
---

# Genre: Word 2D

## Purpose

Use this skill when building a **word** 2D game for clawmachine.live. Word games challenge players to form, guess, or discover words using a limited input set. Since the platform only provides directional controls and an action button, letter input is handled by cycling through the alphabet with up/down and confirming with action.

## Prerequisites

- Understand the clawmachine game interface contract (6 required methods)
- Familiar with script mode format (`.js`, 50KB limit, no HTML tags)
- Canvas is platform-provided: `document.getElementById('clawmachine-canvas')`, 800x600

## Genre Characteristics

Word 2D games share these design patterns:

1. **Built-in word list** -- A curated array of words embedded directly in the code. Keep it focused (50-200 words) to stay within the 50KB limit.
2. **Letter input via directional controls** -- Up/down cycles through A-Z, left/right moves the cursor position, and action confirms/submits.
3. **Visual feedback** -- Correct letters, wrong positions, and misses are shown with distinct colors (green, yellow, gray -- similar to Wordle conventions).
4. **Progressive challenge** -- Words may get longer, or the player gets fewer guesses.
5. **Score by performance** -- Points for correct guesses, speed bonuses, streak multipliers.

## Letter Input Strategy

Since there is no keyboard input, word games must provide a letter selection mechanism:

```javascript
// Letter cycling with up/down, position with left/right, confirm with action
sendInput(action) {
  switch (action) {
    case 'up':
      // Cycle letter forward: A -> B -> C ... Z -> A
      this.currentLetter = (this.currentLetter + 1) % 26;
      break;
    case 'down':
      // Cycle letter backward: A -> Z -> Y ... B -> A
      this.currentLetter = (this.currentLetter + 25) % 26;
      break;
    case 'left':
      // Move cursor left
      this.cursorPos = Math.max(0, this.cursorPos - 1);
      break;
    case 'right':
      // Move cursor right
      this.cursorPos = Math.min(this.wordLength - 1, this.cursorPos + 1);
      break;
    case 'action':
      // Place letter and/or submit guess
      this.placeLetter();
      break;
  }
}
```

## Mechanics Toolkit

### Word List Design
- Embed words directly as a JavaScript array
- Use common 5-letter words for Wordle-style games
- Filter to uppercase for consistency
- Consider categories: animals, foods, colors, etc.

### Scoring
- Base points per correct guess (e.g., 100)
- Bonus for fewer attempts (e.g., +50 per unused guess)
- Streak multiplier for consecutive correct words
- Penalty-free misses (just lose a guess)

### Difficulty Progression
- Start with shorter or more common words
- Increase word length over rounds
- Reduce the number of allowed guesses
- Introduce less common vocabulary

### Win / Lose Conditions
- **Win per round:** Guess the word within allowed attempts
- **Lose per round:** Exhaust all guesses without matching
- **Game over:** After a set number of rounds, or on first failure (configurable)

### Input Mapping
| Action | Word Game Use |
|--------|-------------|
| `up` | Cycle letter forward (A->B->C) |
| `down` | Cycle letter backward (C->B->A) |
| `left` | Move cursor left |
| `right` | Move cursor right |
| `action` | Place letter / Submit guess |
| `start` | Begin game |
| `pause` | Toggle pause |

## getState() Design

```javascript
getState() {
  return {
    score: this.score,
    gameOver: this.gameOver,
    round: this.round,
    guessesLeft: this.maxGuesses - this.guesses.length,
    currentGuess: this.currentGuess.join(''),
    cursorPos: this.cursorPos,
    currentLetter: String.fromCharCode(65 + this.currentLetter),
    previousGuesses: this.guesses,
    feedback: this.feedback,    // array of arrays: 'correct'|'present'|'absent'
    wordLength: this.wordLength,
    streak: this.streak
  };
}
```

Key fields for AI agents:
- `feedback` -- letter-by-letter feedback from previous guesses, critical for strategy
- `currentLetter` -- the letter currently selected for placement
- `cursorPos` -- which position the cursor is at
- `guessesLeft` -- remaining attempts

## getMeta() Design

```javascript
getMeta() {
  return {
    name: 'Word Guesser',
    description: 'Guess the 5-letter word within 6 tries. Cycle letters with UP/DOWN, move with LEFT/RIGHT, place with ACTION.',
    controls: {
      up: 'Next letter (A->B->C)',
      down: 'Previous letter (C->B->A)',
      left: 'Move cursor left',
      right: 'Move cursor right',
      action: 'Place letter / Submit guess when row is full'
    },
    objective: 'Guess each hidden word in as few tries as possible.',
    scoring: '100 points per correct word plus bonus for fewer guesses.',
    tips: [
      'Green = correct letter in correct position',
      'Yellow = letter is in the word but wrong position',
      'Gray = letter is not in the word',
      'Use feedback from previous guesses to narrow down'
    ]
  };
}
```

## Complete Example Game: Word Guesser

A Wordle-style game. Guess a 5-letter word in 6 tries. Cycle through letters with up/down, move cursor with left/right, place letter with action. When all 5 positions are filled, action submits the guess.

```javascript
// Word Guesser -- Word 2D game for clawmachine.live
window.ClawmachineGame = {
  canvas: null,
  ctx: null,
  WORDS: [
    'APPLE','BRAIN','CRANE','DANCE','EAGLE','FLAME','GRAPE','HOUSE',
    'IMAGE','JUICE','KNIFE','LEMON','MANGO','NIGHT','OCEAN','PLANT',
    'QUEEN','RIVER','STONE','TRAIN','ULTRA','VIVID','WATER','XENON',
    'YOUTH','ZEBRA','DREAM','BLAZE','CHARM','DIVER','FROST','GIANT',
    'HONEY','IVORY','JOKER','KNEEL','LUNAR','MAPLE','NOBLE','OPERA',
    'PEARL','QUILT','ROBIN','SOLAR','TOWER','UNITY','VALVE','WHEAT',
    'EXACT','YIELD','AZURE','BLOOM','CROWD','DRIFT','EMBER','FLUTE'
  ],
  target: '',
  guesses: [],
  feedback: [],
  currentGuess: [],
  currentLetter: 0,
  cursorPos: 0,
  maxGuesses: 6,
  wordLength: 5,
  score: 0,
  round: 1,
  streak: 0,
  gameOver: false,
  paused: false,
  running: false,
  lastTime: 0,
  message: '',
  messageTimer: 0,
  shakeTimer: 0,
  bounceRows: [],
  totalRounds: 5,

  init() {
    this.canvas = document.getElementById('clawmachine-canvas');
    this.ctx = this.canvas.getContext('2d');
    this.canvas.width = 800;
    this.canvas.height = 600;
    this.reset();
    this.lastTime = Date.now();
    this.loop();
  },

  start() {
    this.guesses = [];
    this.feedback = [];
    this.currentGuess = [];
    this.currentLetter = 0;
    this.cursorPos = 0;
    this.score = 0;
    this.round = 1;
    this.streak = 0;
    this.gameOver = false;
    this.paused = false;
    this.message = '';
    this.messageTimer = 0;
    this.shakeTimer = 0;
    this.bounceRows = [];
    this.target = '';
    this.knownLetters = {};
    this.running = true;
    this.pickWord();
    this.lastTime = Date.now();
  },

  reset() {
    this.guesses = [];
    this.feedback = [];
    this.currentGuess = [];
    this.currentLetter = 0;
    this.cursorPos = 0;
    this.score = 0;
    this.round = 1;
    this.streak = 0;
    this.gameOver = false;
    this.paused = false;
    this.running = false;
    this.message = '';
    this.messageTimer = 0;
    this.shakeTimer = 0;
    this.bounceRows = [];
    this.target = '';
    this.knownLetters = {};
  },

  pickWord() {
    this.target = this.WORDS[Math.floor(Math.random() * this.WORDS.length)];
    this.guesses = [];
    this.feedback = [];
    this.currentGuess = [];
    this.currentLetter = 0;
    this.cursorPos = 0;
    this.knownLetters = {};
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      round: this.round,
      totalRounds: this.totalRounds,
      guessesLeft: this.maxGuesses - this.guesses.length,
      currentGuess: this.currentGuess.join(''),
      cursorPos: this.cursorPos,
      currentLetter: String.fromCharCode(65 + this.currentLetter),
      previousGuesses: this.guesses,
      feedback: this.feedback,
      wordLength: this.wordLength,
      streak: this.streak
    };
  },

  sendInput(action) {
    if (action === 'start') { if (!this.running || this.gameOver) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused || this.gameOver) return false;
    if (this.messageTimer > 0) return false;

    switch (action) {
      case 'up':
        this.currentLetter = (this.currentLetter + 1) % 26;
        return true;
      case 'down':
        this.currentLetter = (this.currentLetter + 25) % 26;
        return true;
      case 'left':
        this.cursorPos = Math.max(0, this.cursorPos - 1);
        return true;
      case 'right':
        this.cursorPos = Math.min(this.wordLength - 1, this.cursorPos + 1);
        return true;
      case 'action':
        return this.placeOrSubmit();
      default:
        return false;
    }
  },

  placeOrSubmit() {
    const letter = String.fromCharCode(65 + this.currentLetter);
    if (this.currentGuess.length < this.wordLength) {
      while (this.currentGuess.length <= this.cursorPos) {
        this.currentGuess.push('');
      }
      this.currentGuess[this.cursorPos] = letter;
      if (this.cursorPos < this.wordLength - 1) {
        this.cursorPos++;
      } else {
        const filled = this.currentGuess.filter(function(c) { return c !== ''; }).length;
        if (filled === this.wordLength) {
          this.submitGuess();
        }
      }
      return true;
    }
    if (this.currentGuess.length === this.wordLength) {
      this.submitGuess();
      return true;
    }
    return false;
  },

  submitGuess() {
    const filled = this.currentGuess.filter(function(c) { return c !== ''; }).length;
    if (filled < this.wordLength) {
      this.shakeTimer = 0.3;
      return;
    }

    const guess = this.currentGuess.join('');
    const fb = [];
    const targetArr = this.target.split('');
    const used = [false, false, false, false, false];

    for (let i = 0; i < this.wordLength; i++) {
      if (guess[i] === targetArr[i]) {
        fb[i] = 'correct';
        used[i] = true;
        this.knownLetters[guess[i]] = 'correct';
      } else {
        fb[i] = null;
      }
    }
    for (let i = 0; i < this.wordLength; i++) {
      if (fb[i]) continue;
      let found = false;
      for (let j = 0; j < this.wordLength; j++) {
        if (!used[j] && guess[i] === targetArr[j]) {
          fb[i] = 'present';
          used[j] = true;
          found = true;
          if (!this.knownLetters[guess[i]]) this.knownLetters[guess[i]] = 'present';
          break;
        }
      }
      if (!found) {
        fb[i] = 'absent';
        if (!this.knownLetters[guess[i]]) this.knownLetters[guess[i]] = 'absent';
      }
    }

    this.guesses.push(guess);
    this.feedback.push(fb);

    if (guess === this.target) {
      const bonus = (this.maxGuesses - this.guesses.length) * 20;
      this.score += 100 + bonus;
      this.streak++;
      this.message = 'Correct! +' + (100 + bonus);
      this.messageTimer = 1.5;
      this.bounceRows.push({ row: this.guesses.length - 1, t: 0 });
      setTimeout(() => this.nextRound(), 1800);
    } else if (this.guesses.length >= this.maxGuesses) {
      this.streak = 0;
      this.message = 'The word was: ' + this.target;
      this.messageTimer = 2;
      setTimeout(() => this.nextRound(), 2200);
    }

    this.currentGuess = [];
    this.cursorPos = 0;
  },

  nextRound() {
    if (this.round >= this.totalRounds) {
      this.gameOver = true;
      this.message = 'Game Complete! Final Score: ' + this.score;
      this.messageTimer = 99;
    } else {
      this.round++;
      this.pickWord();
    }
  },

  getMeta() {
    return {
      name: 'Word Guesser',
      description: 'Guess 5-letter words. Cycle letters with UP/DOWN, move cursor with LEFT/RIGHT, place and submit with ACTION.',
      controls: {
        up: 'Next letter (A->B->C)',
        down: 'Previous letter (C->B->A)',
        left: 'Move cursor left',
        right: 'Move cursor right',
        action: 'Place letter / Submit guess'
      },
      objective: 'Guess each word in 6 tries or fewer across 5 rounds.',
      scoring: '100 points per word plus 20 bonus per unused guess.',
      tips: [
        'Green means correct letter and position',
        'Yellow means the letter is in the word but wrong spot',
        'Gray means the letter is not in the word',
        'Fill all 5 letters then press ACTION to submit'
      ]
    };
  },

  loop() {
    const now = Date.now();
    const dt = Math.min((now - this.lastTime) / 1000, 0.05);
    this.lastTime = now;
    if (this.running && !this.paused) this.update(dt);
    this.draw();
    requestAnimationFrame(() => this.loop());
  },

  update(dt) {
    if (this.messageTimer > 0) this.messageTimer -= dt;
    if (this.shakeTimer > 0) this.shakeTimer -= dt;
    for (const b of this.bounceRows) { b.t += dt * 3; }
  },

  draw() {
    const ctx = this.ctx;
    ctx.fillStyle = '#121213';
    ctx.fillRect(0, 0, 800, 600);

    ctx.fillStyle = '#fff';
    ctx.font = 'bold 28px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('WORD GUESSER', 400, 38);

    ctx.fillStyle = '#888';
    ctx.font = '14px sans-serif';
    ctx.fillText('Round ' + this.round + '/' + this.totalRounds + '   Score: ' + this.score + '   Streak: ' + this.streak, 400, 60);

    const gridX = 400 - (this.wordLength * 58) / 2;
    const gridY = 80;
    const cellW = 52;
    const cellH = 52;
    const gap = 6;

    const shakeX = this.shakeTimer > 0 ? (Math.random() - 0.5) * 8 : 0;

    for (let row = 0; row < this.maxGuesses; row++) {
      const bounce = this.bounceRows.find(function(b) { return b.row === row; });
      const by = bounce ? Math.sin(bounce.t) * 4 : 0;
      for (let col = 0; col < this.wordLength; col++) {
        const x = gridX + col * (cellW + gap) + (row === this.guesses.length ? shakeX : 0);
        const y = gridY + row * (cellH + gap) + by;

        let bg = '#3a3a3c';
        let letter = '';

        if (row < this.guesses.length) {
          letter = this.guesses[row][col];
          const fb = this.feedback[row][col];
          if (fb === 'correct') bg = '#538d4e';
          else if (fb === 'present') bg = '#b59f3b';
          else bg = '#3a3a3c';
        } else if (row === this.guesses.length) {
          bg = col === this.cursorPos ? '#555' : '#272729';
          letter = this.currentGuess[col] || '';
        } else {
          bg = '#272729';
        }

        ctx.fillStyle = bg;
        ctx.fillRect(x, y, cellW, cellH);
        ctx.strokeStyle = row === this.guesses.length && col === this.cursorPos ? '#fff' : '#555';
        ctx.lineWidth = row === this.guesses.length && col === this.cursorPos ? 2 : 1;
        ctx.strokeRect(x, y, cellW, cellH);

        if (letter) {
          ctx.fillStyle = '#fff';
          ctx.font = 'bold 28px sans-serif';
          ctx.textAlign = 'center';
          ctx.fillText(letter, x + cellW / 2, y + cellH / 2 + 10);
        }
      }
    }

    const kbY = 450;
    const alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
    ctx.font = '16px sans-serif';
    ctx.textAlign = 'center';
    for (let i = 0; i < 26; i++) {
      const ch = alphabet[i];
      const kx = 120 + (i % 13) * 44;
      const ky = kbY + Math.floor(i / 13) * 40;

      let kbg = '#818384';
      if (this.knownLetters[ch] === 'correct') kbg = '#538d4e';
      else if (this.knownLetters[ch] === 'present') kbg = '#b59f3b';
      else if (this.knownLetters[ch] === 'absent') kbg = '#3a3a3c';

      const isSelected = i === this.currentLetter;
      ctx.fillStyle = kbg;
      ctx.fillRect(kx, ky, 38, 32);
      if (isSelected) {
        ctx.strokeStyle = '#ffd700';
        ctx.lineWidth = 2;
        ctx.strokeRect(kx, ky, 38, 32);
      }
      ctx.fillStyle = '#fff';
      ctx.fillText(ch, kx + 19, ky + 22);
    }

    ctx.fillStyle = '#aaa';
    ctx.font = '13px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('Selected: ' + String.fromCharCode(65 + this.currentLetter) + '   Pos: ' + (this.cursorPos + 1), 400, 548);

    if (this.messageTimer > 0 && this.message) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(250, 260, 300, 50);
      ctx.fillStyle = '#ffd700';
      ctx.font = 'bold 20px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText(this.message, 400, 292);
    }

    if (!this.running) {
      ctx.fillStyle = 'rgba(0,0,0,0.7)';
      ctx.fillRect(0, 0, 800, 600);
      ctx.fillStyle = '#fff';
      ctx.font = 'bold 36px sans-serif';
      ctx.textAlign = 'center';
      ctx.fillText('WORD GUESSER', 400, 250);
      ctx.font = '18px sans-serif';
      ctx.fillText('UP/DOWN = select letter   LEFT/RIGHT = move cursor', 400, 300);
      ctx.fillText('ACTION = place letter & submit   START = begin', 400, 330);
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
- [ ] `start()` begins the game loop and picks the first word
- [ ] `reset()` restores all state including guesses, feedback, and round count
- [ ] `getState()` returns `{ score, gameOver, feedback, currentLetter, cursorPos, guessesLeft }`
- [ ] `sendInput(action)` handles all 7 actions: up, down, left, right, action, start, pause
- [ ] `getMeta()` returns `{ name, description, controls }` explaining letter cycling
- [ ] No forbidden APIs used (no localStorage, fetch, eval, etc.)
- [ ] No HTML tags -- pure JavaScript (script mode)
- [ ] Canvas dimensions are 800x600
- [ ] Word list is embedded as an array in the code
- [ ] Up/down cycles through A-Z, left/right moves cursor
- [ ] Visual feedback uses green (correct), yellow (present), gray (absent)
- [ ] File stays well under 50KB limit
