# Ivoirien Quiz Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` static quiz app testing Ivorian cultural knowledge — 12 questions, radar chart results, PDF certificate, 5 easter eggs.

**Architecture:** Everything lives in one `index.html` with inline `<style>` and `<script>`. Three `<section>` elements swap visibility for screen transitions. State is a single global object. No build step, no server.

**Tech Stack:** Vanilla HTML/CSS/JS · Chart.js CDN · jsPDF CDN · canvas-confetti CDN · Google Fonts (Space Mono, Lora)

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | Entire app — markup, styles, scripts |
| `README.md` | Deploy instructions |

All tasks modify only `index.html` unless stated otherwise.

---

### Task 1: HTML scaffold, CSS foundation, screen skeleton

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create the base HTML file**

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Es-tu vraiment Ivoirien·ne ?</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=Lora:ital,wght@0,400;0,600;1,400&display=swap" rel="stylesheet">
  <style>
    /* ── Reset ── */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; }

    /* ── Design tokens ── */
    :root {
      --bg:        #0D0D0D;
      --surface:   #161616;
      --border:    #2A2A2A;
      --orange:    #F77F00;
      --green:     #009A44;
      --white:     #FFFFFF;
      --muted:     #888888;
      --radius:    4px;
      --mono:      'Space Mono', monospace;
      --serif:     'Lora', Georgia, serif;
      --max-w:     480px;
    }

    body {
      background: var(--bg);
      color: var(--white);
      font-family: var(--serif);
      font-size: 16px;
      line-height: 1.6;
      min-height: 100vh;
    }

    /* ── Tricolor bar ── */
    .tricolor {
      height: 3px;
      width: 100%;
      background: linear-gradient(to right, var(--orange) 33.3%, var(--white) 33.3% 66.6%, var(--green) 66.6%);
      flex-shrink: 0;
    }

    /* ── Container ── */
    .container {
      max-width: var(--max-w);
      margin: 0 auto;
      padding: 0 20px;
      width: 100%;
    }

    /* ── Screens ── */
    .screen { display: none; min-height: 100vh; flex-direction: column; }
    .screen.active { display: flex; }

    /* ── Buttons ── */
    .btn {
      display: inline-block;
      padding: 14px 28px;
      min-height: 44px;
      font-family: var(--mono);
      font-size: 13px;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 0.08em;
      border: 1px solid var(--border);
      border-radius: var(--radius);
      cursor: pointer;
      transition: opacity 0.15s, transform 0.1s;
      text-align: center;
      width: 100%;
    }
    .btn:hover { opacity: 0.85; }
    .btn:active { transform: scale(0.98); }
    .btn-primary {
      background: var(--orange);
      border-color: var(--orange);
      color: var(--bg);
    }
    .btn-secondary {
      background: transparent;
      border-color: var(--border);
      color: var(--muted);
    }
    .btn:disabled { opacity: 0.5; cursor: not-allowed; }

    /* ── Toast ── */
    #toast-container {
      position: fixed;
      bottom: 24px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 9999;
      width: calc(100% - 40px);
      max-width: 400px;
      pointer-events: none;
    }
    .toast {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: var(--radius);
      padding: 12px 16px;
      font-family: var(--mono);
      font-size: 12px;
      color: var(--white);
      text-align: center;
      transform: translateY(80px);
      opacity: 0;
      transition: transform 0.3s ease, opacity 0.3s ease;
    }
    .toast.visible { transform: translateY(0); opacity: 1; }
  </style>
</head>
<body>

  <!-- Screen 1: Landing -->
  <section id="screen-landing" class="screen active">
    <div class="tricolor"></div>
    <div><!-- content added in Task 2 --></div>
  </section>

  <!-- Screen 2: Quiz -->
  <section id="screen-quiz" class="screen">
    <div class="tricolor"></div>
    <div><!-- content added in Task 5 --></div>
  </section>

  <!-- Screen 3: Result -->
  <section id="screen-result" class="screen">
    <div class="tricolor"></div>
    <div><!-- content added in Task 7 --></div>
  </section>

  <!-- Toast -->
  <div id="toast-container"></div>

  <!-- CDN scripts -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/canvas-confetti@1.9.2/dist/confetti.browser.min.js"></script>

  <script>
    // ── State ──────────────────────────────────────────────────────────────
    const state = {
      currentQuestion: 0,
      score: 0,
      answers: [],         // [{dimension, correct}]
      answered: false,
      phase: 'landing',
      flagClickCount: 0,
      keyBuffer: '',       // rolling 10-char buffer (lowercase)
      konamiBuffer: [],    // rolling array of last 10 key codes
      questionTimer: null,
      toastQueue: [],
      toastActive: false,
      radarChart: null,    // Chart.js instance (destroyed on restart)
    };

    // ── Screen helper ──────────────────────────────────────────────────────
    function showScreen(id) {
      document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
      document.getElementById(id).classList.add('active');
      state.phase = id.replace('screen-', '');
    }
  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify**

Open `index.html` in a browser. You should see a black page with a 3px tricolor bar at the top. No errors in console.

- [ ] **Step 3: Commit**

```bash
cd /Users/user/Documents/Projects/ivorianform
git init
git add index.html
git commit -m "feat: scaffold html, css tokens, screen system, toast styles"
```

---

### Task 2: Landing screen — static HTML + CSS

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace the landing section content**

Replace `<section id="screen-landing" ...>` with:

```html
<section id="screen-landing" class="screen active">
  <div class="tricolor"></div>

  <!-- Particle canvas (behind everything) -->
  <canvas id="particles" style="position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:0;"></canvas>

  <div class="container landing-content" style="position:relative;z-index:1;display:flex;flex-direction:column;justify-content:center;flex:1;padding-top:60px;padding-bottom:60px;">

    <!-- Typewriter label -->
    <div class="landing-label" id="landing-label" style="opacity:0;">
      <span id="typewriter-text"></span><span id="typewriter-cursor" class="cursor">|</span>
    </div>

    <!-- Title -->
    <h1 class="landing-title">
      <span id="title-line1" class="title-line" style="opacity:0;transform:translateX(-40px);">Es-tu vraiment</span>
      <span id="title-line2" class="title-line title-orange" style="opacity:0;transform:translateX(-40px);">Ivoirien·ne ?</span>
    </h1>

    <!-- Description -->
    <p id="landing-desc" class="landing-desc" style="opacity:0;transform:translateY(20px);">
      12 questions. 6 dimensions culturelles. Un verdict sans appel.
    </p>

    <!-- CTA -->
    <div id="landing-cta" style="opacity:0;transform:translateY(20px);margin-top:40px;">
      <button class="btn btn-primary" onclick="startQuiz()">COMMENCER LE TEST →</button>
      <p class="easter-hint">Psst... il y a des easter eggs cachés quelque part 👀</p>
    </div>

    <!-- Flag (easter egg trigger) -->
    <div id="flag-trigger" style="position:fixed;top:20px;right:20px;font-size:24px;cursor:pointer;z-index:10;" onclick="handleFlagClick()">🇨🇮</div>

  </div>
</section>
```

- [ ] **Step 2: Add landing CSS inside `<style>`**

Add after the last CSS rule (before `</style>`):

```css
/* ── Landing ── */
.landing-content { gap: 16px; }

.landing-label {
  font-family: var(--mono);
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.2em;
  color: var(--muted);
  margin-bottom: 8px;
}
.cursor {
  animation: blink 0.8s step-end infinite;
  color: var(--orange);
}
@keyframes blink { 50% { opacity: 0; } }

.landing-title {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.title-line {
  font-family: var(--mono);
  font-size: clamp(32px, 8vw, 52px);
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: -0.01em;
  line-height: 1.1;
  transition: opacity 0.5s ease, transform 0.5s ease;
}
.title-orange { color: var(--orange); }

.landing-desc {
  font-family: var(--serif);
  font-size: 16px;
  color: var(--muted);
  max-width: 360px;
  transition: opacity 0.5s ease, transform 0.5s ease;
}

.easter-hint {
  font-family: var(--mono);
  font-size: 10px;
  color: var(--border);
  text-align: center;
  margin-top: 16px;
  letter-spacing: 0.05em;
}
```

- [ ] **Step 3: Verify static layout**

Open in browser. Black page, tricolor bar, static text visible (all transparent/off-screen for now due to `opacity:0` — that's correct). No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: landing screen static html and css"
```

---

### Task 3: Landing animations + particle canvas

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add particle canvas system to `<script>`**

Add after the `showScreen` function:

```js
// ── Particles ─────────────────────────────────────────────────────────────
(function initParticles() {
  const canvas = document.getElementById('particles');
  const ctx = canvas.getContext('2d');
  let particles = [];
  let raf;

  function resize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
  }
  window.addEventListener('resize', resize);
  resize();

  function createParticle() {
    const r = Math.random();
    let color;
    if (r < 0.15) color = 'rgba(247,127,0,0.3)';
    else if (r < 0.25) color = 'rgba(0,154,68,0.2)';
    else color = `rgba(255,255,255,${(Math.random() * 0.04 + 0.01).toFixed(2)})`;
    return {
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 2 + 1,
      vx: (Math.random() - 0.5) * 0.3,
      vy: (Math.random() - 0.5) * 0.3,
      color,
    };
  }

  for (let i = 0; i < 22; i++) particles.push(createParticle());

  function tick() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    particles.forEach(p => {
      p.x += p.vx;
      p.y += p.vy;
      // wrap around
      if (p.x < -5) p.x = canvas.width + 5;
      if (p.x > canvas.width + 5) p.x = -5;
      if (p.y < -5) p.y = canvas.height + 5;
      if (p.y > canvas.height + 5) p.y = -5;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
      ctx.fillStyle = p.color;
      ctx.fill();
    });
    raf = requestAnimationFrame(tick);
  }
  tick();
})();
```

- [ ] **Step 2: Add landing animation sequence to `<script>`**

Add after the particles IIFE:

```js
// ── Landing animation sequence ─────────────────────────────────────────────
function runLandingAnimation() {
  const label = document.getElementById('landing-label');
  const typeEl = document.getElementById('typewriter-text');
  const cursor = document.getElementById('typewriter-cursor');
  const line1 = document.getElementById('title-line1');
  const line2 = document.getElementById('title-line2');
  const desc = document.getElementById('landing-desc');
  const cta = document.getElementById('landing-cta');

  const LABEL_TEXT = 'YOUR IVOIRITÉ';
  let i = 0;

  // Step 1: show label container
  label.style.opacity = '1';

  // Step 2: typewriter
  const typeInterval = setInterval(() => {
    typeEl.textContent = LABEL_TEXT.slice(0, ++i);
    if (i === LABEL_TEXT.length) {
      clearInterval(typeInterval);
      // Stop cursor blink after short delay
      setTimeout(() => { cursor.style.display = 'none'; }, 600);
      // Step 3: slide in title lines
      setTimeout(() => {
        line1.style.opacity = '1';
        line1.style.transform = 'translateX(0)';
      }, 200);
      setTimeout(() => {
        line2.style.opacity = '1';
        line2.style.transform = 'translateX(0)';
      }, 400);
      // Step 4: fade in desc
      setTimeout(() => {
        desc.style.opacity = '1';
        desc.style.transform = 'translateY(0)';
      }, 700);
      // Step 5: fade in CTA
      setTimeout(() => {
        cta.style.opacity = '1';
        cta.style.transform = 'translateY(0)';
      }, 950);
    }
  }, 80);
}

// Run on page load
window.addEventListener('DOMContentLoaded', runLandingAnimation);
```

- [ ] **Step 3: Add transitions to title lines and desc/cta in CSS**

The elements already have `transition` from Task 2. Verify the CSS includes:
```css
.title-line { transition: opacity 0.5s ease, transform 0.5s ease; }
.landing-desc { transition: opacity 0.5s ease, transform 0.5s ease; }
```
And add for cta (inside `<style>`):
```css
#landing-cta { transition: opacity 0.5s ease, transform 0.5s ease; }
```

- [ ] **Step 4: Open in browser and verify animation plays**

Page loads → "YOUR IVOIRITÉ" types out letter by letter → cursor disappears → title slides in from left (two lines staggered) → description fades up → button fades up. Background has subtle drifting dots.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: landing animations — typewriter, slide-in titles, particle canvas"
```

---

### Task 4: Data layer — questions + levels

**Files:**
- Modify: `index.html` (inside `<script>`, before state object)

- [ ] **Step 1: Add DIMENSIONS constant and questions array**

Add at the very top of the `<script>` block (before `const state`):

```js
// ── Data ───────────────────────────────────────────────────────────────────
const DIMENSIONS = ['langue', 'bouffe', 'debrouillardise', 'tempo', 'culture', 'fierte'];
const DIM_LABELS = {
  langue: 'LANGUE',
  bouffe: 'BOUFFE',
  debrouillardise: 'DÉBROUIL.',
  tempo: 'TEMPO',
  culture: 'CULTURE',
  fierte: 'FIERTÉ',
};

const questions = [
  {
    id: 1, dimension: 'langue', emoji: '🗣️',
    text: 'Quelqu\'un te dit "On va faire comment ?" — qu\'est-ce qu\'il veut dire vraiment ?',
    options: [
      { text: 'Il cherche des instructions GPS', correct: false },
      { text: 'La situation est compliquée, bonne chance à tous', correct: true },
      { text: 'Il propose de faire la cuisine ensemble', correct: false },
      { text: 'Il n\'a pas compris la question', correct: false },
    ],
    feedback: {
      correct: 'C\'est la résignation élégante ivoirienne face au chaos. Philosophie pure.',
      wrong: 'C\'est une façon de dire que la situation est délicate et qu\'on ne sait pas trop comment s\'en sortir.',
    },
  },
  {
    id: 2, dimension: 'bouffe', emoji: '🍽️',
    text: 'Le dimanche matin idéal à Abidjan commence par...',
    options: [
      { text: 'Un café noir et des croissants', correct: false },
      { text: 'Attiéké poisson braisé du maquis d\'à côté', correct: true },
      { text: 'Du müsli avec du lait écrémé', correct: false },
      { text: 'Un smoothie detox', correct: false },
    ],
    feedback: {
      correct: 'Champion ! Le dimanche appartient à l\'attiéké-poisson. C\'est sacré.',
      wrong: 'Les croissants c\'est pour les touristes. L\'attiéké-poisson du maquis, c\'est le rituel du dimanche ivoirien.',
    },
  },
  {
    id: 3, dimension: 'debrouillardise', emoji: '🚌',
    text: 'Un "gbaka" à Abidjan, c\'est quoi exactement ?',
    options: [
      { text: 'Un plat traditionnel à base de banane', correct: false },
      { text: 'Un minibus collectif informel', correct: true },
      { text: 'Une danse traditionnelle du Centre', correct: false },
      { text: 'Un surnom affectueux pour un grand frère', correct: false },
    ],
    feedback: {
      correct: 'Wêh ! Le gbaka c\'est le transport du peuple — rapide, économique, et les vitres ne ferment pas toujours.',
      wrong: 'Le gbaka c\'est le minibus collectif qui sillonne Abidjan. Sans lui, la ville serait ingérable.',
    },
  },
  {
    id: 4, dimension: 'tempo', emoji: '🕐',
    text: 'Une soirée est annoncée à 20h. Tu arrives à quelle heure, toi ?',
    options: [
      { text: '20h pile — je suis ponctuel', correct: false },
      { text: '19h45 — j\'aime aider à préparer', correct: false },
      { text: '21h30 — c\'est encore tôt', correct: true },
      { text: '23h — je laisse le temps aux autres d\'arriver d\'abord', correct: false },
    ],
    feedback: {
      correct: '21h30, c\'est presque en avance en temps ivoirien. La vraie fête commence bien après.',
      wrong: 'Arriver à l\'heure ? L\'hôte serait encore sous la douche ! Le "temps ivoirien" est une philosophie assumée.',
    },
  },
  {
    id: 5, dimension: 'culture', emoji: '🎵',
    text: 'Qui est Alpha Blondy ?',
    options: [
      { text: 'Un footballeur ivoirien des années 90', correct: false },
      { text: 'Le fondateur du Coupé-Décalé', correct: false },
      { text: 'Un légendaire chanteur de reggae ivoirien', correct: true },
      { text: 'Un acteur de Nollywood populaire en Côte d\'Ivoire', correct: false },
    ],
    feedback: {
      correct: 'Respect ! Connu dans le monde entier, chanté en Dioula, Français et Anglais.',
      wrong: 'Alpha Blondy est une légende mondiale du reggae né à Bouaké. "Jerusalem" c\'est lui.',
    },
  },
  {
    id: 6, dimension: 'langue', emoji: '💬',
    text: '"Djo" s\'utilise pour appeler...',
    options: [
      { text: 'Uniquement quelqu\'un qui s\'appelle Djorkaeff', correct: false },
      { text: 'N\'importe quel gars — un ami, un inconnu, ton cousin', correct: true },
      { text: 'Exclusivement les personnes plus âgées que toi', correct: false },
      { text: 'C\'est un terme irrespectueux à éviter', correct: false },
    ],
    feedback: {
      correct: 'Exactement ! "Djo" c\'est l\'interpellation universelle ivoirienne. Tout le monde est Djo.',
      wrong: '"Djo" c\'est l\'équivalent de "mec" — neutre, affectueux, universel.',
    },
  },
  {
    id: 7, dimension: 'culture', emoji: '🏘️',
    text: 'Yopougon pour un Abidjanais, c\'est...',
    options: [
      { text: 'Un village de brousse loin d\'Abidjan', correct: false },
      { text: 'Le berceau de la culture populaire abidjanaise', correct: true },
      { text: 'Un quartier résidentiel pour les expatriés', correct: false },
      { text: 'Une zone industrielle sans intérêt culturel', correct: false },
    ],
    feedback: {
      correct: 'Yopougon c\'est l\'âme d\'Abidjan ! Le Coupé-Décalé, les maquis, la culture de rue — tout vient de Yop City.',
      wrong: '"Yop City" c\'est le quartier le plus populaire et culturellement riche d\'Abidjan.',
    },
  },
  {
    id: 8, dimension: 'bouffe', emoji: '🥤',
    text: 'Le "bangui" c\'est quoi ?',
    options: [
      { text: 'La capitale de la République centrafricaine', correct: false },
      { text: 'Un vin de palme traditionnel légèrement fermenté', correct: true },
      { text: 'Un style vestimentaire populaire chez les jeunes', correct: false },
      { text: 'Un mot nouchi pour dire "problème"', correct: false },
    ],
    feedback: {
      correct: 'Parfait ! Tiré du palmier, légèrement fermenté, servi frais au maquis.',
      wrong: 'Le bangui c\'est le vin de palme. La boisson du peuple, servie fraîche.',
    },
  },
  {
    id: 9, dimension: 'culture', emoji: '🕺',
    text: 'Le Coupé-Décalé a été créé par...',
    options: [
      { text: 'Des musiciens traditionnels du Nord ivoirien', correct: false },
      { text: 'Des Ivoiriens de la diaspora parisienne dans les années 2000', correct: true },
      { text: 'Alpha Blondy et Tiken Jah Fakoly ensemble', correct: false },
      { text: 'Des DJs de Yopougon dans les années 90', correct: false },
    ],
    feedback: {
      correct: 'Né à Paris dans les clubs de la diaspora, popularisé par Douk Saga.',
      wrong: 'Le Coupé-Décalé a été créé à Paris par la diaspora ivoirienne — Douk Saga en est le pionnier.',
    },
  },
  {
    id: 10, dimension: 'debrouillardise', emoji: '🤝',
    text: 'Face à un problème administratif bloqué, la première solution ivoirienne c\'est...',
    options: [
      { text: 'Envoyer un mail de réclamation formel', correct: false },
      { text: 'Attendre que ça se débloque tout seul', correct: false },
      { text: 'Trouver "quelqu\'un qui connaît quelqu\'un là-bas"', correct: true },
      { text: 'Contacter l\'avocat de la famille', correct: false },
    ],
    feedback: {
      correct: 'Le réseau, toujours le réseau ! "Tu connais quelqu\'un là-bas ?" est la question qui résout 80% des problèmes.',
      wrong: 'En Côte d\'Ivoire, tout se règle par le réseau. C\'est la question magique universelle.',
    },
  },
  {
    id: 11, dimension: 'langue', emoji: '😤',
    text: '"C\'est pas ma faute, c\'est ___" — complète avec la réponse la plus ivoirienne.',
    options: [
      { text: '...la conjoncture', correct: true },
      { text: '...mon manager', correct: false },
      { text: '...l\'algorithme', correct: false },
      { text: '...mon enfance difficile', correct: false },
    ],
    feedback: {
      correct: '"C\'est la conjoncture !" — la réponse universelle à tous les problèmes économiques, personnels et cosmiques.',
      wrong: 'En Côte d\'Ivoire, "la conjoncture" explique absolument tout. Prix qui montent ? Conjoncture.',
    },
  },
  {
    id: 12, dimension: 'fierte', emoji: '⚽',
    text: 'Didier Drogba, au-delà du football, c\'est...',
    options: [
      { text: 'Juste un bon joueur de Chelsea parmi d\'autres', correct: false },
      { text: 'Une icône nationale qui a contribué à la paix civile ivoirienne', correct: true },
      { text: 'Principalement connu pour ses publicités télévisées', correct: false },
      { text: 'Le fondateur du championnat ivoirien de football', correct: false },
    ],
    feedback: {
      correct: 'En 2005, son appel télévisé a contribué à suspendre la guerre civile. Légende absolue.',
      wrong: 'En 2005, il a appelé à un cessez-le-feu en pleine guerre civile — et ça a marché.',
    },
  },
];

const levels = [
  { min: 0,  max: 2,  emoji: '🌍', title: 'Toubabou perdu',           desc: 'Tu es clairement étranger, mais tu essaies et c\'est mignon. Commence par apprendre ce qu\'est l\'attiéké avant ta prochaine visite.' },
  { min: 3,  max: 5,  emoji: '✈️',  title: 'Diaspora confuse',          desc: 'Tu as des souvenirs ivoiriens mais Abidjan t\'a un peu oublié. Tu arrives encore à l\'heure aux événements — ça se voit.' },
  { min: 6,  max: 8,  emoji: '🥭', title: 'Potentiellement Ivoirien·ne', desc: 'Tu connais les bases. Tu manges bien, tu danses presque juste. Mais t\'as encore des progrès sur le temps ivoirien.' },
  { min: 9,  max: 10, emoji: '🔥', title: 'Vrai·e Ivoirien·ne',         desc: 'GPS interne calibré sur Adjamé. Tu arrives deux heures en retard et tu penses être en avance. Félicitations.' },
  { min: 11, max: 12, emoji: '👑', title: 'Pur·e Abidjanais·e',         desc: 'Tu parles Nouchi dans ton sommeil. Tu sais exactement quel gbaka prendre. La Côte d\'Ivoire te réclame. Rentre au village.' },
];

function getLevel(score) {
  return levels.find(l => score >= l.min && score <= l.max);
}
```

- [ ] **Step 2: Verify in console**

Open browser console and type: `questions.length` → should return `12`. Type `getLevel(12).title` → should return `"Pur·e Abidjanais·e"`. Type `getLevel(0).title` → should return `"Toubabou perdu"`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add questions data, levels data, getLevel helper"
```

---

### Task 5: Quiz screen HTML + CSS

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace quiz section content**

Replace `<section id="screen-quiz" ...>` with:

```html
<section id="screen-quiz" class="screen">
  <div class="tricolor"></div>

  <!-- Progress bar -->
  <div class="quiz-header">
    <div class="container">
      <div class="progress-meta">
        <span id="q-counter" class="mono-label">QUESTION 1 / 12</span>
        <span id="q-score" class="mono-label">SCORE : 0</span>
      </div>
      <div class="progress-track">
        <div id="progress-fill" class="progress-fill" style="width:8.33%"></div>
      </div>
    </div>
  </div>

  <!-- Question card -->
  <div class="container quiz-body">
    <div id="question-card" class="question-card">
      <div id="q-emoji" class="q-emoji"></div>
      <p id="q-text" class="q-text"></p>
      <div id="q-options" class="q-options"></div>
      <div id="q-feedback" class="q-feedback" style="display:none;"></div>
      <div id="impatient-msg" class="impatient-msg" style="display:none;">
        ⏰ Tu réfléchis trop mon gars, c'est pas l'examen BEPC !
      </div>
    </div>

    <div id="btn-next-wrap" style="display:none;margin-top:16px;">
      <button class="btn btn-primary" id="btn-next" onclick="nextQuestion()">
        QUESTION SUIVANTE →
      </button>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Add quiz CSS inside `<style>`**

```css
/* ── Quiz ── */
.quiz-header {
  background: var(--bg);
  padding: 16px 0 12px;
  position: sticky;
  top: 0;
  z-index: 10;
  border-bottom: 1px solid var(--border);
}
.progress-meta {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}
.mono-label {
  font-family: var(--mono);
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--muted);
}
.progress-track {
  height: 2px;
  background: var(--border);
  border-radius: 1px;
  overflow: hidden;
}
.progress-fill {
  height: 100%;
  background: var(--orange);
  border-radius: 1px;
  transition: width 0.4s ease;
}

.quiz-body { padding-top: 32px; padding-bottom: 40px; }

.question-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 28px 24px 24px;
  animation: slideUp 0.35s ease forwards;
}
@keyframes slideUp {
  from { opacity: 0; transform: translateY(24px); }
  to   { opacity: 1; transform: translateY(0); }
}

.q-emoji {
  font-size: 40px;
  text-align: center;
  margin-bottom: 16px;
  line-height: 1;
}
.q-text {
  font-family: var(--serif);
  font-size: 17px;
  line-height: 1.5;
  text-align: center;
  margin-bottom: 24px;
  color: var(--white);
}

.q-options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.option-btn {
  display: block;
  width: 100%;
  min-height: 44px;
  padding: 12px 16px;
  font-family: var(--mono);
  font-size: 12px;
  text-align: left;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  color: var(--white);
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
  letter-spacing: 0.03em;
}
.option-btn:hover:not(:disabled) { border-color: var(--muted); }
.option-btn.correct {
  border-color: var(--green);
  color: var(--green);
  background: rgba(0, 154, 68, 0.08);
}
.option-btn.wrong {
  border-color: var(--orange);
  color: var(--orange);
  background: rgba(247, 127, 0, 0.08);
}
.option-btn:disabled { cursor: default; }

.q-feedback {
  margin-top: 16px;
  padding: 12px 14px;
  border-left: 2px solid var(--orange);
  font-family: var(--serif);
  font-size: 14px;
  color: var(--muted);
  animation: fadeIn 0.3s ease;
}
.q-feedback.is-correct { border-left-color: var(--green); }
@keyframes fadeIn {
  from { opacity: 0; } to { opacity: 1; }
}

.impatient-msg {
  margin-top: 12px;
  font-family: var(--mono);
  font-size: 11px;
  color: var(--muted);
  text-align: center;
  animation: fadeIn 0.3s ease;
}
```

- [ ] **Step 3: Verify HTML in browser**

Temporarily change `screen-landing` to not be active and `screen-quiz` to be active (add `active` class to quiz section in HTML) — you should see the quiz header with progress bar and the empty card. Revert after checking.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: quiz screen html and css"
```

---

### Task 6: Quiz JS logic

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add quiz rendering and logic functions**

Add after the `showScreen` function:

```js
// ── Quiz logic ─────────────────────────────────────────────────────────────
function startQuiz() {
  state.currentQuestion = 0;
  state.score = 0;
  state.answers = [];
  state.answered = false;
  showScreen('screen-quiz');
  renderQuestion(0);
}

function renderQuestion(index) {
  state.answered = false;
  const q = questions[index];

  // Update header
  document.getElementById('q-counter').textContent = `QUESTION ${index + 1} / 12`;
  document.getElementById('q-score').textContent = `SCORE : ${state.score}`;
  document.getElementById('progress-fill').style.width = `${((index + 1) / 12) * 100}%`;

  // Trigger card re-animation by cloning
  const card = document.getElementById('question-card');
  card.style.animation = 'none';
  card.offsetHeight; // force reflow
  card.style.animation = '';

  // Populate card
  document.getElementById('q-emoji').textContent = q.emoji;
  document.getElementById('q-text').textContent = q.text;

  // Options
  const optWrap = document.getElementById('q-options');
  optWrap.innerHTML = '';
  q.options.forEach((opt, i) => {
    const btn = document.createElement('button');
    btn.className = 'option-btn';
    btn.textContent = opt.text;
    btn.addEventListener('click', () => handleAnswer(i));
    optWrap.appendChild(btn);
  });

  // Reset feedback
  const fb = document.getElementById('q-feedback');
  fb.style.display = 'none';
  fb.textContent = '';
  fb.className = 'q-feedback';

  // Reset next button
  document.getElementById('btn-next-wrap').style.display = 'none';
  const btnNext = document.getElementById('btn-next');
  btnNext.textContent = index === 11 ? 'VOIR MON RÉSULTAT →' : 'QUESTION SUIVANTE →';
  btnNext.onclick = index === 11 ? showResult : nextQuestion;

  // Reset impatient message
  const imp = document.getElementById('impatient-msg');
  imp.style.display = 'none';

  // Start impatient timer (easter egg 5)
  clearTimeout(state.questionTimer);
  state.questionTimer = setTimeout(() => {
    if (!state.answered) {
      imp.style.display = 'block';
      setTimeout(() => { imp.style.display = 'none'; }, 4000);
    }
  }, 45000);
}

function handleAnswer(selectedIndex) {
  if (state.answered) return;
  state.answered = true;
  clearTimeout(state.questionTimer);

  const q = questions[state.currentQuestion];
  const isCorrect = q.options[selectedIndex].correct;
  if (isCorrect) state.score++;

  state.answers.push({ dimension: q.dimension, correct: isCorrect });

  // Update score display immediately
  document.getElementById('q-score').textContent = `SCORE : ${state.score}`;

  // Style option buttons
  const buttons = document.querySelectorAll('.option-btn');
  buttons.forEach((btn, i) => {
    btn.disabled = true;
    if (q.options[i].correct) btn.classList.add('correct');
    else if (i === selectedIndex && !isCorrect) btn.classList.add('wrong');
  });

  // Show feedback
  const fb = document.getElementById('q-feedback');
  fb.textContent = isCorrect ? q.feedback.correct : q.feedback.wrong;
  fb.className = 'q-feedback' + (isCorrect ? ' is-correct' : '');
  fb.style.display = 'block';

  // Show next button after short delay
  setTimeout(() => {
    document.getElementById('btn-next-wrap').style.display = 'block';
  }, 400);
}

function nextQuestion() {
  state.currentQuestion++;
  renderQuestion(state.currentQuestion);
}
```

- [ ] **Step 2: Open browser and test full quiz flow**

Click "COMMENCER LE TEST →". You should see question 1. Click an option — buttons disable, correct answer highlights green, wrong selection highlights orange, feedback appears, "QUESTION SUIVANTE →" appears. Click through all 12 questions. On question 12, the button should read "VOIR MON RÉSULTAT →".

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: quiz logic — answer handling, feedback, progression, impatient timer"
```

---

### Task 7: Result screen HTML + CSS

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Replace result section content**

Replace `<section id="screen-result" ...>` with:

```html
<section id="screen-result" class="screen">
  <div class="tricolor"></div>

  <div class="container result-body">

    <!-- Score header -->
    <div id="result-header" class="result-block" style="opacity:0;transform:translateY(20px);">
      <p class="result-scored">
        You scored <span id="result-score-badge" class="score-badge">0</span> / 12
      </p>
      <p id="result-level-line" class="result-level-line"></p>
    </div>

    <!-- Radar chart -->
    <div id="result-radar-wrap" class="result-block" style="opacity:0;transform:translateY(20px);">
      <canvas id="radar-chart"></canvas>
    </div>

    <!-- Dimension bars -->
    <div id="result-bars-wrap" class="result-block" style="opacity:0;transform:translateY(20px);">
      <div id="dimension-bars"></div>
    </div>

    <!-- Description -->
    <div id="result-desc-wrap" class="result-block" style="opacity:0;transform:translateY(20px);">
      <p id="result-desc" class="result-desc"></p>
      <p id="result-bonus" class="result-bonus" style="display:none;">
        🏆 Tu peux rentrer au village. Les anciens sont fiers de toi.
      </p>
    </div>

    <!-- Actions -->
    <div id="result-actions" class="result-block" style="opacity:0;transform:translateY(20px);">
      <button class="btn btn-primary" id="btn-certificate" onclick="generatePDF()">
        TÉLÉCHARGER MON CERTIFICAT
      </button>
      <button class="btn btn-secondary" style="margin-top:10px;" onclick="restartQuiz()">
        Recommencer
      </button>
    </div>

  </div>
</section>
```

- [ ] **Step 2: Add result CSS inside `<style>`**

```css
/* ── Result ── */
.result-body {
  padding-top: 32px;
  padding-bottom: 60px;
  display: flex;
  flex-direction: column;
  gap: 32px;
}
.result-block {
  transition: opacity 0.5s ease, transform 0.5s ease;
}

.result-scored {
  font-family: var(--mono);
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: var(--muted);
  text-align: center;
  margin-bottom: 8px;
}
.score-badge {
  font-family: var(--mono);
  font-size: 28px;
  font-weight: 700;
  color: var(--orange);
  display: inline-block;
  padding: 0 4px;
}
.result-level-line {
  font-family: var(--serif);
  font-style: italic;
  font-size: 16px;
  color: var(--muted);
  text-align: center;
}

/* Radar */
#result-radar-wrap {
  max-width: 320px;
  margin: 0 auto;
  width: 100%;
  aspect-ratio: 1;
}
#radar-chart { width: 100% !important; height: 100% !important; }

/* Dimension bars */
.dim-bar-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}
.dim-bar-label {
  font-family: var(--mono);
  font-size: 10px;
  color: var(--muted);
  width: 76px;
  flex-shrink: 0;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}
.dim-bar-track {
  flex: 1;
  height: 4px;
  background: var(--border);
  border-radius: 2px;
  overflow: hidden;
}
.dim-bar-fill {
  height: 100%;
  background: var(--orange);
  border-radius: 2px;
  width: 0%;
  transition: width 0.6s ease;
}
.dim-bar-pct {
  font-family: var(--mono);
  font-size: 10px;
  color: var(--muted);
  width: 32px;
  text-align: right;
  flex-shrink: 0;
}

.result-desc {
  font-family: var(--serif);
  font-size: 16px;
  line-height: 1.6;
  color: var(--muted);
  text-align: center;
  max-width: 380px;
  margin: 0 auto;
}
.result-bonus {
  font-family: var(--mono);
  font-size: 12px;
  color: var(--orange);
  text-align: center;
  margin-top: 12px;
  letter-spacing: 0.05em;
}

/* Perfect score glow on score badge */
.score-badge.glow {
  animation: glowPulse 1.2s ease-in-out infinite alternate;
}
@keyframes glowPulse {
  from { text-shadow: 0 0 8px rgba(247,127,0,0.4); }
  to   { text-shadow: 0 0 24px rgba(247,127,0,1), 0 0 48px rgba(247,127,0,0.5); }
}
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: result screen html and css"
```

---

### Task 8: Result JS — score calculation, radar chart, bars

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add dimension score calculation helper**

Add after `getLevel`:

```js
function getDimensionScores() {
  // Count correct answers and totals per dimension
  const correct = {};
  const totals = {};
  DIMENSIONS.forEach(d => { correct[d] = 0; totals[d] = 0; });
  questions.forEach(q => totals[q.dimension]++);
  state.answers.forEach(a => { if (a.correct) correct[a.dimension]++; });
  return DIMENSIONS.map(d => totals[d] > 0 ? Math.round((correct[d] / totals[d]) * 100) : 0);
}
```

- [ ] **Step 2: Add showResult function**

Add after `nextQuestion`:

```js
function showResult() {
  clearTimeout(state.questionTimer);
  showScreen('screen-result');

  const level = getLevel(state.score);
  const dimScores = getDimensionScores();

  // Destroy previous chart instance if exists
  if (state.radarChart) { state.radarChart.destroy(); state.radarChart = null; }

  // Staggered block reveals
  const blocks = ['result-header', 'result-radar-wrap', 'result-bars-wrap', 'result-desc-wrap', 'result-actions'];
  blocks.forEach((id, i) => {
    setTimeout(() => {
      const el = document.getElementById(id);
      el.style.opacity = '1';
      el.style.transform = 'translateY(0)';
    }, i * 200);
  });

  // Score header
  document.getElementById('result-score-badge').textContent = state.score;
  document.getElementById('result-level-line').textContent =
    `That puts you in the ${level.title} range.`;

  // Description
  document.getElementById('result-desc').textContent = level.desc;

  // Perfect score bonus (easter egg 3)
  if (state.score === 12) {
    document.getElementById('result-bonus').style.display = 'block';
    document.getElementById('result-score-badge').classList.add('glow');
    setTimeout(() => triggerConfetti(), 800);
  }

  // Radar chart (rendered after block is visible)
  setTimeout(() => renderRadar(dimScores), 250);

  // Dimension bars (staggered)
  setTimeout(() => renderBars(dimScores), 450);
}

function renderRadar(dimScores) {
  const ctx = document.getElementById('radar-chart').getContext('2d');
  state.radarChart = new Chart(ctx, {
    type: 'radar',
    data: {
      labels: DIMENSIONS.map(d => DIM_LABELS[d]),
      datasets: [{
        data: dimScores,
        backgroundColor: 'rgba(247, 127, 0, 0.12)',
        borderColor: '#F77F00',
        borderWidth: 2,
        pointBackgroundColor: '#F77F00',
        pointBorderColor: '#F77F00',
        pointRadius: 4,
        pointHoverRadius: 5,
      }],
    },
    options: {
      animation: { duration: 1000 },
      responsive: true,
      maintainAspectRatio: true,
      scales: {
        r: {
          min: 0,
          max: 100,
          ticks: { display: false, stepSize: 25 },
          grid: { color: '#2A2A2A', circular: false },
          angleLines: { color: '#2A2A2A' },
          pointLabels: {
            color: '#888888',
            font: { family: 'Space Mono', size: 9 },
          },
        },
      },
      plugins: { legend: { display: false }, tooltip: { enabled: false } },
    },
  });
}

function renderBars(dimScores) {
  const wrap = document.getElementById('dimension-bars');
  wrap.innerHTML = '';
  DIMENSIONS.forEach((dim, i) => {
    const pct = dimScores[i];
    const row = document.createElement('div');
    row.className = 'dim-bar-row';
    row.innerHTML = `
      <span class="dim-bar-label">${DIM_LABELS[dim]}</span>
      <div class="dim-bar-track"><div class="dim-bar-fill" id="bar-${dim}"></div></div>
      <span class="dim-bar-pct">${pct}%</span>
    `;
    wrap.appendChild(row);
    // Animate fill with stagger
    setTimeout(() => {
      document.getElementById(`bar-${dim}`).style.width = `${pct}%`;
    }, i * 120 + 50);
  });
}
```

- [ ] **Step 3: Add restartQuiz function**

```js
function restartQuiz() {
  // Reset state
  state.currentQuestion = 0;
  state.score = 0;
  state.answers = [];
  state.answered = false;
  state.flagClickCount = 0;
  // Destroy chart
  if (state.radarChart) { state.radarChart.destroy(); state.radarChart = null; }
  // Reset result screen opacities for next visit
  ['result-header','result-radar-wrap','result-bars-wrap','result-desc-wrap','result-actions'].forEach(id => {
    const el = document.getElementById(id);
    el.style.opacity = '0';
    el.style.transform = 'translateY(20px)';
  });
  document.getElementById('result-bonus').style.display = 'none';
  document.getElementById('result-score-badge').classList.remove('glow');
  showScreen('screen-landing');
}
```

- [ ] **Step 4: Test full flow end-to-end**

Start quiz, answer all 12 questions, reach result screen. Verify: score badge shows correct number, level line shows correct level name, radar chart renders with 6 axes, all 6 dimension bars animate in staggered, description shows correct text. Click "Recommencer" — returns to landing.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: result screen logic — radar chart, dimension bars, level display, restart"
```

---

### Task 9: PDF certificate generation

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add generatePDF function**

Add after `restartQuiz`:

```js
// ── PDF Certificate ────────────────────────────────────────────────────────
function generatePDF() {
  const btn = document.getElementById('btn-certificate');
  btn.textContent = 'GÉNÉRATION EN COURS...';
  btn.disabled = true;

  // Small delay so button state renders before heavy PDF work
  setTimeout(() => {
    try {
      const { jsPDF } = window.jspdf;
      const doc = new jsPDF({ orientation: 'landscape', unit: 'mm', format: 'a4' });
      const W = 297, H = 210;

      // ── Background ──
      doc.setFillColor(13, 13, 13);
      doc.rect(0, 0, W, H, 'F');

      // ── Top tricolor bar ──
      const barH = 5;
      doc.setFillColor(247, 127, 0);  doc.rect(0, 0, W / 3, barH, 'F');
      doc.setFillColor(255, 255, 255); doc.rect(W / 3, 0, W / 3, barH, 'F');
      doc.setFillColor(0, 154, 68);   doc.rect((W * 2) / 3, 0, W / 3, barH, 'F');

      // ── Bottom tricolor bar ──
      doc.setFillColor(247, 127, 0);  doc.rect(0, H - barH, W / 3, barH, 'F');
      doc.setFillColor(255, 255, 255); doc.rect(W / 3, H - barH, W / 3, barH, 'F');
      doc.setFillColor(0, 154, 68);   doc.rect((W * 2) / 3, H - barH, W / 3, barH, 'F');

      let y = barH + 14;

      // ── Label ──
      doc.setFont('courier', 'normal');
      doc.setFontSize(8);
      doc.setTextColor(100, 100, 100);
      doc.text('CERTIFICAT OFFICIEUX D\'IVOIRITÉ', W / 2, y, { align: 'center', charSpace: 2 });
      y += 12;

      // ── Title line 1 ──
      doc.setFont('courier', 'bold');
      doc.setFontSize(22);
      doc.setTextColor(255, 255, 255);
      doc.text('ES-TU VRAIMENT', W / 2, y, { align: 'center', charSpace: 1 });
      y += 10;

      // ── Title line 2 (orange) ──
      doc.setTextColor(247, 127, 0);
      doc.text('IVOIRIEN·NE ?', W / 2, y, { align: 'center', charSpace: 1 });
      y += 14;

      // ── Flag emoji (rendered as text — jsPDF may not render emoji; use text substitute) ──
      doc.setFontSize(28);
      doc.setTextColor(255, 255, 255);
      doc.text('🇨🇮', W / 2, y, { align: 'center' });
      y += 14;

      // ── Separator line ──
      doc.setDrawColor(42, 42, 42);
      doc.setLineWidth(0.3);
      doc.line(40, y, W - 40, y);
      y += 12;

      // ── Score ──
      doc.setFont('courier', 'bold');
      doc.setFontSize(36);
      doc.setTextColor(247, 127, 0);
      doc.text(`${state.score} / 12`, W / 2, y, { align: 'center' });
      y += 12;

      // ── Level ──
      const level = getLevel(state.score);
      doc.setFont('times', 'italic');
      doc.setFontSize(14);
      doc.setTextColor(200, 200, 200);
      doc.text(`\u00AB ${level.title} \u00BB`, W / 2, y, { align: 'center' });
      y += 16;

      // ── Dimension bars (condensed row) ──
      const dimScores = getDimensionScores();
      const totalBarWidth = W - 80;
      const barSpacing = 8;
      const barWidth = (totalBarWidth - barSpacing * (DIMENSIONS.length - 1)) / DIMENSIONS.length;
      const barStartX = 40;
      const barBlockH = 18;

      DIMENSIONS.forEach((dim, i) => {
        const x = barStartX + i * (barWidth + barSpacing);
        const pct = dimScores[i] / 100;

        // Track
        doc.setFillColor(42, 42, 42);
        doc.rect(x, y, barWidth, 4, 'F');

        // Fill
        if (pct > 0) {
          doc.setFillColor(247, 127, 0);
          doc.rect(x, y, barWidth * pct, 4, 'F');
        }

        // Label
        doc.setFont('courier', 'normal');
        doc.setFontSize(6);
        doc.setTextColor(100, 100, 100);
        doc.text(DIM_LABELS[dim], x + barWidth / 2, y + 8, { align: 'center' });
      });
      y += barBlockH + 6;

      // ── Separator ──
      doc.setDrawColor(42, 42, 42);
      doc.setLineWidth(0.3);
      doc.line(40, y, W - 40, y);
      y += 8;

      // ── Date + URL ──
      doc.setFont('courier', 'normal');
      doc.setFontSize(7);
      doc.setTextColor(80, 80, 80);
      const dateStr = new Date().toLocaleDateString('fr-FR', { day: '2-digit', month: 'long', year: 'numeric' });
      const urlStr = window.location.href.replace(/\/$/, '');
      doc.text(`Généré le ${dateStr} · ${urlStr}`, W / 2, y, { align: 'center' });
      y += 10;

      // ── Disclaimer ──
      doc.setFontSize(6.5);
      doc.setTextColor(60, 60, 60);
      doc.text(
        'Ce certificat est purement humoristique et non officiel. La Côte d\'Ivoire n\'est pas responsable de son contenu.',
        W / 2, H - barH - 6, { align: 'center' }
      );

      // ── Save ──
      doc.save(`certificat-ivoirien-${state.score}-sur-12.pdf`);

    } catch (err) {
      console.error('PDF generation error:', err);
    } finally {
      btn.textContent = 'TÉLÉCHARGER MON CERTIFICAT';
      btn.disabled = false;
    }
  }, 50);
}
```

- [ ] **Step 2: Test PDF generation**

Complete the quiz, reach results, click "TÉLÉCHARGER MON CERTIFICAT". A PDF named `certificat-ivoirien-X-sur-12.pdf` should download. Open it — verify dark background, tricolor bars top and bottom, score in orange, level in italic, dimension bars visible, disclaimer at bottom.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: pdf certificate generation with jsPDF — dark layout, tricolor bars, scores"
```

---

### Task 10: Toast system

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add toast functions**

Add after `generatePDF`:

```js
// ── Toast system ───────────────────────────────────────────────────────────
function showToast(message) {
  state.toastQueue.push(message);
  if (!state.toastActive) processToastQueue();
}

function processToastQueue() {
  if (state.toastQueue.length === 0) { state.toastActive = false; return; }
  state.toastActive = true;
  const msg = state.toastQueue.shift();
  const container = document.getElementById('toast-container');

  const toast = document.createElement('div');
  toast.className = 'toast';
  toast.textContent = msg;
  container.appendChild(toast);

  // Slide up
  requestAnimationFrame(() => {
    requestAnimationFrame(() => toast.classList.add('visible'));
  });

  // Hold then slide down
  setTimeout(() => {
    toast.classList.remove('visible');
    setTimeout(() => {
      toast.remove();
      processToastQueue();
    }, 350);
  }, 3000);
}
```

- [ ] **Step 2: Test toast manually in console**

Open browser console and type: `showToast('Test toast 🎉')` — a dark toast should slide up from the bottom, hold 3s, slide back down. Call it twice quickly: `showToast('First'); showToast('Second')` — they should queue and appear one after the other.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: toast system with queue — slide up/down animation"
```

---

### Task 11: Easter eggs

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add confetti helper and flag easter egg**

Add after `processToastQueue`:

```js
// ── Easter eggs ────────────────────────────────────────────────────────────

// Shared confetti helper — tricolor colors
function triggerConfetti() {
  confetti({
    particleCount: 120,
    spread: 80,
    origin: { y: 0.5 },
    colors: ['#F77F00', '#FFFFFF', '#009A44'],
    ticks: 200,
  });
}

// EE1 — Flag click (5×)
function handleFlagClick() {
  state.flagClickCount++;
  if (state.flagClickCount >= 5) {
    state.flagClickCount = 0;
    const flag = document.getElementById('flag-trigger');
    flag.style.animation = 'flagDance 3s ease forwards';
    flag.addEventListener('animationend', () => { flag.style.animation = ''; }, { once: true });
    showToast('Vive la Côte d\'Ivoire ! 🇨🇮🎉');
  }
}
```

- [ ] **Step 2: Add flagDance keyframe CSS**

Inside `<style>`:

```css
@keyframes flagDance {
  0%   { transform: rotate(0deg) scale(1); }
  15%  { transform: rotate(20deg) scale(1.3); }
  30%  { transform: rotate(-18deg) scale(1.2); }
  45%  { transform: rotate(16deg) scale(1.3); }
  60%  { transform: rotate(-14deg) scale(1.1); }
  75%  { transform: rotate(10deg) scale(1.2); }
  90%  { transform: rotate(-6deg) scale(1.1); }
  100% { transform: rotate(0deg) scale(1); }
}

/* Emoji dance for Konami */
@keyframes emojiDance {
  0%, 100% { transform: rotate(0deg) scale(1); }
  25%  { transform: rotate(30deg) scale(1.4); }
  50%  { transform: rotate(-30deg) scale(0.8); }
  75%  { transform: rotate(20deg) scale(1.3); }
}
.dancing { animation: emojiDance 2s ease; }
```

- [ ] **Step 3: Add global keydown listener for EE2 (Drogba) + EE4 (Konami)**

Add after `handleFlagClick`:

```js
// EE2 — "drogba" typed anywhere + EE4 — Konami code
const KONAMI_SEQUENCE = [
  'ArrowUp','ArrowUp','ArrowDown','ArrowDown',
  'ArrowLeft','ArrowRight','ArrowLeft','ArrowRight',
  'b','a'
];

document.addEventListener('keydown', (e) => {
  const key = e.key.toLowerCase();

  // Rolling char buffer (10 chars) for "drogba"
  state.keyBuffer = (state.keyBuffer + key).slice(-10);
  if (state.keyBuffer.includes('drogba')) {
    state.keyBuffer = '';
    triggerConfetti();
    showToast('DROGBA ! 🐉 Le Boss de Chelsea et d\'Abidjan !');
  }

  // Konami buffer (10-key sequence)
  state.konamiBuffer.push(e.key);
  if (state.konamiBuffer.length > 10) state.konamiBuffer.shift();
  if (state.konamiBuffer.join(',') === KONAMI_SEQUENCE.join(',')) {
    state.konamiBuffer = [];
    // Dance all visible emojis
    document.querySelectorAll('.q-emoji, #flag-trigger').forEach(el => {
      el.classList.remove('dancing');
      el.offsetHeight; // reflow
      el.classList.add('dancing');
      el.addEventListener('animationend', () => el.classList.remove('dancing'), { once: true });
    });
    showToast('🕺 Coupé-Décalé activé ! Douk Saga approuve.');
  }
});
```

- [ ] **Step 4: Verify all 5 easter eggs**

Test each one:

1. **Flag dance:** Click 🇨🇮 flag 5 times → flag oscillates, toast appears.
2. **Drogba mode:** During quiz, type "drogba" on keyboard → confetti + toast.
3. **Score parfait:** Complete quiz with 12/12 → confetti on result, glow on badge, bonus message. (To test quickly: in console before quiz ends, set `state.score = 11` then answer last question correctly.)
4. **Konami:** Press ↑ ↑ ↓ ↓ ← → ← → B A → emojis dance, toast appears.
5. **Impatient:** On a question, wait 45 seconds without clicking → message appears under card, disappears after 4s.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: all 5 easter eggs — flag dance, drogba confetti, perfect score, konami, impatient"
```

---

### Task 12: Responsive polish + accessibility

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Verify mobile layout at 375px**

Open DevTools → toggle device toolbar → set width to 375px. Check each screen:
- Landing: title doesn't overflow, button is full width, font sizes feel right
- Quiz: card fits, option buttons are tall enough (44px+), no horizontal scroll
- Result: radar chart fits, bars render correctly

- [ ] **Step 2: Add focus styles for keyboard navigation**

Add inside `<style>`:

```css
/* Keyboard focus */
.btn:focus-visible,
.option-btn:focus-visible {
  outline: 2px solid var(--orange);
  outline-offset: 2px;
}
```

- [ ] **Step 3: Add aria-labels to icon-only interactive elements**

In the HTML, update the flag trigger:
```html
<div id="flag-trigger" ... aria-label="Drapeau ivoirien" role="button" tabindex="0" ...>🇨🇮</div>
```

Also add `aria-live="polite"` to the toast container:
```html
<div id="toast-container" aria-live="polite"></div>
```

- [ ] **Step 4: Check no console errors on any screen transition**

Open console, run full quiz, verify zero errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "fix: responsive polish, focus styles, aria labels"
```

---

### Task 13: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Create README**

```markdown
# Es-tu vraiment Ivoirien·ne ?

A standalone viral web quiz testing your knowledge of Ivorian culture across 12 questions and 6 cultural dimensions. No backend, no build step — everything lives in a single `index.html`.

## What it does

- 12 questions on language, food, transport, music, mindset, and Ivorian pride
- Radar chart showing your score across 6 cultural dimensions
- 5 hidden easter eggs
- PDF certificate generation (client-side, no server)

## Run locally

```bash
python3 -m http.server 3000
# then open http://localhost:3000
```

Or just open `index.html` directly in your browser.

## Deploy

**Netlify:** Drag and drop the project folder onto netlify.com/drop

**Vercel:**
```bash
npx vercel
```

**GitHub Pages:** Push to a repo, enable Pages from the root of `main`.

No build step required. The single `index.html` is the entire app.

## Environment

No environment variables. No API keys. No server.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with deploy instructions"
```

---

## Self-Review

**Spec coverage check:**
- Landing cinematic animation ✓ (Task 3)
- 12 questions with dimensions ✓ (Task 4)
- Quiz screen with feedback ✓ (Tasks 5–6)
- Result screen with radar + bars ✓ (Tasks 7–8)
- PDF certificate dark design ✓ (Task 9)
- Toast system ✓ (Task 10)
- All 5 easter eggs ✓ (Task 11)
- Mobile responsive ✓ (Task 12)
- README ✓ (Task 13)

**Placeholder scan:** No TBDs, no "implement later", all code blocks present.

**Type consistency:** `state.radarChart` used in Tasks 8 and 8 (destroy on restart) — consistent. `getDimensionScores()` defined in Task 8 and used in both Task 8 and Task 9 — consistent. `getLevel()` defined in Task 4 and used in Tasks 8 and 9 — consistent. `DIMENSIONS` and `DIM_LABELS` defined in Task 4, used throughout — consistent.
