# Premium Visual Overhaul — 2026-06-10

Goal: make the game feel like a real, crowded Red Bull Flugtag event with premium water and a cohesive premium UI.

## 1. Environment — festival atmosphere
- [x] Replace per-mesh crowd with InstancedMesh crowd (legs+body+head share one matrix buffer) — 3,120 spectators lining the full course, packed at ramp + finish
- [x] Animated crowd: jumping/bobbing in a ±100m window around the craft, extra hype for 2.5s on splash
- [x] Riverside promenade (stone retaining wall + sandstone deck + trim) so crowds stand on a real embankment
- [x] ~135 hand flags waving in the crowd (instanced, color palette)
- [x] ~28 festival tents with striped canvas roofs (instanced, 3 color variants)
- [x] 34 rising balloons (instanced, loop 2m→26m with sway)
- [x] 5 spectator boats moored at the river edges, bobbing on the wave function
- [x] Bunting line sagging between the slingshot posts (canvas texture)
- [x] 90-piece confetti burst on launch (instanced, fluttering)
- [x] Trees converted to InstancedMesh (~520 trees: 2 draw calls instead of ~3,000)
- [x] Buoys converted to InstancedMesh + bobbing/rocking on waves
- [x] Wired up the dead finish-banner bob animation

## 2. Water — premium look
- [x] Procedural diffuse texture: cross-river depth gradient, soft streaks, sparkles; scrolls with river flow
- [x] Procedural normal map (radial-blob heightfield → Sobel) for sun glints, scrolling at a second speed for shimmer
- [x] Water switched to MeshPhongMaterial (specular 0xBFEFFF, shininess 85) keeping the vertex swells
- [x] Foam strips along both banks: scalloped alpha texture, vertices follow the wave function, mirrored per side
- [x] Two expanding foam rings on splash impact

## 3. UI/UX — premium pass (Sled Surfers design system everywhere)
- [x] @font-face for local fonts: Slingshot Logo / Body / Flavor / Numerals (all verified loading)
- [x] .game-btn rebuilt: gradient pills, 3px navy outline, colored drop-stack shadows, press states
- [x] Title: white stroked logo, flavor-font subtitle, green TAP TO PLAY, best-distance chip
- [x] Aim: coin chip + best chip, DRAG TO LAUNCH copy, animated drag hint (first flight only), power readout as white chip
- [x] Upgrade cards: white cards, per-track colored icon badges (red/blue/gold), coin-glyph cost buttons, gold MAX, navy pips
- [x] Flight HUD: stroked numeral distance counter, stat chips, ss-style fuel bar, coin chip
- [x] BOOST button (hold) — engine thrust was previously unreachable on touch (dead `upgrades.engine` check); particles now only when actually thrusting
- [x] Results: pale card, overhanging FLIGHT COMPLETE ribbon, white stat rows, coin count-up (tickNumber), orange NEW RECORD chip, pop-in animation
- [x] Garage: green FLY! confirm pill, panel pop-in
- [x] Dead CSS removed (old garage block, .aim-instruction); float text restyled

## Bonus fixes found during verification
- [x] Village houses/chapels rendered pitch black (pre-existing): the GLBs ship without normal attributes (unlit assets); toonifyModel now runs computeVertexNormals() when normals are missing
- [x] FINISH banner text was mirrored for the approaching player (pre-existing): rotated to face -z

## Verification
- [x] Full flow stepped headlessly: TITLE → GARAGE → READY → LAUNCH → FLIGHT → SPLASH → RESULTS, console clean
- [x] DOM checks per screen (cards, chips, copy, classes, fonts) — all pass
- [x] Canvas captures in tasks/shots/: f1-title, f2-garage, f4-launch-confetti, f5-flight, f6-splash, f9-finish-fixed, house-after-fix
- [x] Perf: 157 draw calls / 1.2M tris in flight (previously ~3,000+ calls from per-mesh trees alone)
- Note: the preview panel ran hidden (rAF throttled), so captures were driven manually; DOM overlay (HUD) verified via computed DOM state rather than pixels. Worth one quick human pass in a visible browser.

## Review
Biggest risks/judgment calls:
- Crowd matrix sharing (legs/heads reuse body instanceMatrix) keeps per-frame cost to one buffer upload; animation window is binary-searched on a z-sorted array.
- Water stays vertex-displaced at 24×96 segments; lighting detail comes from the scrolling normal map, so no per-frame normal recompute.
- All instanced meshes set frustumCulled=false (instance bounds aren't tracked by three r128).
- Crowd/trees/tents still cast shadows; verified no shadow-frustum artifacts (the black blobs were the missing-normals bug, not shadows).

## Round 2 — 2026-06-10 (font legibility + input feel)
- [x] Replaced local Slingshot TTFs with Google Fonts: Lilita One (display/flavor/numerals — the Monopoly Go look) + Baloo 2 (body); removed @font-face blocks
- [x] Removed -webkit-text-stroke from small text (<20px): upgrade buttons, NEW RECORD chip, boost button, skin tabs, title subtitle — shadows only
- [x] FINISH banner rebakes its canvas texture on document.fonts.ready in Lilita One
- [x] Drag surface is now the whole window (was canvas-only): listeners on window, only `button`/craft-card/skin-tab elements excluded — upgrade-card strip and HUD no longer dead zones
- [x] Pitch input captured during the 300ms LAUNCH transition (was swallowed → flight controls felt dead right after launch); pitch sensitivity 80px → 70px for full deflection
- [x] Verified: fonts loaded (FontFace registry), synthetic event-path tests for guard/drag/launch/pitch/boost all pass, console clean, banner capture in tasks/shots/f10-banner-lilita.jpg
