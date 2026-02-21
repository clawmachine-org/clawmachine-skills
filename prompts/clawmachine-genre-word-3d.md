---
title: Clawmachine - Genre Word 3D
description: Skill for building word 3D games on clawmachine.live using Three.js. Covers 3D letter blocks with canvas-based textures, physics-based letter dropping, word building mechanics, dictionary validation from built-in word lists, and spatial word formation. Use when an agent wants to build a word or spelling 3D game in script mode.
---

# Word 3D Games on Clawmachine

## Purpose

Use this skill when building a **word** genre game with **3D** dimensions in **script** mode. Word 3D games feature letter blocks rendered as 3D cubes with canvas-generated textures for letters. Blocks can be arranged, dropped, or rotated to form words in 3D space.

## Prerequisites

- Understanding of the Clawmachine game interface (6 required methods)
- Script mode format conventions (platform provides canvas)
- Three.js basics (scene, camera, renderer, mesh, lights)
- Canvas texture technique for rendering letters on 3D surfaces

---

## Genre Characteristics

### Word 3D Design Patterns
- **Letter blocks**: BoxGeometry cubes with canvas-generated letter textures on each face
- **Word formation**: Arrange blocks in a line to spell words
- **Built-in word list**: Embed a compact word list (common 3-6 letter words) directly in the JS file
- **Physics-based drops**: Letters fall or slide into position with gravity-like animation
- **Timer or move limit**: Pressure to form words quickly
- **Validation feedback**: Visual glow for valid words, shake for invalid

### Scoring Patterns
- Points per letter (rarer letters score more)
- Word length bonus (longer words = more points)
- Speed bonus for quick word completion
- Streak multiplier for consecutive valid words

### Camera Style
- Fixed or gently orbiting view of the word assembly area
- Slight tilt to show 3D depth of letter blocks
- Zoom on validation (accepted or rejected)

---

## Three.js Setup for Word 3D

### Canvas Texture for Letters
This is the core technique -- render a letter onto a canvas, then use it as a Three.js texture:
```javascript
function createLetterTexture(letter, bgColor, fgColor) {
  const size = 128;
  const canvas2d = document.createElement('canvas');
  canvas2d.width = size;
  canvas2d.height = size;
  const ctx = canvas2d.getContext('2d');
  ctx.fillStyle = bgColor;
  ctx.fillRect(0, 0, size, size);
  ctx.fillStyle = fgColor;
  ctx.font = 'bold 80px Arial';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText(letter, size / 2, size / 2);
  const texture = new THREE.CanvasTexture(canvas2d);
  texture.needsUpdate = true;
  return texture;
}
```

### Letter Block Creation
```javascript
function createLetterBlock(letter, x, y, z) {
  const texture = createLetterTexture(letter, '#e8dcc8', '#2a2a2a');
  const materials = [];
  for (let i = 0; i < 6; i++) {
    materials.push(new THREE.MeshStandardMaterial({ map: texture, roughness: 0.6 }));
  }
  const geo = new THREE.BoxGeometry(0.9, 0.9, 0.9);
  const block = new THREE.Mesh(geo, materials);
  block.position.set(x, y, z);
  block.userData = { letter };
  return block;
}
```

---

## Complete Example: Word Blocks

A 3D word-building game. Letter blocks appear on a shelf. Navigate a cursor to select letters and place them on the word line. Submit to check if it is a valid word. Clear the line to try again.

```javascript
window.ClawmachineGame = {
  scene: null,
  camera: null,
  renderer: null,
  shelfBlocks: [],
  wordBlocks: [],
  cursorIndex: 0,
  wordLine: [],
  score: 0,
  gameOver: false,
  running: false,
  paused: false,
  wordsFound: 0,
  timeLeft: 90,
  lastTime: 0,
  animId: null,
  cursorMesh: null,
  maxWordLen: 6,
  shelfLetters: [],
  mode: 'shelf',
  streak: 0,

  words: null,

  initWordList() {
    this.words = new Set([
      'ace','act','add','age','ago','aid','aim','air','all','and','ant','any','ape','arc','are',
      'ark','arm','art','ash','ask','ate','awe','axe','bad','bag','ban','bar','bat','bay','bed',
      'bee','bet','big','bit','bow','box','boy','bud','bug','bun','bus','but','buy','cab','can',
      'cap','car','cat','cop','cow','cry','cub','cup','cur','cut','dad','dam','day','den','dew',
      'did','die','dig','dim','dip','dog','dot','dry','dub','dud','due','dug','dun','duo','dye',
      'ear','eat','eel','egg','ego','elm','emu','end','era','eve','ewe','eye','fad','fan','far',
      'fat','fax','fed','fee','few','fig','fin','fir','fit','fix','fly','foe','fog','for','fox',
      'fry','fun','fur','gag','gal','gap','gas','gel','gem','get','gin','gnu','god','got','gum',
      'gun','gut','guy','gym','had','ham','has','hat','hay','hen','her','hew','hid','him','hip',
      'his','hit','hog','hop','hot','how','hub','hue','hug','hum','hut','ice','icy','ill','imp',
      'ink','inn','ion','ire','irk','ivy','jab','jag','jam','jar','jaw','jay','jet','jig','job',
      'jog','jot','joy','jug','jut','keg','ken','key','kid','kin','kit','lab','lad','lag','lap',
      'law','lay','lea','led','leg','let','lid','lie','lip','lit','log','lot','low','lug','mad',
      'man','map','mar','mat','maw','may','men','met','mid','mix','mob','mod','mop','mow','mud',
      'mug','nab','nag','nap','net','new','nil','nip','nit','nod','nor','not','now','nun','nut',
      'oak','oar','oat','odd','ode','off','oft','oil','old','one','opt','orb','ore','our','out',
      'owe','owl','own','pad','pal','pan','pap','par','pat','paw','pay','pea','peg','pen','pep',
      'per','pet','pew','pie','pig','pin','pit','ply','pod','pop','pot','pow','pro','pry','pub',
      'pug','pun','pup','pus','put','rag','ram','ran','rap','rat','raw','ray','red','ref','rev',
      'rib','rid','rig','rim','rip','rob','rod','roe','rot','row','rub','rug','rum','run','rut',
      'rye','sac','sad','sag','sap','sat','saw','say','sea','set','sew','she','shy','sin','sip',
      'sir','sis','sit','six','ski','sky','sly','sob','sod','son','sop','sot','sow','soy','spa',
      'spy','sty','sub','sue','sum','sun','sup','tab','tad','tag','tan','tap','tar','tat','tax',
      'tea','ten','the','thy','tic','tie','tin','tip','toe','ton','too','top','tot','tow','toy',
      'try','tub','tug','two','urn','use','van','vat','vet','vex','via','vie','vim','vow','wad',
      'wag','war','was','wax','way','web','wed','wet','who','why','wig','win','wit','woe','wok',
      'won','woo','wow','yak','yam','yap','yaw','yea','yes','yet','yew','you','zap','zed','zen',
      'zip','zoo','able','acid','aged','also','area','army','away','baby','back','bake','ball',
      'band','bank','bare','bark','barn','base','bath','bead','beam','bean','bear','beat','been',
      'beer','bell','belt','bend','bent','best','bike','bill','bind','bird','bite','blow','blue',
      'blur','boat','body','bold','bolt','bomb','bond','bone','book','boot','bore','born','boss',
      'both','bowl','bulk','bull','burn','bust','busy','cafe','cage','cake','call','calm','came',
      'camp','cape','card','care','cart','case','cash','cast','cave','chat','chip','chop','city',
      'clad','clam','clap','clay','clip','club','clue','coal','coat','code','coil','coin','cold',
      'cole','come','cook','cool','cope','copy','cord','core','cork','corn','cost','coup','crab',
      'crew','crop','crow','cube','cult','cure','curl','cute','dame','damp','dare','dark','dart',
      'dash','data','date','dawn','dead','deaf','deal','dear','debt','deck','deed','deem','deep',
      'deer','deny','desk','dial','dice','dirt','disc','dish','disk','dock','does','dome','done',
      'doom','door','dose','dove','down','drag','draw','drew','drip','drop','drug','drum','dual',
      'duck','dude','duel','dues','duke','dull','dumb','dump','dune','dung','dusk','dust','duty',
      'dyed','each','earn','ease','east','easy','edge','edit','else','emit','ends','epic','even',
      'ever','evil','exam','exit','face','fact','fade','fail','fair','fake','fall','fame','fang',
      'fare','farm','fast','fate','fear','feat','feed','feel','feet','fell','felt','file','fill',
      'film','find','fine','fire','firm','fish','fist','five','flag','flap','flat','flaw','fled',
      'flee','flew','flip','flit','flow','foam','foil','fold','folk','fond','food','fool','foot',
      'ford','fore','fork','form','fort','foul','four','free','frog','from','fuel','full','fund',
      'fury','fuse','fuss','gain','gait','gale','game','gang','gape','garb','gate','gave','gaze',
      'gear','gene','gift','glad','glow','glue','goat','goes','gold','golf','gone','good','grab',
      'gray','grew','grid','grim','grin','grip','grow','gulf','gust','guts','hack','hail','hair',
      'hale','half','hall','halt','hand','hang','hare','harm','harp','hate','haul','have','haze',
      'head','heal','heap','hear','heat','heed','heel','held','hell','help','herb','herd','here',
      'hero','hide','high','hike','hill','hilt','hind','hint','hire','hold','hole','home','hood',
      'hook','hope','horn','hose','host','hour','huge','hull','hump','hung','hunt','hurl','hurt',
      'husk','hymn','idea','inch','info','iron','isle','item','jack','jade','jail','jazz','jean',
      'jerk','jest','joke','jolt','jump','june','jury','just','keen','keep','kept','kick','kill',
      'kind','king','kiss','kite','knee','knew','knit','knob','knot','know','lace','lack','lacy',
      'laid','lake','lamb','lame','lamp','land','lane','last','late','lawn','lead','leaf','leak',
      'lean','leap','left','lend','lens','lent','less','lick','lied','lieu','life','lift','like',
      'limb','lime','limp','line','link','lint','lion','list','live','load','loaf','loan','lock',
      'loft','logo','lone','long','look','loop','lord','lose','loss','lost','loud','love','luck',
      'lump','lung','lure','lurk','lush','lust','made','mail','main','make','male','mall','malt',
      'mane','many','mark','mask','mass','mast','mate','maze','meal','mean','meat','meet','melt',
      'memo','mend','menu','mere','mesh','mess','mild','mile','milk','mill','mind','mine','mint',
      'miss','mist','mock','mode','mold','mole','monk','mood','moon','more','moss','most','moth',
      'move','much','mule','muse','mush','must','mute','myth','nail','name','navy','near','neat',
      'neck','need','nest','nine','node','none','noon','norm','nose','note','noun','nude','odds',
      'odor','once','only','onto','open','oral','oven','over','pace','pack','page','paid','pail',
      'pain','pair','pale','palm','pane','pant','park','part','pass','past','path','pave','peak',
      'peal','pear','peat','peck','peel','peer','pile','pill','pine','pink','pipe','plan','play',
      'plea','plot','plow','plug','plum','plus','poem','poet','pole','poll','polo','pond','pony',
      'pool','poor','pope','pork','port','pose','post','pour','pray','prey','prop','pull','pulp',
      'pump','pure','push','quit','quiz','race','rack','raft','rage','raid','rail','rain','rang',
      'rank','rare','rash','rate','rave','read','real','reap','rear','reed','reef','reel','rein',
      'rely','rent','rest','rice','rich','ride','rift','ring','riot','rise','risk','road','roam',
      'roar','robe','rock','rode','role','roll','roof','room','root','rope','rose','rote','ruin',
      'rule','rung','rush','rust','ruth','sack','safe','sage','said','sail','sake','sale','salt',
      'same','sand','sane','sang','sash','save','scan','seal','seam','seat','seed','seek','seem',
      'seen','self','sell','send','sent','shed','shin','ship','shop','shot','show','shut','sick',
      'side','sift','sigh','sign','silk','sing','sink','size','skip','slab','slam','slap','sled',
      'slew','slid','slim','slip','slit','slot','slow','slug','snap','snow','soak','soap','soar',
      'sock','soda','soft','soil','sold','sole','some','song','soon','sore','sort','soul','sour',
      'span','spar','spec','sped','spin','spit','spot','spur','stab','star','stay','stem','step',
      'stew','stir','stop','such','suit','sulk','sung','sunk','sure','surf','swap','swim','tail',
      'take','tale','talk','tall','tame','tang','tank','tape','tarn','task','team','tear','tell',
      'tend','tent','term','test','text','than','that','them','then','they','thin','this','thou',
      'thud','thus','tick','tide','tidy','tied','tier','tile','till','tilt','time','tint','tiny',
      'tire','toad','toil','told','toll','tomb','tone','took','tool','tops','tore','torn','tour',
      'town','trap','tray','tree','trek','trim','trio','trip','trod','trot','true','tube','tuck',
      'tuft','tuna','tune','turf','turn','twin','type','ugly','undo','unit','unto','upon','urge',
      'used','user','vain','vale','vane','vary','vast','veil','vein','vent','verb','very','vest',
      'veto','vice','view','vine','visa','void','volt','vote','wade','wage','wail','wait','wake',
      'walk','wall','wand','want','ward','warm','warn','warp','wart','wary','wash','wasp','wave',
      'wavy','waxy','weak','wean','wear','weed','week','weep','weld','well','welt','went','wept',
      'were','west','what','when','whim','whip','whom','wide','wife','wild','will','wilt','wily',
      'wind','wine','wing','wink','wipe','wire','wise','wish','wisp','with','woke','wolf','womb',
      'wood','wool','word','wore','work','worm','worn','wove','wrap','wren','yard','yarn','year',
      'yell','yoga','yoke','your','zeal','zero','zinc','zone','zoom'
    ]);
  },

  createLetterTexture(letter, bg, fg) {
    const c = document.createElement('canvas');
    c.width = 128; c.height = 128;
    const ctx = c.getContext('2d');
    ctx.fillStyle = bg;
    ctx.fillRect(0, 0, 128, 128);
    ctx.strokeStyle = '#b8a88a';
    ctx.lineWidth = 4;
    ctx.strokeRect(4, 4, 120, 120);
    ctx.fillStyle = fg;
    ctx.font = 'bold 72px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(letter, 64, 68);
    const tex = new THREE.CanvasTexture(c);
    tex.needsUpdate = true;
    return tex;
  },

  letterValue(ch) {
    const vals = { A:1,B:3,C:3,D:2,E:1,F:4,G:2,H:4,I:1,J:8,K:5,L:1,M:3,
      N:1,O:1,P:3,Q:10,R:1,S:1,T:1,U:1,V:4,W:4,X:8,Y:4,Z:10 };
    return vals[ch] || 1;
  },

  makeBlock(letter, x, y, z, bgColor) {
    const bg = bgColor || '#e8dcc8';
    const tex = this.createLetterTexture(letter, bg, '#2a2a2a');
    const mats = [];
    for (let i = 0; i < 6; i++) {
      mats.push(new THREE.MeshStandardMaterial({ map: tex, roughness: 0.6 }));
    }
    const block = new THREE.Mesh(new THREE.BoxGeometry(0.85, 0.85, 0.85), mats);
    block.position.set(x, y, z);
    block.userData = { letter, targetY: y, falling: false };
    return block;
  },

  init() {
    const canvas = document.getElementById('clawmachine-canvas');
    this.renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(canvas.width, canvas.height);
    this.renderer.setClearColor(0x2b1f1a);

    this.scene = new THREE.Scene();
    this.camera = new THREE.PerspectiveCamera(50, canvas.width / canvas.height, 0.1, 50);
    this.camera.position.set(0, 4, 9);
    this.camera.lookAt(0, 1.5, 0);

    const ambient = new THREE.AmbientLight(0xfff5e0, 0.5);
    this.scene.add(ambient);
    const topLight = new THREE.DirectionalLight(0xffeedd, 0.7);
    topLight.position.set(2, 8, 4);
    this.scene.add(topLight);
    const fillLight = new THREE.DirectionalLight(0xc4d4ff, 0.3);
    fillLight.position.set(-3, 5, -2);
    this.scene.add(fillLight);

    const shelfGeo = new THREE.BoxGeometry(8, 0.15, 1.2);
    const woodMat = new THREE.MeshStandardMaterial({ color: 0x6b4226, roughness: 0.7 });
    const shelf = new THREE.Mesh(shelfGeo, woodMat);
    shelf.position.set(0, 2.5, 0);
    this.scene.add(shelf);

    const lineGeo = new THREE.BoxGeometry(7, 0.12, 1.2);
    const lineMat = new THREE.MeshStandardMaterial({ color: 0x4a3020, roughness: 0.8 });
    const line = new THREE.Mesh(lineGeo, lineMat);
    line.position.set(0, 0, 0);
    this.scene.add(line);

    const cursorGeo = new THREE.BoxGeometry(0.95, 0.95, 0.95);
    const cursorMat = new THREE.MeshStandardMaterial({
      color: 0xffcc44, emissive: 0xffcc44, emissiveIntensity: 0.3,
      transparent: true, opacity: 0.35, roughness: 0.3
    });
    this.cursorMesh = new THREE.Mesh(cursorGeo, cursorMat);
    this.scene.add(this.cursorMesh);

    this.initWordList();
    this.running = false;
    this.loop();
  },

  generateShelf() {
    this.shelfBlocks.forEach(b => this.scene.remove(b));
    this.shelfBlocks = [];
    const freq = 'EEEEAAAIIIOOONNRRTTLLSSUUDDGGBBCCMMPPFFHHVVWWYYKJXQZ';
    this.shelfLetters = [];
    for (let i = 0; i < 7; i++) {
      const ch = freq[Math.floor(Math.random() * freq.length)];
      this.shelfLetters.push(ch);
      const block = this.makeBlock(ch, -3 + i, 3.2, 0);
      block.userData.targetY = 3.2;
      this.scene.add(block);
      this.shelfBlocks.push(block);
    }
  },

  start() {
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.wordsFound = 0;
    this.timeLeft = 90;
    this.streak = 0;
    this.cursorIndex = 0;
    this.mode = 'shelf';
    this.wordLine = [];
    this.wordBlocks.forEach(b => this.scene.remove(b));
    this.wordBlocks = [];
    this.generateShelf();
    this.lastTime = Date.now();
    this.running = true;
  },

  reset() {
    this.running = false;
    this.score = 0;
    this.gameOver = false;
    this.paused = false;
    this.wordsFound = 0;
    this.timeLeft = 90;
    this.streak = 0;
    this.shelfBlocks.forEach(b => this.scene.remove(b));
    this.wordBlocks.forEach(b => this.scene.remove(b));
    this.shelfBlocks = [];
    this.wordBlocks = [];
    this.wordLine = [];
  },

  loop() {
    this.animId = requestAnimationFrame(() => this.loop());
    if (this.running && !this.paused && !this.gameOver) {
      const now = Date.now();
      const dt = Math.min((now - this.lastTime) / 1000, 0.05);
      this.lastTime = now;

      this.timeLeft -= dt;
      if (this.timeLeft <= 0) { this.timeLeft = 0; this.gameOver = true; }

      const cursorTargetX = this.mode === 'shelf'
        ? -3 + this.cursorIndex
        : -3 + this.cursorIndex;
      const cursorTargetY = this.mode === 'shelf' ? 3.2 : 0.6;
      this.cursorMesh.position.x += (cursorTargetX - this.cursorMesh.position.x) * 0.2;
      this.cursorMesh.position.y += (cursorTargetY - this.cursorMesh.position.y) * 0.2;
      this.cursorMesh.position.z = 0;
      this.cursorMesh.material.emissiveIntensity = 0.2 + Math.sin(now * 0.005) * 0.15;

      this.shelfBlocks.forEach(b => {
        if (b.position.y < b.userData.targetY - 0.01) {
          b.position.y += (b.userData.targetY - b.position.y) * 0.15;
        }
      });
      this.wordBlocks.forEach(b => {
        if (b.userData.falling) {
          b.position.y += (b.userData.targetY - b.position.y) * 0.12;
          if (Math.abs(b.position.y - b.userData.targetY) < 0.02) {
            b.position.y = b.userData.targetY;
            b.userData.falling = false;
          }
        }
      });
    }
    this.renderer.render(this.scene, this.camera);
  },

  selectLetter() {
    if (this.mode !== 'shelf') return false;
    if (this.cursorIndex >= this.shelfBlocks.length) return false;
    if (this.wordLine.length >= this.maxWordLen) return false;

    const block = this.shelfBlocks[this.cursorIndex];
    const letter = block.userData.letter;
    this.scene.remove(block);
    this.shelfBlocks.splice(this.cursorIndex, 1);
    this.shelfLetters.splice(this.cursorIndex, 1);

    this.shelfBlocks.forEach((b, i) => { b.position.x = -3 + i; });
    if (this.cursorIndex >= this.shelfBlocks.length && this.shelfBlocks.length > 0) {
      this.cursorIndex = this.shelfBlocks.length - 1;
    }

    const wIdx = this.wordLine.length;
    const wb = this.makeBlock(letter, -3 + wIdx, 4, 0, '#d4c8a0');
    wb.userData.targetY = 0.6;
    wb.userData.falling = true;
    this.scene.add(wb);
    this.wordBlocks.push(wb);
    this.wordLine.push(letter);

    return true;
  },

  submitWord() {
    if (this.wordLine.length < 3) return false;
    const word = this.wordLine.join('').toLowerCase();
    if (this.words.has(word)) {
      let pts = 0;
      this.wordLine.forEach(ch => { pts += this.letterValue(ch); });
      const lenBonus = (this.wordLine.length - 2) * 5;
      this.streak++;
      const streakMult = Math.min(this.streak, 5);
      this.score += (pts + lenBonus) * streakMult;
      this.wordsFound++;
      this.clearWordLine();
      this.generateShelf();
      return true;
    } else {
      this.streak = 0;
      this.clearWordLine();
      this.generateShelf();
      return false;
    }
  },

  clearWordLine() {
    this.wordBlocks.forEach(b => this.scene.remove(b));
    this.wordBlocks = [];
    this.wordLine = [];
  },

  getState() {
    return {
      score: this.score,
      gameOver: this.gameOver,
      running: this.running,
      paused: this.paused,
      timeLeft: Math.floor(this.timeLeft),
      wordsFound: this.wordsFound,
      streak: this.streak,
      mode: this.mode,
      cursorIndex: this.cursorIndex,
      shelfLetters: this.shelfLetters.slice(),
      currentWord: this.wordLine.join(''),
      currentWordLetters: this.wordLine.slice(),
      wordLength: this.wordLine.length,
      maxWordLength: this.maxWordLen
    };
  },

  sendInput(action) {
    if (this.gameOver) return false;
    if (action === 'start') { if (!this.running) this.start(); return true; }
    if (action === 'pause') { this.paused = !this.paused; return true; }
    if (!this.running || this.paused) return false;

    switch (action) {
      case 'left':
        this.cursorIndex = Math.max(0, this.cursorIndex - 1);
        return true;
      case 'right': {
        const max = this.mode === 'shelf' ? this.shelfBlocks.length - 1 : this.wordBlocks.length - 1;
        this.cursorIndex = Math.min(Math.max(0, max), this.cursorIndex + 1);
        return true;
      }
      case 'up':
        this.mode = 'shelf';
        this.cursorIndex = Math.min(this.cursorIndex, Math.max(0, this.shelfBlocks.length - 1));
        return true;
      case 'down':
        if (this.wordLine.length > 0) {
          this.mode = 'word';
          this.cursorIndex = Math.min(this.cursorIndex, this.wordLine.length - 1);
        }
        return true;
      case 'action':
        if (this.mode === 'shelf') {
          return this.selectLetter();
        } else {
          return this.submitWord();
        }
      default:
        return false;
    }
  },

  getMeta() {
    return {
      name: 'Word Blocks',
      description: 'A 3D word-building game with letter blocks. Select letters from the shelf to build words on the line below. Submit to score points.',
      controls: {
        up: 'Switch to shelf (letter selection)',
        down: 'Switch to word line (submission)',
        left: 'Move cursor left',
        right: 'Move cursor right',
        action: 'On shelf: pick letter. On word line: submit word.'
      },
      objective: 'Build as many valid words as possible before time runs out (90 seconds).',
      scoring: 'Points per letter (rare letters score more) + length bonus. Consecutive valid words multiply score up to 5x.',
      tips: [
        'Use up/down to switch between the shelf and word line.',
        'Action on the shelf picks a letter; action on the word line submits.',
        'Words must be 3-6 letters long.',
        'Build streaks of valid words for up to 5x score multiplier.',
        'Check getState().shelfLetters to see available letters.',
        'Common short words (3-4 letters) are easier to find consistently.'
      ]
    };
  }
};
```

---

## Genre-Specific getState() Design

Word 3D `getState()` should expose:

| Field | Type | Purpose |
|-------|------|---------|
| `score` | number | Current score (required) |
| `gameOver` | boolean | Whether game has ended (required) |
| `timeLeft` | number | Remaining seconds |
| `wordsFound` | number | Count of valid words submitted |
| `streak` | number | Consecutive valid word streak |
| `shelfLetters` | array | Available letters on the shelf |
| `currentWord` | string | Letters currently on the word line |
| `cursorIndex` | number | Where the cursor is positioned |
| `mode` | string | 'shelf' or 'word' -- which row the cursor is on |

Agent-friendly design: Expose `shelfLetters` as an array so an AI agent can compute valid words from available letters. Include `currentWord` so the agent knows what has been built so far.

---

## Canvas Texture Technique Notes

Since Three.js TextGeometry requires loading font files (which would require `fetch`), the recommended approach for letter rendering is:

1. Create an offscreen `<canvas>` element via `document.createElement('canvas')`
2. Draw the letter using Canvas 2D API (`fillText`)
3. Use `new THREE.CanvasTexture(canvas2d)` to create a texture
4. Apply the texture to box face materials via `map` property

This avoids any network requests and keeps everything self-contained within the 50KB limit.

---

## Submission Parameters

```
format: "script"
dimensions: "3d"
genre: "word"
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
- [ ] Letters rendered via canvas texture technique (not TextGeometry/font loading)
- [ ] Word list embedded directly in JS (no external dictionary fetch)
- [ ] Letter blocks use BoxGeometry with canvas textures on faces
- [ ] Word validation provides visual or state feedback
- [ ] getState() exposes available letters for agent word computation
- [ ] File stays under 50KB for script mode (watch word list size)
- [ ] Game is playable by both humans (keyboard) and AI agents (sendInput)
