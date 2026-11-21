# Prompts — paste these into Claude Code, in order

One prompt per session. Between features, `/clear`. For every prompt after Prompt 1, hit **Shift+Tab**
first so Claude plans before editing. After each green run: "commit this with a sensible message."

---

## Prompt 1 — Migration to a structured project (run first, alone)

```
Read CLAUDE.md fully before doing anything.

Migrate night-sky-3d.html into a structured Vite project with ZERO behavior change. This is a pure
restructuring — no feature work, no visual changes, no library upgrades.

Requirements:
1. Vite + React 18 + three pinned to 0.128.0 (the code targets r128 APIs; upgrading Three is
   explicitly out of scope today). Remove Babel-standalone — Vite compiles JSX now.
2. Structure: src/engine/ (astro.js: all pure astronomy functions; events.js: findEvents + evMeta),
   src/data/ (stars.js, starFacts.js, constellations.js, presets.js), src/components/ (App, Scene3D,
   SkyView, FlyToEarth, DepthView, cards/), src/styles/tokens.js + global.css (the <style> rules).
   Engine and data modules must be importable in Node with no DOM.
3. Build must emit ONE self-contained file named dist/night-sky-3d.html (use vite-plugin-singlefile;
   inline all JS/CSS). My public GitHub Pages link points at that exact filename — it is the release
   artifact and must open from a double-click and over https identically to today.
4. Tests before refactor-freedom: set up vitest and port the verification anchors from CLAUDE.md's
   "Verification culture" section — skyToEq round-trip < 0.01°, the five known event dates
   (Saturn opp 2026-10-04, Venus GE-E 2026-08-15, Venus inf. conj 2026-10-24, Jupiter opp 2027-02-11,
   Mars opp 2027-02-19, each ±1 day), and a cross-dissolve geometry test asserting each visible
   planet's projected 3D landing position is within 25 px of its 2D sky position at p=1 for the
   Haddonfield June-2026 scenario (replicate the camera math from FlyToEarth; see the
   "cross-dissolve math" section).
5. Honor every rule in "Invariants & hard-won bug lessons" — especially: no component definitions
   inside App, the canvas touch-action rules, and jump() vs setTimeMs semantics.
6. Finish with: npm test green, npm run build green, a PARITY.md checklist of every feature from the
   legacy file marked verified, and the legacy night-sky-3d.html moved to legacy/ for reference.
   Then give me the one-command dev loop (npm run dev) and the release loop.

Work in small commits. If any behavior is ambiguous, read the legacy file — it is the spec.
```

---

## Prompt 2 — Analemma tracer (roadmap #3)

```
Read CLAUDE.md (roadmap item 3). Add the analemma tracer.

In SkyView planets mode, add a "trace the Sun" toggle chip (visible when the Sun is the relevant
body — make the Sun selectable like a planet for this purpose). When on, day-step playback leaves a
dot at the Sun's alt/az each simulated day, accumulating the figure-8 over a year at fixed clock
time. Style: small cream dots fading with age, brass dot for "today", mono date labels at the
solstices and the two lobe crossings. Add a one-tap preset in the Sky Calendar area: "Trace the
Sun's analemma (1 year)" that orchestrates it like startLoopDemo does (noon local, day-step, auto-stop
after 366 days, narrated banner explaining tilt = the height of the 8, eccentricity = the lobe
asymmetry). Engine test: at fixed clock time, the Sun's az/alt range over a year matches the tilt
(±23.4° declination swing) — assert in vitest. Update the banner copy in the app's plain physical voice.
```

---

## Prompt 3 — Jupiter's Grand Tour (roadmap #4)

```
Read CLAUDE.md (roadmap item 4). Build "Jupiter's Grand Tour" — the app's origin question: why does
Jupiter sit lower each year?

A guided sequence (entry point: a featured card at the top of the Sky Calendar): starting from the
2026 Gemini opposition, step year-by-year through ~2036. At each stop: jump to that year's Jupiter
opposition (reuse findEvents, extended-scan or compute oppositions on demand), show Jupiter selected
in both views, and display a compact chart (canvas, design tokens) of opposition altitude at my
latitude vs year, with the current stop highlighted. Captions narrate the mechanism: opposition
slides ~30° east per year along the ecliptic; the ecliptic's height at midnight varies with season;
low 2030s oppositions in the southern constellations, recovery by 2036. Next/Back/Auto-play controls,
scrubbable, and a "go explore this sky" exit like the cinematic has. Vitest: Jupiter opposition dates
2027–2036 are each ~398–400 days apart, and opposition ecliptic longitude advances ~30–36°/year.
```

---

## Prompt 4 — Guided tours framework (roadmap #5)

```
Read CLAUDE.md (roadmap item 5). Extract a small guided-tour framework from what the cinematic,
retrograde demo, analemma preset, and Grand Tour now share: a tour = ordered steps of
{clock target or playback spec, view/selection state, caption, optional overlay}, with a shared
runner (play/pause/scrub/skip, auto-stop, narrated banner). Refactor those four onto the runner
(behavior-identical — the existing tests must stay green), then add two new tours written in the
app voice: "Why Venus never leaves the Sun's side" (tangent-line geometry at greatest elongation,
inferior conjunction crossing, morning/evening star rhythm) and "Why winter skies run high"
(ecliptic tilt vs season using the Both view). Tours live as data files in src/tours/.
```

---

## Prompt 5 — Poster export (roadmap #6)

```
Read CLAUDE.md (roadmap item 6). Add poster export: render the current accumulated trail (retrograde
loop or analemma) plus horizon, cardinal letters, ecliptic, labeled bright stars into an offscreen
2048-wide canvas styled with the design tokens (brass on ink, instrument-plate title, place + date
range + latitude stamp, thin brass rule), then download as PNG (toBlob → anchor download; on iOS
Safari verify the share-sheet path works). Entry: a "poster" button that appears in the demo banner
when a trail exists. The poster is generated from the same engine state, not a screenshot — it should
look like a page from a Victorian atlas, per the frontend design system.
```

---

## Prompt 6 — ISS passes (later; needs network)

```
Read CLAUDE.md (roadmap item 7). Add ISS visible passes: fetch current TLEs from CelesTrak at
runtime (fetch from the hosted https page; handle offline gracefully by hiding the feature), propagate
with satellite.js, list the next visible passes for my location (sun below −6°, ISS sunlit, max alt
>10°) in the Sky Calendar with a distinct icon, and draw the pass arc across SkyView when one is
selected. Keep the single-file build constraint: satellite.js gets bundled, TLEs are the only runtime
fetch. Add a vitest with a frozen TLE fixture asserting pass-prediction plumbing is sane.
```

---

## Habits that keep quality high

- **Plan mode first** (Shift+Tab) on every feature; approve the plan, then let it run.
- After it claims done: **"run the tests and the build, then show me what changed"** — make it prove it.
- New bug discovered? Fix it, then: **"add the lesson to CLAUDE.md's invariants section"** — that file
  is the project's accumulated scar tissue; keep it current.
- One feature per session, `/clear` between, commit at every green checkpoint.
```
