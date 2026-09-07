# Qoldau AI

Next.js (App Router) + TypeScript + Tailwind implementation of the Qoldau AI
platform per `PROMPT.md`.

## Run locally

```bash
npm install
npm run dev
```

Open http://localhost:3000. The camera features need HTTPS or `localhost` —
both work for dev; a real deployment needs HTTPS (Render provides it
automatically).

## What's implemented

- Header with functional links, a "Переводчик" dropdown (Жест → текст /
  Текст → жест), and a KZ/RU/ENG language switcher. All the demo-data menu
  items from the reference site (books, police, login, "15+/3.05" counters)
  are removed, not just hidden.
- Home page: hero, the 6-card "barriers" grid, the 3-card tools section, and
  the interface CTA — all copy pulled from `messages/*.json`, scroll fade-in
  via a small `IntersectionObserver` wrapper (no extra animation library).
- Dictionary page backed by `data/dictionary.ts` — 20 candidate words, **every
  entry starts `verified: false`** and shows an "unverified" badge with a
  shared placeholder image instead of a specific rendered sign.
- Translator page with both directions:
  - **Text → sign**: looks a typed word up in the dictionary (kk/ru/en) and
    shows its render, or a clear "not in the dictionary" message if it isn't
    found.
  - **Sign → text**: really requests `getUserMedia`, loads MediaPipe's
    `HandLandmarker` on demand (dynamic import, doesn't block first paint),
    and draws the 21 hand landmarks live on a canvas over the video. It
    buffers ~18 frames and would compare them against reference templates via
    cosine similarity + DTW (`lib/gestureMatcher.ts`) — but see the honesty
    note below.
- About page: mission/values/vision + the three team cards, laid out with
  card 3 centered on its own row as described.
- A lightweight custom i18n context (`components/LanguageProvider.tsx`) reads
  `messages/kk.json`, `messages/ru.json`, `messages/en.json`. It's simpler
  than `next-intl` (no `/kk/…` URL segments) but meets the actual requirement:
  no hardcoded UI strings, and switching the dropdown re-renders the whole
  site. Swap in `next-intl` later if you specifically need localized URLs.

## Honesty note: sign recognition is not classifying anything yet

This matters enough to repeat outside a code comment. The camera pipeline is
real — camera access, live hand-landmark detection, and a normalize →
cosine-similarity → DTW matching pipeline are all wired up in
`lib/gestureMatcher.ts` and `components/translator/GestureToText.tsx`. What's
missing is the one thing that can't be faked responsibly: **verified
reference recordings of a fluent ҚЖТ signer performing each of the 20
words.** `referenceTemplates` in `lib/gestureMatcher.ts` is intentionally
empty, so the UI always shows "Модель распознавания пока не подключена"
rather than a made-up result — exactly per the spec's instruction not to fake
a demo classifier. It does show a live "✓ hand landmarks detected" indicator
so you can confirm the camera + MediaPipe half of the pipeline genuinely
works on your device.

To finish Variant A for real:
1. Record each of the 20 signs on camera, several repetitions, a couple of
   angles — ideally performed by (or corrected by) a fluent ҚЖТ signer.
2. Save each performance's landmark sequence (the same normalized shape
   `lib/gestureMatcher.ts` expects) into `referenceTemplates`.
3. Flip that word's `verified: true` in `data/dictionary.ts` only once a
   qualified person has confirmed the *sign itself* is correct — that's a
   separate check from "the recording matches itself."

A dataset-recording admin screen (`/admin/record-dataset`) is the natural way
to capture #1/#2 but isn't built in this pass — happy to add it next if
useful.

## What still needs real assets from you

- **Team photos**: place square, shoulders-up crops at
  `public/team/1-berikkyzy.jpg`, `public/team/2-erlankyzy.jpg`,
  `public/team/3-kurakbaeva.jpg`. Until then the About page shows empty
  circles (broken image swallowed gracefully, not a fake photo).
- **Dictionary sign renders**: once a sign is verified, replace
  `/public/signs/placeholder.svg` for that word with a real per-word
  static render (3D hand-pose image or clean line art) and flip `verified`
  to `true` in `data/dictionary.ts`.

## Deploy to Render.com

1. Push this repo to GitHub.
2. On Render: **New +** → **Web Service** → select the repo.
3. Build command: `npm install && npm run build`
4. Start command: `npm run start`
5. Environment: Node (latest LTS).
6. Render issues HTTPS automatically — required for `getUserMedia` on
   mobile Safari.

## Custom domain via easydomain

1. In Render, open the service → **Settings** → **Custom Domains** → add
   your domain; Render shows the exact target host to point at.
2. In easydomain's DNS settings, add a CNAME record for your domain pointing
   at that host.
3. Wait for DNS propagation (10–60 min), then Render verifies automatically.

## Before calling the dictionary "final"

Show the 20 candidate signs and their renders to a certified ҚЖТ interpreter
before flipping any `verified` flag to `true` — an unverified sign shown as
if it were correct is worse than an honest "under review" badge.
