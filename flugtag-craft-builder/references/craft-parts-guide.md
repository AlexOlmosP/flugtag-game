# Craft Parts Asset Guide

Complete asset catalog for all craft parts in the Red Bull Flugtag game.

---

## Table of Contents

1. [Asset Requirements](#asset-requirements)
2. [File Naming Convention](#file-naming-convention)
3. [Fuselages](#fuselages)
4. [Wings](#wings)
5. [Propulsion](#propulsion)
6. [Decorations](#decorations)
7. [Helmets](#helmets)
8. [Pilot Body](#pilot-body)
9. [AI Generation Prompts](#ai-generation-prompts)
10. [Environment Assets](#environment-assets)
11. [Collectible Assets](#collectible-assets)
12. [Blender Export Checklist](#blender-export-checklist)

---

## Asset Requirements

| Property | Value |
|----------|-------|
| **File format** | GLB (binary glTF 2.0) |
| **Max poly count** | 2,000 triangles per part (8,000 total assembled craft) |
| **Texture size** | 512x512 max, prefer 256x256 |
| **Color space** | sRGB, white base for tintable parts |
| **Scale** | 1 unit = 1 meter in Three.js |
| **Origin point** | Center-bottom of the mesh bounding box |
| **Up axis** | Y-up |
| **Forward axis** | -Z (Three.js convention) |
| **Materials** | Baked vertex colors or single diffuse texture, no PBR |
| **Animation** | None for static parts; propulsion may have spinning anim |
| **Naming inside GLB** | Root node named `{Category}_{Type}` (e.g., `Fuselage_bathtub`) |

### Tintable parts (modeled white)

Fuselages, wings, and helmets must be exported with **pure white** base color
(#FFFFFF or vertex color white). Color is applied at runtime via `tintModel()`.
Keep any accent details (knobs, edges, trim) as separate non-white meshes so
they are not affected by tinting.

---

## File Naming Convention

```
flugtag-{category}-{type}.glb
```

Examples:
- `flugtag-fuselage-bathtub.glb`
- `flugtag-wings-biplane.glb`
- `flugtag-propulsion-fan.glb`
- `flugtag-deco-side-team_banner.glb`
- `flugtag-deco-tip-streamers.glb`
- `flugtag-deco-nose-mascot.glb`
- `flugtag-deco-tail-tail_flag.glb`
- `flugtag-helmet-aviator.glb`
- `flugtag-pilot-body.glb`
- `flugtag-env-pier.glb`

---

## Fuselages

All fuselages sit on the Y=0 ground plane. The pilot sits approximately at the
center-top of the fuselage. Wing mount points are defined per fuselage type in
the rendering guide.

### bathtub

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-bathtub.glb` |
| **Dimensions** | 1.2m L x 0.6m W x 0.5m H |
| **Poly budget** | 800 tris |
| **Description** | Classic claw-foot bathtub with rounded rim, four ornate legs, drain plug chain. White porcelain base (tintable), gold accent legs and faucet handles. |
| **Tintable** | Yes (porcelain body) |
| **Pilot seat offset** | (0, 0.45, 0) |

### shopping_cart

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-shopping_cart.glb` |
| **Dimensions** | 1.0m L x 0.5m W x 0.8m H |
| **Poly budget** | 600 tris |
| **Description** | Supermarket shopping cart with wire basket, fold-out child seat, four caster wheels. Wire mesh body (tintable), grey metal frame. |
| **Tintable** | Yes (wire basket) |
| **Pilot seat offset** | (0, 0.5, -0.1) |

### rubber_duck

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-rubber_duck.glb` |
| **Dimensions** | 1.4m L x 0.8m W x 0.9m H |
| **Poly budget** | 900 tris |
| **Description** | Giant rubber duck with a hollow cockpit carved into the back. Round body, prominent beak, beady eyes. Yellow base (tintable), orange beak and black eyes are non-tintable accents. |
| **Tintable** | Yes (duck body, not beak/eyes) |
| **Pilot seat offset** | (0, 0.5, 0.1) |

### piano

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-piano.glb` |
| **Dimensions** | 1.5m L x 0.8m W x 0.6m H |
| **Poly budget** | 1000 tris |
| **Description** | Baby grand piano with lid propped open, keys visible on one side. Three chunky legs. Black/white body (tintable main body), ivory keys and gold pedals are accents. |
| **Tintable** | Yes (piano body) |
| **Pilot seat offset** | (0, 0.55, 0) |

### boat

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-boat.glb` |
| **Dimensions** | 1.6m L x 0.6m W x 0.5m H |
| **Poly budget** | 700 tris |
| **Description** | Chunky wooden rowboat with visible plank lines, small bench seat, two oar locks (no oars). Wood base (tintable), darker wood trim accents. |
| **Tintable** | Yes (hull planks) |
| **Pilot seat offset** | (0, 0.35, 0) |

### rocket

| Property | Value |
|----------|-------|
| **File** | `flugtag-fuselage-rocket.glb` |
| **Dimensions** | 1.8m L x 0.5m W x 0.7m H |
| **Poly budget** | 800 tris |
| **Description** | Cardboard tube rocket ship with a cone nose, three tail fins, painted window circles, and visible duct tape seams. Cardboard base (tintable), grey duct tape and drawn-on details as accents. |
| **Tintable** | Yes (cardboard body) |
| **Pilot seat offset** | (0, 0.4, 0.2) |

---

## Wings

Wings are modeled as a **pair** (left + right) in a single file. The center gap
between left and right wing should be approximately 0.6m to straddle the fuselage.

### biplane

| Property | Value |
|----------|-------|
| **File** | `flugtag-wings-biplane.glb` |
| **Dimensions** | 3.0m total span x 0.5m chord x 0.6m stack height |
| **Poly budget** | 500 tris |
| **Description** | Dual stacked rectangular wings connected by vertical struts and wire bracing. Canvas surface with visible rib lines. White (tintable). |

### delta

| Property | Value |
|----------|-------|
| **File** | `flugtag-wings-delta.glb` |
| **Dimensions** | 2.5m span x 0.8m root chord |
| **Poly budget** | 300 tris |
| **Description** | Swept-back triangular delta wings, smooth surface. Think paper airplane but chunkier. White (tintable). |

### hang_glider

| Property | Value |
|----------|-------|
| **File** | `flugtag-wings-hang_glider.glb` |
| **Dimensions** | 4.0m span x 1.0m root chord |
| **Poly budget** | 400 tris |
| **Description** | Large fabric hang glider delta shape with visible aluminum tube frame along leading edge and keel. Fabric (tintable), silver tubes are accents. |

### cardboard_box

| Property | Value |
|----------|-------|
| **File** | `flugtag-wings-cardboard_box.glb` |
| **Dimensions** | 2.0m span x 0.4m chord |
| **Poly budget** | 200 tris |
| **Description** | Flat cardboard flaps with visible corrugation lines, duct-taped to a central dowel rod. Brown cardboard (tintable), grey tape accents. |

### umbrella

| Property | Value |
|----------|-------|
| **File** | `flugtag-wings-umbrella.glb` |
| **Dimensions** | 2.4m total span (two 0.8m radius domes) |
| **Poly budget** | 500 tris |
| **Description** | Two open umbrellas mounted horizontally, dome-side up, with visible spoke ribs. Fabric dome (tintable), black metal shaft/spokes accents. |

---

## Propulsion

Mounted at the rear of the fuselage. May include animated sub-meshes.

### rubber_band

| Property | Value |
|----------|-------|
| **File** | `flugtag-propulsion-rubber_band.glb` |
| **Dimensions** | 0.4m L x 0.3m propeller diameter |
| **Poly budget** | 300 tris |
| **Description** | Wound rubber band wrapped around a wooden peg, driving a small two-blade balsa propeller. The propeller spins (animated or driven in code). |

### fan

| Property | Value |
|----------|-------|
| **File** | `flugtag-propulsion-fan.glb` |
| **Dimensions** | 0.3m L x 0.35m cage diameter |
| **Poly budget** | 400 tris |
| **Description** | Desk oscillating fan with wire cage guard and three plastic blades. Plugged into a comically large battery via a dangling cable. Blades spin. |

### pedal

| Property | Value |
|----------|-------|
| **File** | `flugtag-propulsion-pedal.glb` |
| **Dimensions** | 0.5m L x 0.4m H |
| **Poly budget** | 500 tris |
| **Description** | Bicycle pedal crank connected via chain to a rear propeller. Two pedals, a chain loop, a sprocket, and a two-blade prop. Animated pedal rotation drives prop spin. |

### none

No asset needed. Empty group in scene hierarchy.

---

## Decorations

Small accent pieces attached to anchor points on the craft.

### Fuselage Side (slot: `fuselage_side`)

| ID | File | Dimensions | Polys | Description |
|----|------|-----------|-------|-------------|
| `team_banner` | `flugtag-deco-side-team_banner.glb` | 0.4m x 0.25m flat | 50 | Rectangular cloth banner with wavy edges, hanging from two pins |
| `sponsor_logo` | `flugtag-deco-side-sponsor_logo.glb` | 0.3m x 0.3m flat | 50 | Circular sticker / painted logo decal |
| `flame_paint` | `flugtag-deco-side-flame_paint.glb` | 0.5m x 0.2m flat | 80 | Hot rod flame paint decal, orange-to-yellow gradient |
| `number` | `flugtag-deco-side-number.glb` | 0.25m x 0.3m flat | 60 | Large bold racing number "42" in a white circle |
| `stars` | `flugtag-deco-side-stars.glb` | 0.4m x 0.3m flat | 100 | Cluster of 5 gold star stickers at various angles |

### Wing Tip (slot: `wing_tip`)

| ID | File | Dimensions | Polys | Description |
|----|------|-----------|-------|-------------|
| `streamers` | `flugtag-deco-tip-streamers.glb` | 0.6m trailing length | 100 | Three colorful crepe paper streamers trailing from wingtip |
| `ribbons` | `flugtag-deco-tip-ribbons.glb` | 0.5m trailing length | 80 | Satin ribbon loops fluttering from wingtip |
| `lights` | `flugtag-deco-tip-lights.glb` | 0.4m strand | 120 | String of fairy lights with tiny bulb meshes along the wing edge |
| `flags` | `flugtag-deco-tip-flags.glb` | 0.3m each flag | 80 | Two small triangular pennant flags on sticks |

### Nose (slot: `nose`)

| ID | File | Dimensions | Polys | Description |
|----|------|-----------|-------|-------------|
| `mascot` | `flugtag-deco-nose-mascot.glb` | 0.3m x 0.3m x 0.4m | 200 | Chunky cartoon animal mascot (generic bear) mounted on a spring |
| `eyes` | `flugtag-deco-nose-eyes.glb` | 0.2m x 0.15m each | 100 | Two large googly eyes with rattling black pupils |
| `horn` | `flugtag-deco-nose-horn.glb` | 0.3m x 0.15m | 80 | Coiled party horn / noisemaker, bright red |
| `figurehead` | `flugtag-deco-nose-figurehead.glb` | 0.2m x 0.35m | 250 | Carved wooden ship figurehead, mermaid or eagle |

### Tail (slot: `tail`)

| ID | File | Dimensions | Polys | Description |
|----|------|-----------|-------|-------------|
| `tail_flag` | `flugtag-deco-tail-tail_flag.glb` | 0.3m x 0.5m | 60 | Flag on a short pole, flapping in the wind |
| `smoke_can` | `flugtag-deco-tail-smoke_can.glb` | 0.1m x 0.2m | 50 | Small smoke canister strapped on, trails colored smoke (VFX) |
| `parachute` | `flugtag-deco-tail-parachute.glb` | 0.2m packed / 1.0m deployed | 150 | Bundled parachute pack, deploys on landing |
| `cape` | `flugtag-deco-tail-cape.glb` | 0.6m x 0.8m | 120 | Superhero cape flowing behind, cloth-sim-like geometry |

---

## Helmets

All helmets modeled to fit on a spherical head ~0.2m radius.
White base (tintable), with non-tintable accents.

| ID | File | Dimensions | Polys | Description |
|----|------|-----------|-------|-------------|
| `aviator` | `flugtag-helmet-aviator.glb` | 0.25m x 0.2m x 0.22m | 300 | Leather aviator cap with raised goggles on forehead. Cap (tintable), goggles brown leather + amber lens accent. |
| `construction` | `flugtag-helmet-construction.glb` | 0.24m x 0.22m x 0.18m | 200 | Hard hat / construction helmet with brim. Shell (tintable), black chin strap accent. |
| `viking` | `flugtag-helmet-viking.glb` | 0.26m x 0.25m x 0.28m | 400 | Round viking helmet with two curving horns. Metal bowl (tintable), bone-colored horns accent. |
| `astronaut` | `flugtag-helmet-astronaut.glb` | 0.3m x 0.28m x 0.3m | 350 | Bubble space helmet with gold visor. Shell (tintable), gold reflective visor accent. |
| `chicken` | `flugtag-helmet-chicken.glb` | 0.3m x 0.25m x 0.35m | 450 | Full chicken head mask with comb, beak, wattle. Body (tintable), red comb, orange beak, red wattle accents. |

---

## Pilot Body

| Property | Value |
|----------|-------|
| **File** | `flugtag-pilot-body.glb` |
| **Dimensions** | 0.3m W x 0.8m H x 0.25m D (seated pose) |
| **Poly budget** | 500 tris |
| **Description** | Chunky cartoon human in a seated/riding pose. Legs forward, arms gripping (an invisible handlebar position). Simple jumpsuit with visible zipper line. Not tintable -- fixed bright orange jumpsuit. Head is a smooth sphere so helmets fit cleanly on top. |
| **Head center offset** | (0, 0.7, 0) from pilot body origin |

---

## AI Generation Prompts

Use these prompts with NanoBanana, Meshy, Tripo3D, or similar AI 3D generators.

### General prefix for all prompts

> Chunky cartoon low-poly 3D model, Subway Surfers style, bright saturated colors,
> simple shapes, smooth shading, white background, game asset, Y-up orientation

### Fuselage prompts

**bathtub**:
> {prefix}, classic claw-foot bathtub as a vehicle, four ornate gold legs,
> round porcelain rim, drain plug with chain, white porcelain body,
> hollow cockpit interior, side view

**shopping_cart**:
> {prefix}, supermarket shopping cart as a vehicle, wire mesh basket body,
> fold-out child seat, four caster wheels, handlebar at rear,
> metallic silver wire frame, isometric view

**rubber_duck**:
> {prefix}, giant rubber duck bath toy as a vehicle, hollow cockpit on back,
> round body, prominent orange beak, small black dot eyes,
> bright yellow body, three quarter view

**piano**:
> {prefix}, baby grand piano as a vehicle, lid propped open,
> black and white keys visible on one side, three chunky legs,
> glossy black body, seated position inside, isometric view

**boat**:
> {prefix}, chunky wooden rowboat as a vehicle, visible plank lines,
> small bench seat in middle, two oar locks on sides,
> light brown wood body, slightly curved hull, three quarter view

**rocket**:
> {prefix}, cardboard rocket ship as a vehicle, cone nose,
> three triangular tail fins, painted circle windows,
> visible duct tape seams, corrugated cardboard texture,
> beige/brown cardboard color, side view

### Wing prompts

**biplane**:
> {prefix}, biplane double wing set, two rectangular wings stacked,
> vertical struts connecting them, wire bracing, white canvas surface,
> visible rib bumps, front view showing both tiers

**delta**:
> {prefix}, paper airplane delta wings, swept back triangular,
> smooth surface, slightly thick cross-section, white,
> front view symmetric

**hang_glider**:
> {prefix}, hang glider wing, large triangular fabric sail,
> aluminum tube frame along leading edge and center keel,
> colorful fabric with silver tube frame, top-down view

**cardboard_box**:
> {prefix}, flat cardboard box flaps used as wings,
> visible corrugation texture, grey duct tape at root,
> brown cardboard color, front view showing both sides

**umbrella**:
> {prefix}, two open umbrellas used as wings, dome side up,
> visible spoke ribs, colorful fabric panels,
> mounted horizontally on sticks, front view

### Helmet prompts

**aviator**:
> {prefix}, leather aviator pilot cap with goggles on forehead,
> brown leather cap, amber-tinted round goggles raised up,
> chin strap, white base cap for color tinting, three quarter view

**construction**:
> {prefix}, cartoon construction hard hat helmet,
> rounded dome with front brim, chin strap,
> white base shell for color tinting, three quarter view

**viking**:
> {prefix}, cartoon viking helmet with two curving horns,
> round metal bowl shape, nose guard, bone-colored horns,
> white base metal for color tinting, three quarter view

**astronaut**:
> {prefix}, cartoon astronaut space helmet, bubble dome visor,
> gold reflective face shield, collar ring at neck,
> white base shell for color tinting, three quarter view

**chicken**:
> {prefix}, cartoon chicken head mask helmet, red comb on top,
> orange pointed beak, red wattle under chin, round eye holes,
> white base feathers for color tinting, three quarter view

---

## Environment Assets

These are separate from the craft builder but are used in the launch/flight scene.

| Asset | File | Dimensions | Polys | Description |
|-------|------|-----------|-------|-------------|
| **Pier** | `flugtag-env-pier.glb` | 20m L x 4m W x 3m H | 2000 | Wooden pier / dock extending over water, with vertical posts, planked surface, railing along sides. Crowd barriers on sides. |
| **Ramp** | `flugtag-env-ramp.glb` | 8m L x 3m W x 4m H | 500 | Launch ramp at end of pier, angled 25-30 degrees, wooden surface with side rails, "FLUGTAG" painted on side. |
| **Water plane** | Procedural | Infinite | N/A | Animated water surface using `THREE.PlaneGeometry` with vertex displacement, blue-green toon shaded. |
| **Crowd section** | `flugtag-env-crowd.glb` | 10m W x 3m D x 2m H | 1500 | Block of chunky spectators on bleachers, varied colors, some holding signs. Instance and repeat along pier. |
| **Birds** | `flugtag-env-bird.glb` | 0.3m wingspan | 50 | Single seagull, flapping wing animation. Spawn as flock of instances. |
| **Clouds** | Procedural | Varied | N/A | Fluffy toon clouds using merged `SphereGeometry` clusters. Placed at Y=20-40. |
| **Thermal column** | Procedural | 3m radius cylinder | N/A | Visible updraft using transparent cylinder with animated upward-scrolling texture or particles. |

---

## Collectible Assets

Floating items the craft can fly through during the flight phase.

| Asset | File | Dimensions | Polys | Description |
|-------|------|-----------|-------|-------------|
| **Star** | `flugtag-collect-star.glb` | 0.5m diameter | 100 | Spinning gold star, classic 5-pointed, slight glow bloom. +100 points. |
| **Wind boost** | `flugtag-collect-wind.glb` | 0.6m diameter | 80 | Swirling arrow ring / wind spiral in light blue. +speed boost for 2s. |
| **Style ring** | `flugtag-collect-style_ring.glb` | 1.5m diameter ring | 60 | Large ring to fly through, rainbow colored. +200 style points. |
| **Coin** | `flugtag-collect-coin.glb` | 0.3m diameter | 50 | Flat spinning coin with Red Bull logo. +50 points. |

---

## Blender Export Checklist

Follow these steps for every GLB export:

1. **Scale**: Ensure dimensions match the spec table above (1 unit = 1 meter)
2. **Origin**: Set origin to center-bottom of bounding box (`Ctrl+Shift+Alt+C > Origin to Geometry`, then move to bottom)
3. **Rotation**: Apply all transforms (`Ctrl+A > All Transforms`)
4. **Y-Up**: Blender uses Z-up; the glTF exporter handles the conversion automatically
5. **Forward -Z**: Default Blender glTF export uses -Z forward (correct)
6. **Materials**: Use Principled BSDF with base color only. Set Metallic=0, Roughness=0.8 for toon look
7. **Textures**: If using texture maps, pack into GLB. Keep 256x256 or 512x512
8. **Vertex colors**: Alternative to textures for simple parts. Use vertex paint mode
9. **Tintable parts**: Base color must be pure white (#FFFFFF)
10. **Naming**: Root object named `{Category}_{Type}` (e.g., `Fuselage_bathtub`)
11. **Poly count**: Check face count in viewport overlay. Stay under budget
12. **Export settings**:
    - Format: glTF Binary (.glb)
    - Include: Selected Objects only
    - Transform: +Y Up (default)
    - Geometry: Apply Modifiers, UVs, Normals, Vertex Colors
    - Animation: Include only if part has animation (propulsion spinning)
    - Compression: Enable Draco compression
13. **File naming**: `flugtag-{category}-{type}.glb`
14. **Test**: Load in Three.js viewer, verify scale, orientation, and material appearance
