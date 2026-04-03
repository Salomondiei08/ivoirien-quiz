# Design Doc — "Es-tu vraiment Ivoirien·ne ?"
**Date:** 2026-04-03
**Status:** Approved

---

## Overview

A standalone static web quiz testing Ivorian cultural knowledge across 12 questions. Single `index.html` file, no build step, no backend. Deployable via drag & drop on Netlify, GitHub Pages, or Vercel. Inspired visually by amiatechbro.com — dark, cinematic, editorial.

---

## Stack

- **Runtime:** Vanilla HTML/CSS/JS — no framework
- **CDN dependencies (3 only):**
  - Chart.js — radar chart on result screen
  - jsPDF — PDF certificate generation (client-side)
  - canvas-confetti — easter egg confetti effects
- **Fonts:** Google Fonts CDN — monospace (e.g. Space Mono or IBM Plex Mono) for labels/titles, serif (e.g. Lora or Playfair Display) for descriptions/feedback
- **Output:** Single `index.html` + `README.md`

---

## Visual Identity

| Token | Value |
|---|---|
| Background | `#0D0D0D` (near-black) |
| Card surface | `#161616` |
| Card border | `#2A2A2A` (1px, subtle) |
| Primary accent | `#F77F00` (Ivorian orange) |
| Secondary accent | `#009A44` (Ivorian green) |
| Text primary | `#FFFFFF` |
| Text secondary | `#888888` |
| Border radius | `4px` max (quasi-square) |
| Shadows | None |
| Gradients | None |

**Typography:**
- Labels, headings, buttons: monospace, uppercase, `letter-spacing: 0.08em`
- Descriptions, feedback, body copy: serif, sentence case

**Recurring graphic element:** 3px tricolor bar (orange | white | green) pinned to the top of every screen.

---

## Application Structure

Three `<section>` elements inside `<body>`. Only one visible at a time via `display: none/block`. No routing library needed.

```
#screen-landing   → Phase 1
#screen-quiz      → Phase 2
#screen-result    → Phase 3
```

Global state object:

```js
const state = {
  currentQuestion: 0,     // 0–11
  score: 0,
  answers: [],            // [{questionId, dimension, correct, selectedIndex}]
  answered: false,
  phase: 'landing',       // 'landing' | 'quiz' | 'result'

  // Easter egg trackers
  flagClickCount: 0,
  keyBuffer: '',          // rolling 10-char buffer for 'drogba' + Konami
  questionTimer: null,    // setTimeout ref for impatient EE
  toastQueue: [],         // queued toast messages
  toastActive: false,
};
```

---

## Phase 1 — Landing

**Animation sequence (sequential, not simultaneous):**
1. `"YOUR IVOIRITÉ"` typewriter letter-by-letter, cursor blinks then disappears
2. `"Es-tu vraiment"` slides in from left
3. `"Ivoirien·ne ?"` slides in from left, 150ms after line 1, rendered in `#F77F00`
4. Short description fades up from below
5. `"COMMENCER LE TEST →"` button fades in
6. Discreet note about easter eggs fades in last

**Background:** `<canvas>` particle system — ~20 dots drifting slowly. Mix: some orange (`#F77F00` at 30% opacity), some green (`#009A44` at 20% opacity), most near-invisible gray. Uses `requestAnimationFrame` loop. Particles wrap around edges.

**Flag easter egg trigger:** 🇨🇮 emoji in header, click listener attached.

---

## Phase 2 — Quiz

**Layout per question:**
- Tricolor bar (top)
- Progress bar: `((currentQuestion + 1) / 12) * 100%` width, animated via CSS transition
- Label row: `"QUESTION X / 12"` left, `"SCORE : Y"` right — both monospace
- Question card (slide-up animation on each new question):
  - Question number + dimension tag (dimension invisible to user — stored in data only)
  - Emoji (large, centered above question text)
  - Question text (serif)
  - 4 option buttons (monospace, 44px min-height, full width)
- After answer: buttons disabled, correct answer → green border + checkmark, wrong selection → orange border + X, feedback text fades in below card
- "QUESTION SUIVANTE →" button appears 400ms after answer

**Dimension tags per question (for radar calculation):**

| Q | Dimension |
|---|---|
| 1 | langue |
| 2 | bouffe |
| 3 | débrouillardise |
| 4 | tempo |
| 5 | culture |
| 6 | langue |
| 7 | culture |
| 8 | bouffe |
| 9 | culture |
| 10 | débrouillardise |
| 11 | langue |
| 12 | fierté |

Radar axis scores = correct answers per dimension / total questions in that dimension, expressed as percentage.

---

## Phase 3 — Result

**Block arrival sequence (staggered with short delays):**

1. **Score header:** `"You scored X / 12"` — score in orange badge, then `"That puts you in the [LEVEL] range."` in italic serif
2. **Radar chart:** Chart.js, 6 axes, transparent background, dark grid lines, gray monospace labels, animated fill (~1s)
3. **Dimension bars:** 6 rows, each `label — bar — percentage`. Bars animate left-to-right, staggered 120ms apart. Bar color: `#F77F00` (or a muted blue as per amiatechbro aesthetic — orange preferred for brand consistency)
4. **Profile description:** 2–3 sentences in serif, centered
5. **Buttons:** `"TÉLÉCHARGER MON CERTIFICAT"` (orange filled) + `"Recommencer"` (outline)

**Level thresholds:**

| Score | Level |
|---|---|
| 0–2 | toubabou perdu |
| 3–5 | diaspora confuse |
| 6–8 | potentiellement ivoirien·ne |
| 9–10 | vrai·e ivoirien·ne |
| 11–12 | pur·e abidjanais·e |

---

## Phase 4 — PDF Certificate

Generated client-side via jsPDF. Format: A4 landscape (297mm × 210mm). Dark background (`#0D0D0D`). Fully vectorial — no html2canvas.

**Layout top-to-bottom:**
1. Tricolor bar (top, 4mm height): orange | white | green thirds
2. Small gray monospace label: `"CERTIFICAT OFFICIEUX D'IVOIRITÉ"`
3. Title line 1 white: `"ES-TU VRAIMENT"`, line 2 orange: `"IVOIRIEN·NE ?"`
4. Flag emoji centered (via text, large font)
5. Horizontal separator line (gray, 0.5px)
6. Score in large orange monospace: `"X / 12"`
7. Level in white serif italic with guillemets: `« pur·e abidjanais·e »`
8. 6 condensed dimension bars on one row (using `doc.rect()`)
9. Generation date + URL (`window.location.href`)
10. Tricolor bar (bottom)
11. Disclaimer in small gray serif: `"Ce certificat est purement humoristique et non officiel. La Côte d'Ivoire n'est pas responsable de son contenu."`

**Download filename:** `certificat-ivoirien-X-sur-12.pdf`
**Button state during generation:** text changes to `"GÉNÉRATION EN COURS..."`, disabled.

---

## Easter Eggs

All 5 are implemented. Toast component is shared across all.

| # | Trigger | Effect |
|---|---|---|
| 1 | Click 🇨🇮 5× | Flag oscillates 3s, toast: "Vive la Côte d'Ivoire ! 🇨🇮🎉" |
| 2 | Type "drogba" (any time) | canvas-confetti tricolor 3s, toast: "DROGBA ! 🐉 Le Boss de Chelsea et d'Abidjan !" |
| 3 | Score 12/12 | canvas-confetti cascade, badge pulses with orange glow, bonus message below description |
| 4 | Konami code (↑↑↓↓←→←→BA) | All emojis spin 2s, toast: "🕺 Coupé-Décalé activé ! Douk Saga approuve." |
| 5 | 45s on same question | Discreet message under card: "⏰ Tu réfléchis trop mon gars, c'est pas l'examen BEPC !" — auto-disappears 4s |

**Key buffer strategy:** Single global `keydown` listener. Rolling buffer of last 10 chars (lowercase). After each keydown, check if buffer ends with `"drogba"` OR if last 10 chars match Konami sequence (stored as a separate array tracker).

**Toast system:**
- Queue array. One toast visible at a time.
- Toast slides up from bottom, holds 3s, slides back down.
- On dismiss, next in queue is shown.
- Style: dark bg, white monospace text, 1px border `#2A2A2A`, `border-radius: 4px`.

---

## Responsive

- Mobile-first layout
- Max container width: `480px`, centered on desktop
- All interactive elements: `min-height: 44px`
- `font-size` minimum: `14px`
- No horizontal overflow on any screen
- Radar chart: `width: 100%` with `aspect-ratio: 1` — scales to container
- Canvas particles: resizes on `window.resize`

---

## File Structure

```
ivorianform/
├── index.html        ← entire app (HTML + CSS + JS inline)
├── README.md         ← deploy instructions
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-04-03-ivoirien-quiz-design.md
```

---

## Out of Scope

- No social share button (PDF certificate replaces sharing)
- No analytics or tracking
- No server-side anything
- No dark/light toggle (dark only by design)
- No localization (French only)
