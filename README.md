# Es-tu vraiment Ivoirien·ne ?

A standalone viral web quiz testing your knowledge of Ivorian culture — 12 questions, 6 cultural dimensions, radar chart results, and a downloadable PDF certificate. No backend, no build step, no dependencies to install.

## What it does

12 questions spanning language (Nouchi), food, transport, music, mindset, and national pride. Your answers map onto 6 cultural dimensions shown as a radar chart. Depending on your score (0–12) you receive one of 5 levels from "Toubabou perdu" to "Pur·e Abidjanais·e". You can download a dark-themed PDF certificate of your result.

There are also 5 hidden easter eggs.

## Run locally

```bash
python3 -m http.server 3000
```

Then open `http://localhost:3000` in your browser. Or just open `index.html` directly — no server required.

## Deploy

**Netlify:** Drag the project folder onto [netlify.com/drop](https://app.netlify.com/drop)

**Vercel:**
```bash
npx vercel
```

**GitHub Pages:** Push to a repo, go to Settings → Pages, set source to the root of `main`.

No build step. The single `index.html` is the entire app.

## Environment

No environment variables. No API keys. No server needed.
