# Orrery · 3D — Night Sky & Solar System Simulator

A mobile-first astronomy teaching instrument. Its core thesis: **hold two representations of the same
moment — the heliocentric orbital model and the lived night sky — and let the user move between them.**
Every feature should deepen that bridge, not chase planetarium apps (no deep-sky catalogs, no AR).

- **Primary user/device:** iPhone Safari. Owner is at ~39.9°N, −75.0°W (Haddonfield, NJ); defaults assume it.
- **Deployed:** GitHub Pages, as a single self-contained file `night-sky-3d.html`. The shared public
  link points at that exact filename — **every build must keep emitting one self-contained
  `night-sky-3d.html`** (inline everything; CDN script tags are acceptable).
- **Current state:** one 1,175-line HTML file (`night-sky-3d.html`) using UMD React 18.2.0,
  Three.js **r128**, and in-browser Babel 7.23.2 from cdnjs. The first migration task is to turn this
  into a structured Vite project *without changing behavior* (see PROMPTS.md, Prompt 1).

## Architecture (as it exists in the single file, to be preserved through migration)

**Astronomy engine** (pure functions, no DOM — extract to `src/engine/`):
- J2000 Keplerian elements + per-century rates for Mercury–Saturn (incl. Earth); iterative Kepler solver.
- Coordinate chain: heliocentric ecliptic → geocentric ecliptic → equatorial (obliquity
  `OBL = 23.43928°`) → RA/Dec → alt/az via GMST
  (`LST = 280.46061837 + 360.98564736629·d + lon`, d = days since J2000).
- `skyState(ms, lat, lon)` → sun, moon (low-precision, ~1° — fine for display, never for eclipse math),
  planets with `{ra, dec, alt, az, eclLon, elong, dist}`.
- `eclConstellation(lon)`: zodiac bins by ecliptic longitude **including Ophiuchus** (deliberate).
- `findDarkMoment` (scan local solar 18:00→02:00 for sun alt ≈ −15°, darkest-moment fallback for
  white nights), `bestLookAz` (altitude-weighted circular mean azimuth of planets up).
- `findEvents(fromMs, days)`: daily coarse scan + hourly bisection refinement → oppositions,
  conjunctions (inferior/superior split by geocentric distance vs 1 AU), greatest elongations
  (local max of elongation, >15° threshold filters Mercury noise), retrograde stations (sign change
  of unwrapped geocentric longitude). ~145 ms for 3 years; computed post-mount in a deferred effect.
- `STARS`: 48-star catalog `[name, ra, dec, mag, dist_ly]`; `STAR_FACTS` has an entry for **all 48**
  (type, radius, luminosity, one-line story, Wikipedia slug). `FEATURED`: 5 constellations
  (Orion, Big Dipper, Gemini, Cygnus, Cassiopeia) with line segments + depth blurbs.

**Components:**
- `App` — all shared state; sticky time ribbon (two rows: readout+Now+date picker / transport
  −1y…+1y); Orbits/Sky/Both view switcher; Tonight's Report; Sky Calendar (event list → tap jumps
  clock + selects planet + expands explanation; outer-planet opposition/station rows get
  "watch the retrograde loop" which orchestrates: view=both, trail on, day-step play across a
  per-event window, narrated banner, auto-stop).
- `Scene3D` — heliocentric Three.js scene. **Scene frame:** ecliptic coords mapped
  `(x, y, z)_scene = (x_ecl, z_ecl, −y_ecl)`. **Radial compression:**
  `sceneR = sqrt(r_AU / 9.6) · 9` (Saturn ≈ 8.97 units). Drag/pinch camera, tap-to-select
  (raycaster), Earth→selected sight line, follow-Earth mode.
- `SkyView` — 120° swipeable horizon window, canvas 2D. Maps azimuth linearly across width,
  altitude linearly −6°…88° down the height. Modes: planets+ecliptic (retrograde trail = 300-day
  sampling) / stars (all 48 tappable → fact card; top 10 haloed+labeled; constellation lines toggle
  with tappable centroid names → story card → "unstack in depth view" cross-link). Phone-compass
  mode (see invariants). Hit-testing: nearest within 22 px.
- `FlyToEarth` — the cinematic. Heliocentric establishing orbit (4.5 s) → curved Bézier descent
  (6.5 s) through a waypoint on the −gaze side (never transits the Sun) → lands essentially AT
  Earth → cross-dissolves (final 22% of descent) into the 2D sky window. Scrubber reconstructs any
  frame (`frame(p)` is pure in p).
- `DepthView` — constellation sky⟷distance morph with log-scale toggle.

## The cross-dissolve math (hard-won — do not "simplify")
The landing camera must make the 3D frame coincide with the 2D sky window:
- **Gaze:** `skyToEq(lookAz, GAZE_ALT=37)` → RA/Dec → `eqToSceneDir` (eq→ecliptic rotation by −OBL,
  then the scene frame swap). 37 (not the window's 41° center) deliberately splits the tan-vs-linear
  nonlinearity across the planet band.
- **Up vector:** local zenith (`skyToEq(0,90)` → scene) so roll matches; eased from (0,1,0).
- **End FOV = 78.7° = 2·atan(47·π/180)** — matches the 2D window's vertical px/deg exactly,
  independent of canvas size.
- **Direction morph:** radial compression warps apparent geocentric directions, so during descent
  each planet nlerps from its compressed-scene direction onto its **true** geocentric direction
  (preserving its scene distance from Earth). At the cut, planets sit where the real sky puts them.
- Verified by headless harness: Venus Δ≈10 px, Jupiter ≈8 px, Mercury ≈13 px on a 770-px frame;
  Δx ≈ 0 because `bestLookAz` centers the window on the planet cluster. Horizontal scale can't match
  everywhere (linear panorama vs perspective) — known, accepted.

## Invariants & hard-won bug lessons (violating these reintroduces shipped bugs)
1. **Never define a component inside `App`** (or any component that re-renders during playback).
   The clock re-renders ~60 fps while playing; an inner component type remounts its DOM every frame
   and **buttons eat taps** (the original unpausable-pause bug). Inline element-returning functions
   (`{TimeStepper()}`) or hoisted components only.
2. **WebGL canvas touch:** `touch-action: none` on the canvas; `touchmove` listener non-passive with
   `preventDefault()`; swallow Safari `gesturestart/change/end`. **Never `preventDefault()` on
   `touchstart`** — it kills the synthesized click and breaks tap-to-select.
3. **Compass needs real `https://`.** `file://` falsely reports `isSecureContext === true`; check
   `location.protocol === "file:"` explicitly and explain. iOS requires the permission request to run
   inside the user's tap handler.
4. **`jump()` pauses playback by design.** Demos that must keep playing call `setTimeMs` directly
   then `setPlaying(true)` (see `startLoopDemo`).
5. Month/year steps go through `addMonths` (calendar-aware; Jan 31 + 1 mo lands on a real date).
6. No literal `\uXXXX` escapes in JSX text — use real Unicode characters.
7. SkyView tap-vs-swipe: a pointer that moved ≥6 px is a swipe, not a tap.
8. Trail length is 300 days — long enough to show a full retrograde loop; don't shorten.

## Verification culture (keep it; port to vitest)
This project tests by **computing ground truth**, not by eyeballing:
- After every change the whole JSX must compile (Babel in CI terms → in Vite land: `npm run build`
  must pass; add `vitest` for the engine).
- Engine regression anchors (all previously verified): `skyToEq` round-trips alt/az ↔ RA/Dec to
  <0.01°; planet `dirTrue` matches `eqToSceneDir(ra,dec)` to ~0°; event dates vs known astronomy —
  **Saturn opposition 2026-10-04, Venus greatest elong E 2026-08-15, Venus inferior conj 2026-10-24,
  Jupiter opposition 2027-02-11, Mars opposition 2027-02-19**; oppositions report elong 175–179°
  (orbital inclination — correct, not a bug).
- A headless canvas harness (`@napi-rs/canvas`) renders the cross-dissolve frame and measures per-planet
  pixel deltas (3D vs 2D). Recreate as `scripts/verify-dissolve.mjs`; alignment regressions must fail loudly.

## Design system (sharpen, never replace)
Antique brass observatory instrument floating in deep space.
- Tokens (`C`): void `#05060f`, panel `#0c1226`, panel2 (cards), hairline `#26345a`,
  cream `#ece4cf`, brass `#c9a14e`, star-gold/sel `#ffd97a`, dim `#8e97b3`.
- Type: Georgia/serif display; `ui-monospace` for ALL data readouts (dates, degrees, distances).
- Motifs: engraved letterspaced "instrument plate" section labels (TIME, YOUR SKY · 120° WINDOW…),
  segmented pill selectors, brass range-thumbs, press-down button feedback, `prefers-reduced-motion`
  respected. Boldness lives in the cinematic; surrounding UI stays quiet.
- Writing voice in-app: plain, physical, second-person ("Earth, on the faster inside track, has
  begun to overtake it").

## Roadmap (agreed order; one feature per session, each behind a test)
1. ✅ Event engine + Sky Calendar  2. ✅ Retrograde lockstep demo
3. **Analemma tracer** — day-step mode + "leave a Sun trail" → figure-8 over a simulated year.
4. **Jupiter's Grand Tour** — guided year-step sequence: one zodiac constellation/year, opposition
   altitude falling to the ~2030 lows, recovering by 2036 (the app's origin question).
5. **Guided tours framework** — generalize the cinematic's script (clock + camera + captions) into a
   small tour format; first tours: "Why Venus never leaves the Sun's side", "Why winter skies run high".
6. **Poster export** — accumulate a trail (Mars 2027 loop, a year of Venus) → styled PNG in the
   brass-and-ink look, stamped with place/dates.
7. Later: comet (Halley's elements drop into the engine — eccentric/inclined counterexample),
   light-time labels ("Jupiter: 43 min ago"), ISS passes via live CelesTrak TLEs (feasible now that
   we have a build step + hosting).
Deliberately out of scope: full star catalogs, deep-sky objects, AR overlay.

## Migration target (Prompt 1 executes this)
Vite + React 18 + `three` pinned to the r128-compatible npm release (`three@0.128.0`) for parity —
upgrade Three in a *separate* later task, never during migration. Structure:
`src/engine/` (pure, tested) · `src/components/` · `src/data/` (stars, facts, constellations,
presets) · `src/styles/` (tokens) · `scripts/verify-dissolve.mjs` · `vite-plugin-singlefile` build
emitting `dist/night-sky-3d.html`. Parity bar: every behavior in the legacy file, byte-for-byte
feature list, before any refactor-enabled improvements.
