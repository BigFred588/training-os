# CLAUDE.md — Training OS

> This file is for Claude Code and any AI assistant working on this repo.
> Read it fully before touching anything.

---

## Project Overview

Training OS is a single-file PWA (Progressive Web App) for a personalised calisthenics program built on the **2-5-15 Framework** (2 phases minimum, 5 sets per muscle group, 15 total sets per session).

The app is used daily on a Android phone (Pixel, GrapheneOS) by a single user (Ed, Perth WA). It tracks a 40-week program across 3 phases: Foundation → Strength → Skill. Primary goal is a strict ring muscle-up.

**It is not a SaaS product. It is a personal training tool. Optimise for the user's specific needs, not for generalisation.**

---

## Tech Stack

| Layer | What |
|-------|------|
| Framework | React 18 (no Next.js, no CRA) |
| Build | Webpack 5 + Babel |
| Styling | Inline React styles only — no CSS files, no Tailwind, no CSS modules |
| Storage | localStorage only — no backend, no database, no auth |
| Deployment | GitHub Pages → training.collict.me via Cloudflare DNS CNAME |
| Runtime | Single HTML file, fully self-contained, no CDN dependencies |

---

## File Structure

```
training-os/
├── index.html              # The entire app — compiled, self-contained, 250KB
├── CLAUDE.md               # This file
```

**Source files live locally on the build machine, not in this repo:**

```
/home/claude/
├── part1.jsx               # Constants, benchmark tests, progression logic
├── part2.jsx               # Skill blocks, progression unlocks
├── part3.jsx               # Phase data (all 3 phases, all days, all exercises)
├── part4.jsx               # CAT_STYLE, CHECKIN_PROMPTS, utility components
├── part5.jsx               # App state, Dashboard component
├── part6.jsx               # Program component
├── part7.jsx               # Log component
├── part8.jsx               # Benchmark, CheckIn, App return
├── entry.jsx               # Webpack entry — concatenates parts + adds React imports
├── webpack.config.cjs      # Webpack config
├── dist/bundle.js          # Compiled output (not committed)
```

**Build command:**
```bash
cat part1.jsx part2.jsx part3.jsx part4.jsx part5.jsx part6.jsx part7.jsx part8.jsx > training-app.jsx
# then prep entry.jsx and run:
node_modules/.bin/webpack --config webpack.config.cjs
# then wrap dist/bundle.js in the HTML shell to produce index.html
```

---

## Architecture

### Data Flow
- All state lives in React via `useStorage` hook
- `useStorage` reads from localStorage on init, writes on every set call
- No server, no sync, no accounts — data is device-local only
- Export/import via JSON backup in the Data tab

### Key State Variables (App level)
```
phaseIdx        — current phase (0/1/2)
currentWeek     — week number (1-40)
dayId           — current day (A/B/C)
milestones      — completed milestones map
workoutLogs     — all session logs keyed as p{phase}_d{day}_w{week}
benchmarks      — benchmark results keyed as b{point}_{testId}
checkins        — array of weekly check-in objects
dailyWeights    — Garmin weekly averages keyed as w{week}_avg
```

### Session State (also App level — important)
```
sessionStarted, sessionFinished, earlyConfirm
warmupChecks, skillChecks, coreChecks, coolChecks, neckChecks
sleepRating, sessionNotes, sessionCompleted
```

**Session state MUST stay at App level.** If moved into the Log component it will reset every time a re-render occurs (e.g. rest timer ticking). This was a hard-won fix — do not move it back.

### Log Key Format
- Workout logs: `p${phaseId}_d${dayId}_w${week}`
- Weight per set: `w_${phaseId}_${dayId}_${week}_${setIndex}`
- Benchmark: `b${benchmarkPoint}_${testId}`
- Weekly weight: `w${week}_avg`

---

## Program Structure

### The 2-5-15 Framework
- **15 sets per session** — hard cap, time constraint is 60 min in/out
- **3 skill sets** — count toward the 15 (not extra)
- **12 working sets** — the remainder
- Active recovery between every set — antagonist muscle, 20-30s

### 3 Training Days
- **Day A (Tuesday)** — Pull heavy, Legs, Hinge, Bicep
- **Day B (Thursday)** — Hinge heavy, Skill, Push, Pull, Carry
- **Day C (Saturday)** — Push, Pull heavy, Legs, Bicep, Carry

### Session Flow (Log tab)
Start → Sleep quality → Day selector → ▶ Start Session → 🔥 Warm-up → 🎯 Skill block → 💪 Working sets → 🧱 Core → ❄️ Cool-down → 💆 Neck work → ✓ Complete

Each stage unlocks the next. Early finish available at every stage.

### 3 Phases
- **Phase 1** (Weeks 1-15) — Foundation, RPE 7-8, bodyweight/light load
- **Phase 2** (Weeks 16-28) — Strength, RPE 8-9, weighted progressions
- **Phase 3** (Weeks 29-40) — Skill, RPE 8-9, advanced statics, muscle-up pathway

### Deload
Every 5th week — 10 sets, RPE 6, statics at 60% hold. Triggered automatically by `currentWeek % 5 === 0`.

---

## Key Conventions

### Styling
- All styles are inline React style objects
- Colour palette: `#1a1a2e` (dark navy), `#2d7a5f` (green), `#4a4a9a` (purple), `#a04a2a` (rust), `#e8c97a` (gold)
- Phase colours: Phase 1 = `#2d7a5f`, Phase 2 = `#4a4a9a`, Phase 3 = `#a04a2a`
- Category colours defined in `CAT_STYLE` object — always use this, never hardcode category colours

### Inputs
- All text/number inputs use `defaultValue` + `onBlur` pattern to prevent keyboard dismiss on mobile
- **Exception: benchmark inputs** — use controlled `value` + `onChange` calling `setBench` directly. This was reverted after multiple failed attempts at uncontrolled approach. The keyboard dismiss is accepted as a known tradeoff.
- `setBench` builds the new state object from current React state, then writes to localStorage — never reads from localStorage inside setBench

### Components
- All major components (Dashboard, Program, Log, Benchmark, CheckIn) are defined as arrow functions inside App()
- **Exception: DataBackup, RestTimer, SessionTimer, SkillBlockCard, RPEBadge, BenchInput** — these are defined outside App() to prevent re-mount on re-render
- If a component has its own state and is rendered inside a frequently-updating parent, define it outside App()

### useStorage Hook
```js
function useStorage(key, fallback) {
  // Handles "undefined" and "null" strings from corrupted localStorage
  // Cleans up corrupted values on init
  // Never writes fallback to localStorage on init — only writes on set()
}
```
Always use this for persistent state. Never call `localStorage` directly from components except in `setBench` and `BenchInput`.

---

## Known Quirks and Hard-Won Fixes

### 1. Benchmark inputs keyboard dismiss
`onChange` → `setBenchmarks` → re-render → keyboard dismisses after 1 character. Accepted tradeoff. Multiple approaches attempted (useRef, debounce, uncontrolled inputs, direct localStorage writes). All either failed to save or caused other issues. Current controlled input with keyboard dismiss is the stable version.

### 2. localStorage "undefined" string corruption
Early versions wrote `undefined` as a string to localStorage. `JSON.parse("undefined")` throws SyntaxError. `useStorage` now checks for and removes `"undefined"` and `"null"` strings on init. `setBench` builds state from React object, never re-reads localStorage.

### 3. Session state resetting on timer tick
`Log` component was defined inside `App`. Every `App` re-render (caused by `RestTimer` or `SessionTimer` state updates) created a new `Log` function reference, causing React to fully remount it and reset all checklist state. Fixed by lifting all session state to App level with a `useEffect` sync when day/week/phase changes.

### 4. Service worker cache
The service worker caches the HTML file aggressively. When deploying updates, the cache key (e.g. `tos-v514`) must be incremented to force clients to fetch the new version. Found in the inline service worker script at the bottom of `index.html`.

### 5. Babel in browser = blank screen on mobile
Previous versions used `<script type="text/babel">` with Babel CDN. This times out on mobile and shows a blank screen. The app is now pre-compiled with Webpack + Babel server-side. Never use browser Babel.

### 6. React CDN load order
If using CDN React (not bundled), the app script must only run after React AND ReactDOM are both confirmed loaded via `onload` callbacks. Bundled approach avoids this entirely.

### 7. onBlur unreliable on mobile
`onBlur` does not reliably fire on mobile browsers when tapping away from an input to a non-focusable element. Use `onChange` for anything that must save reliably. Use `onBlur` only as a secondary/backup save.

---

## What Not To Touch

### Never change these without understanding the full impact:

1. **`useStorage` hook** — fragile, handles corrupted state. Any change risks breaking all persistence.

2. **Session state location** — must stay at App level. Moving any of `sessionStarted`, `warmupChecks`, `skillChecks`, `coreChecks`, `coolChecks`, `neckChecks` back into the Log component will cause them to reset mid-session.

3. **`setBench` implementation** — must build from `{...benchmarks, [k]:v}` not from localStorage read. Re-reading localStorage inside setBench risks hitting corrupted values.

4. **Phase data structure** — `PHASES` array is the source of truth for the entire program. Warmup items, skill blocks, sets, core, cooldown, neck work all read from here. Changes here affect every tab.

5. **Log key format** — `workoutLogs` keys (`p${phaseId}_d${dayId}_w${week}`) and weight keys (`w_${phaseId}_${dayId}_${week}_${setIndex}`) are used everywhere. Changing format breaks all existing saved data.

6. **Service worker cache key** — must increment on every deploy or mobile users get stale version.

---

## The User

Ed, Perth WA. Licensed electrician, PM at Genus Plus Group. Training at a gym with rings, rack, DB/KB, GHD. Currently 112kg, goal 95kg. 

- **Training days:** Tuesday, Thursday, Saturday
- **Active recovery:** Monday, Wednesday, Friday — 5km walk/run with dogs
- **False grip goal:** Strict ring muscle-up
- **Benchmark starting points:** Pull-ups 5 max, false grip active hang 15s, box pistol (Level 5), HSPU Level 2, Nordic Level 1
- **Communication style:** Direct, fast-moving, builder-biased. Working solution before explanation.

---

## Preferred Commit Style

```
fix: benchmark inputs now save reliably on blur
feat: add dragon flag progression ladder across all 3 phases  
fix: session state lifted to App level — prevents checklist reset on timer tick
feat: guided session flow — warmup → skill → sets → core → cooldown → neck
chore: increment service worker cache key to tos-v515
```

Format: `type: short description in present tense`
Types: `feat`, `fix`, `chore`, `refactor`
No ticket numbers, no lengthy descriptions in the commit message — the CLAUDE.md covers context.

---

## Deployment

1. Build locally → produces `index.html`
2. Push to `github.com/BigFred588/training-os` — replace `index.html`
3. GitHub Pages auto-deploys to `bigfred588.github.io/training-os`
4. Cloudflare DNS CNAME: `training` → `bigfred588.github.io` (proxy OFF)
5. Live at `https://training.collict.me` within 1-2 minutes

**Before deploying:** increment the service worker cache key in the inline SW script at the bottom of index.html.

**Data migration:** if localStorage keys or structure change, users need to export backup before update and reimport after. Warn in commit message if this applies.
