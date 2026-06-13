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

## Round 3 — 2026-06-11 (Tier 1+2 graphics: full three.js pipeline)
Tier 1 — rendering pipeline:
- [x] Sky moved in-scene: gradient dome, sun disc + halo, cropped Salzburg panorama as far backdrop plane, 3 layered haze ridges; sky group follows camera down-course
- [x] ACES filmic tone mapping (exposure 1.0) + opaque clear — CSS background image dropped (saves the 3MB PNG from the page background; it loads once as a texture instead)
- [x] EffectComposer: RenderPass -> UnrealBloom (0.22/0.4/0.92) -> Vignette -> GammaCorrection; composer resize wired
- [x] Distance fog (0xC8E2F0, 320–1900) for atmospheric depth; sky elements opt out
- [x] Water waves moved to GPU (onBeforeCompile, 4 octaves, analytic normals ×10 slope boost); CPU vertex loop deleted; 36×200 mesh; shininess 110
- [x] Shadows 2048px; sun + shadow frustum track the craft in every state
Tier 2 — reflections + set dressing:
- [x] Planar reflections: THREE.Reflector with custom shader (wave-distorted via shared normal map + waterTime clock, distance-faded alpha), 512px RT, blended over stylized water; water/foam/rings/beams/village excluded from mirror pass via onBeforeRender wrapper
- [x] Clouds (11 drifting 3-lobe instanced puffs), 8 birds with flapping instanced wings, haze ridges
- [x] Ramp hero props: striped inflatable start arch (raised to 7.2r so the launch camera flies through it), jumbotron with live canvas screen (state text + live distance during flight, 0.25s throttle), speaker stacks, 2 additive searchlight beams (pre-flight only)
- [x] Crowd v2: raised cheering arms on ~50% of jumpers (own instanced buffer, y-synced), stadium wave crest traveling down the banks
- [x] Water spray when skimming below 2.4m
Tuning found in verification:
- [x] First capture was blown out: searchlights were DoubleSide/0.14 additive at frame center + bloom threshold 0.82 over a bright toon palette → narrowed beams (0.06, FrontSide), bloom 0.92 threshold, exposure 1.0
- [x] Arch originally blocked the camera right after launch → raised/widened into a fly-through gate; panorama's baked water band cropped via texture repeat/offset
Verification: console clean, full flow stepped, splash shows craft mirror reflection + rings, garage framed by props; 272 draw calls / 12.8ms CPU frame including mirror pass (village GLBs stay the tris heavyweight — excluded from reflections; decimating them is a possible follow-up). Captures: t3-title, t5-flight, t6-splash, t7-reflections, t8-garage.

## Round 4 — 2026-06-11 (launch stage, procedural village, premium buoys)
- [x] Launch stage rebuilt as a branded Flugtag pier: runway top texture (red edge bands, dashed center line, painted FLUGTAG lettering, rebaked on fonts.ready), yellow/navy chevron takeoff lip, navy/white striped skirt panels, FLUGTAG banner faces + big hanging banner on the drop edge, instanced scaffold truss underneath, side railings, candy-striped slingshot pylons w/ gold knobs (band anchors unchanged), 4 waving corner flags, stone bridge w/ pillar caps + trim behind
- [x] GLB village removed (austrian-alpine-house / salzburg_baroque_chappel no longer loaded) and replaced with procedural toon village: 3 plaster wall variants w/ baked timber trim + shuttered windows + flower boxes, prism shingle roofs tinted per-instance (3 brick tones), chimneys on ~55%, houses face the river; 5 chapel landmarks (white tower, clock, patina onion dome + gold tip, nave). 5 instanced draw calls for ~100 houses; total scene tris dropped 4.1M -> 1.87M and houses now appear in water reflections (no longer excluded)
- [x] Buoys rebuilt as inflatable regatta markers: float ring + squashed sphere body + white band + navy cone cap (4 instanced parts sharing one matrix buffer), red/yellow alternating with darker rings, per-buoy scale variety, markers alternate sides of the channel; same bob/rock animation
- [x] Start arch raised earlier still clipped the chase camera on the way through -> arch now fades out as the camera approaches (opacity by z-distance) and the fly-through reads cinematic
- [x] Fixed a TDZ crash I introduced (launchArch declared in a later section than its assignment — script init died silently; found via binding probes since the hidden tab swallowed the uncaught error)
- Verification: console clean, full launch->flight pass captured (s8 arch crossing, s9 downstream), stage water-side hero shot (s6), village (s3), buoy closeup (s5); 315 draw calls / 1.87M tris / ~14ms CPU per frame incl. mirror pass

## Round 5 — 2026-06-11 (wide-screen backdrop + Salzburg riverfront city)
- [x] Backdrop gap on wide screens fixed: flat panorama plane -> cylinder arc (~166°, r2250) wrapped around the camera, mirror-corrected for BackSide; covers any aspect ratio
- [x] Haze ridges converted to arcs too, then REMOVED entirely — from flight altitude the transparent cylinder walls produced ray-like sky artifacts; fog + panorama carry the depth
- [x] Salzach riverfront city built along both banks (the empty grass): cobbled street strips, ~380 Getreidegasse-style townhouses (5 pastel facade variants with baked windows/shutters/shop fronts/awnings, instanced multi-tone prism roofs seated to per-instance body height), side-street gaps, 124 street lamps, 13 cafe terraces (parasols + tables), 2 Salzburg-cathedral landmarks (twin towers + patina domes + clocks) with reserved row gaps
- [x] Mid-ground reshuffle: near-bank trees moved behind the city (offset 30+), tents shrunk into street market stalls, jumbotron moved out of the river onto the bank
- [x] City is cheap (instanced boxes) so it reflects in the river — no mirror-pass exclusion
- Verification (wide 1300×634 + street/cathedral closeups): backdrop seamless edge to edge, city canyon during flight, console clean; 426 draw calls / 2.2M tris incl. mirror pass. Captures: w2-street, w3-cathedral, w6-aim-final.

## Round 6 — 2026-06-11 (backdrop revert + procedural side scenery)
- [x] Cylinder panorama reverted: the arc stretch distorted the painting -> back to the original flat 3400x680 plane (undistorted)
- [x] Empty side sky filled with procedural painted-style scenery in skyGroup (all MeshBasic/unlit to match the PNG): instanced snowy peaks w/ flush-fitting snow caps (cones), rolling green hill band, 32 pastel building blocks w/ instanceColor, 4 church towers w/ patina domes; drawn renderOrder -9 so the PNG wins where they overlap
- [x] Flight-sky pink haze chased down via raycast through the tinted pixels: it was the LAUNCH ARCH at ~2m from the camera at 19% opacity (the fade window let the camera reach the tube half-faded, smearing it across the sky) -> fade completes while the arch is still 10+ units ahead ((d-10)/5)
- Verification (wide + portrait): aim view undistorted with seamless side fill, launch crossing clean, flight canyon clean. Captures: b1-aim-wide, c1-crossing, c2-flight.

## Round 7 — 2026-06-13 (quality pass + screen-locked backdrop)
- [x] Backdrop refit: panorama is now a screen-locked plane (PlaneGeometry(1,1) scaled each frame in updateSky to full viewport width at PANO_DIST, height from the image's own aspect — never stretched, base pinned to PANO_BOTTOM_Y). Removed the procedural side-fill block (createBackdropSides, ~113 lines). Image aspect read from the loaded PNG; verified seamless at wide (1300x640) and portrait (420x760).
- [x] Building facades rebuilt at 512x768: plaster gradient + weathering, 4 floors x 4 bays of tall windows with alternating triangular/flat pediments, green shutters, flower boxes on mid floors, string courses, ornate dentil cornice, rusticated ground-floor arcade with arched shopfronts + striped awnings + hanging sign. Replaces the flat 256px 3-row facade.
- [x] Roof eaves added: dark overhang box capping each city building under the roof prism (instanced) for crisp roofline depth.
- [x] Cobblestone street upres'd to 256px warm Salzburg setts (per-stone tone variation + top sheen) from the old 128px grey grid.
- [x] Crowd quality: blocky box figures -> tapered cylinder torsos, smoother 10x8 heads, smoother legs, and a NEW hair/hat cap layer (instanced, shares body matrix, browns/blacks/greys + bright caps) killing the uniform bald look.
- Verification (wide + portrait + street/crowd closeups, console clean): 423 draw calls / 4.1M tris / ~10ms CPU frame incl. mirror pass (tris up from crowd geometry detail; fine for desktop). Captures: q1-aim-wide, q2-street, q4-flight.
