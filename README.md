# ⚡ flashread

A single-file web app that flashes text one word at a time with a red anchor letter fixed at screen center. Paste text or a URL, pick a speed, and read at 200–1200 words per minute.

No build step, no backend, no tracking. One `index.html` — works on desktop and mobile.

---

## What this is

An implementation of **RSVP** (Rapid Serial Visual Presentation), a reading-psychology technique that presents text one unit at a time at a fixed screen position, eliminating the saccadic eye movements used in normal reading. RSVP has been used in experimental reading research since at least Forster (1970) and is the basis of apps like Spritz and Spreeder.

The app is suited for: clearing a backlog of articles, reading familiar material quickly, reading on a phone, or doing speed-reading practice. It's less suited for complex material where regression and parafoveal preview (both removed by RSVP) contribute to comprehension.

---

## Quick start

1. Open `index.html` or visit the deployed page.
2. Paste text, or switch to **From URL** and paste a link.
3. Choose speed, font, weight, chunk size.
4. Click **Start reading**.

Tap or click the word to pause. Preferences, input text, and reading position persist in `localStorage`.

---

## Features

### Reading engine
- **ORP-anchored words** — each word renders with its Optimal Recognition Point letter in red at screen center. Subsequent words align so the red letter stays at a fixed x-coordinate.
- **Chunking** (1 / 2 / 3 words per unit) — displays multiple words per frame. Boundaries: never cross sentence-ending punctuation or paragraph breaks; cap at 18 characters. Timing scales with chunk size so effective WPM is preserved.
- **Paragraph micropauses** — blank-line boundaries in the source insert a silent unit at 2.5× a word slot (floored at 200ms).
- **Inter-word blank frame** — 5% of each word's slot is blank (floored at 16ms, one 60Hz frame) to prevent ghosting at high WPM.
- **Ease-in ramp** — the first 5 words of a play session run 1.6× slower, lerping to target WPM.
- **Punctuation pauses** — sentence marks 1.7×, commas/semicolons 1.3×, words ≥9 chars 1.08×.
- **Drift-corrected scheduler** — each tick schedules against an absolute `performance.now()` target rather than a chained relative `setTimeout`, which drifts cumulatively.

### UI
- **Parafoveal flankers** (optional) — prev/next chunk shown dim. Opacity decays with WPM.
- **Transient WPM overlay** — large number above the top guide line when speed changes.
- **Corner badges** — current WPM (bottom-left), time remaining (bottom-right), word counter (top-right). Persist when the control bar auto-hides.
- **Progress line** at viewport top.
- **Book gauge** — estimated completion time for ~20 cataloged books at the current WPM; updates live.
- **Two-line media readout** — elapsed / total, flanking the scrub slider.
- **Dark / light theme**, persisted.

### Persistence
- **Resume per text** — position saved under a hash of the content, LRU-capped at 50 entries. Rewinds 2 chunks on resume for context.
- **Input persistence** — textarea, URL, active tab, and all settings survive refresh.
- **Auto-pause** on `visibilitychange` (tab hidden) and `blur` (window loses focus).

### Mobile
- **Tap** word area — play/pause
- **Horizontal swipe** — seek (one chunk per 24px)
- **Vertical swipe** — speed (±25 wpm per 18px, up = faster)
- Uses `100dvh` sizing, `touch-action: none` on the reading surface, `color-scheme` CSS for native control theming
- PWA-capable meta tags — Add to Home Screen on iPhone gives a chrome-less reader

### Fonts

Ten options, each previewed in its own face in the picker:

| Serif | Sans | Reading-research |
|---|---|---|
| Iowan Old Style | System Sans | Atkinson Hyperlegible |
| New York | Helvetica Neue | Lexend |
| Charter | Verdana | |
| Georgia | | |
| Palatino | | |

Three weights (Regular / Semibold / Bold). Letter-spacing scales with weight (0.025em → 0.042em).

### Keyboard

| Key | Action |
|---|---|
| `Space` | play / pause |
| `← →` | seek one chunk |
| `Shift + ← →` | seek 25 chunks |
| `↑ ↓` | speed ±25 wpm |
| `Shift + ↑ ↓` | font size ±8px |
| `Cmd/Ctrl + B` | cycle weight |
| `F` | fullscreen toggle |
| `Home` / `End` | jump to start / end |
| `Esc` | exit fullscreen, or back to input |

---

## Design choices

### ORP placement

The red letter position is set per-word by a lookup table:

| Word length | Index |
|---|---|
| 1 | 0 |
| 2–5 | 1 |
| 6–9 | 2 |
| 10–13 | 3 |
| 14+ | 4 |

This approximates the **Optimal Viewing Position**, where fixation time is minimized at roughly 25–30% from a word's start rather than its geometric center.

- Rayner, K. (1975). *The perceptual span and peripheral cues in reading.* Cognitive Psychology, 7(1), 65–81.
- Rayner, K., & Pollatsek, A. (1989). *The Psychology of Reading.* Prentice-Hall.
- OVP formalization in the 1980s–90s by Kevin O'Regan, Thierry Nazir, and Marc Brysbaert.

The 5-bucket step function is the same table used by the Spritz implementation.

### Chunking

Single-word RSVP eliminates all parafoveal context. 2–3 word chunks reintroduce some, with the ORP anchored on the middle word. Chunks never cross sentence-ending punctuation or paragraph breaks; they also cap at 18 characters. Chunk-size timing is linear — N words = N × single-word slot — so the effective WPM set by the user is preserved.

### Context preview (flankers)

Dim prev and next chunk on either side of the anchor. Research on parafoveal preview suggests ~30–50ms per saccade is saved in normal reading by pre-processing the next word. RSVP eliminates this; optional flankers restore some of it.

Opacity follows a quadratic decay with WPM to reduce saccade temptation at high speeds:

| WPM | Flanker opacity |
|---|---|
| 200 | 0.45 |
| 450 | 0.28 |
| 750 | 0.10 |
| 1200 | 0.03 |

### Letter tracking

Positive letter-spacing reduces visual crowding between adjacent letters, relevant in the parafovea where letters flanking the ORP live during RSVP.

- Zorzi, M., Barbiero, C., Facoetti, A., et al. (2012). *Extra-large letter spacing improves reading in dyslexia.* PNAS, 109(28), 11455–11459.
- Work by Manuel Perea and collaborators on inter-letter spacing in visual-word recognition; gains in normal readers plateau around ~0.025em and reverse past ~0.05em.
- Work by Susana Chung on crowding and critical spacing in central and peripheral vision.

Current values: Regular 0.025em, Semibold 0.034em, Bold 0.042em. Flankers 0.030em. Scaled with weight because thicker strokes visually compress perceived letter gaps.

### Inter-word blank frame

At high WPM, the previous word can still be on the pixel (LCD transition time + retinal persistence) when the next arrives. A blank frame between words forces a hard edge. Floored at 16ms because at 1200 wpm a 5% ratio would be 2.5ms, shorter than a display frame.

### Drift-corrected scheduler

`setTimeout(fn, 50)` fires no earlier than 50ms, typically a few ms late. Over 1000 ticks at 50ms, the drift accumulates to several percent below set WPM. Each tick now schedules against an absolute `performance.now()` target, so cumulative timing remains accurate regardless of per-tick jitter.

### Weight option

Bolder strokes reduce the perceived halo at high screen contrast (optical aberration + screen halation + anti-aliasing) because the halo is an edge phenomenon and bolder letters have lower edge-to-ink ratio. Also scales up letter-spacing since thicker strokes visually compress letter gaps.

### Color

`#0b0b0c` / `#f4f4f5` in dark mode; `#fafaf7` / `#1a1a1c` in light. Both pairs retreat slightly from pure endpoints to reduce maximum-contrast edge artifacts while keeping contrast above WCAG AAA.

### Font selection

Five serifs covering system-native (Iowan, New York), screen-designed (Charter, Georgia), and classical (Palatino). Three sans covering system, Helvetica, and Verdana (largest x-height of common sans-serifs). Two reading-research-motivated faces: Atkinson Hyperlegible (Braille Institute) and Lexend.

---

## Privacy and data

- All preferences, input text, URLs, and resume positions live in `localStorage`. No backend.
- URL-fetch uses two public services, in order:
  1. **Jina Reader** (`r.jina.ai`) — returns clean markdown extraction. No API key.
  2. **allorigins.win** — CORS proxy, used as fallback. HTML is stripped locally.
- Google Fonts is requested only if Atkinson Hyperlegible or Lexend is selected.
- No analytics, cookies, or tracking.

To avoid third-party fetches entirely: use a self-hosted CORS proxy by swapping the two URLs in `fetchUrlText`, and pick any of the other eight fonts.

---

## Hosting on GitHub Pages

```bash
git init && git add . && git commit -m "init"
gh repo create flashread --public --source=. --push
gh api -X POST repos/:owner/flashread/pages -f source='{"branch":"main","path":"/"}'
```

The site will be at `https://<username>.github.io/flashread/`.

---

## Tuning constants

All in `index.html`:

| Constant | Default | Purpose |
|---|---|---|
| `BLANK_RATIO` | 0.05 | Fraction of each word's slot spent blank |
| `BLANK_MIN_MS` | 16 | Minimum blank duration (one 60Hz frame) |
| `EASE_IN_STEPS` | 5 | Words affected by ease-in ramp |
| `EASE_IN_FACTOR` | 1.6 | Start-of-session slow-factor |
| `CHUNK_MAX_CHARS` | 18 | Cap for multi-word chunk length |
| `RESUME_LRU_CAP` | 50 | Texts that remember their resume position |

The WPM-to-flanker-opacity curve is in `flankerOpacityForWpm()`.

---

## License

Unlicensed; no warranty.
