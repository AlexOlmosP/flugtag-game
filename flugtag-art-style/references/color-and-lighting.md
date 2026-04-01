# Color & Lighting Reference

Complete color palette and lighting configuration for the Flugtag game. This document
defines every color in the game and provides copy-paste lighting rigs for each game state.

---

## Table of Contents

- [Master Color Palette](#master-color-palette)
- [Craft Accent Colors](#craft-accent-colors)
- [Lighting Rigs](#lighting-rigs)
  - [Flight / Outdoor Rig](#flight--outdoor-rig)
  - [Workshop Rig](#workshop-rig)
  - [Title Screen Rig](#title-screen-rig)
- [Environment Rendering](#environment-rendering)
  - [Sky Gradient](#sky-gradient)
  - [Water Surface](#water-surface)
  - [Fog & Atmospheric Perspective](#fog--atmospheric-perspective)
  - [Cloud Coloring](#cloud-coloring)
- [Color Rules by Object](#color-rules-by-object)
- [Atmospheric Perspective](#atmospheric-perspective)

---

## Master Color Palette

```javascript
const FLUGTAG_PALETTE = {
  // ─── Sky ───────────────────────────────────────────
  skyTop:         0x2A6FA8,  // deep blue at zenith
  skyMid:         0x4A90D9,  // mid sky blue
  skyBottom:      0x87CEEB,  // light sky near horizon
  skyHorizon:     0xB8D4E8,  // hazy horizon band

  // ─── Water ─────────────────────────────────────────
  oceanDeep:      0x005577,  // deep water far below
  oceanMid:       0x0088AA,  // mid-depth water
  oceanSurface:   0x00AACC,  // bright surface water
  oceanFoam:      0xCCEEEE,  // foam and wave crests

  // ─── Pier & Ramp ──────────────────────────────────
  pierWood:       0xB88855,  // warm wood brown
  pierWoodDark:   0x8A6633,  // shadow plank color
  pierMetal:      0x888899,  // metal bolts and rails
  rampSurface:    0xCC4444,  // bold red launch ramp
  rampStripes:    0xFFCC00,  // caution stripes on ramp edge

  // ─── Crowd ─────────────────────────────────────────
  crowdBase:      0xFFCC88,  // warm skin base tone
  crowdShirt1:    0xFF6644,  // orange shirts
  crowdShirt2:    0x44AAFF,  // blue shirts
  crowdShirt3:    0xFFDD22,  // yellow shirts
  crowdShirt4:    0x66DD66,  // green shirts
  confetti: [
    0xFF4444, 0xFF8844, 0xFFDD44, 0x44DD44,
    0x44AAFF, 0xAA44FF, 0xFF44AA, 0xFFFFFF
  ],

  // ─── Clouds ────────────────────────────────────────
  cloudWhite:     0xF0F4FF,  // bright cloud face
  cloudShadow:    0xB8C8DD,  // shadow side of clouds
  cloudHighlight: 0xFFFFFF,  // sun-facing highlight

  // ─── Atmosphere & Effects ──────────────────────────
  thermalShimmer: 0xFFDD88,  // warm thermal column particles
  windTrail:      0xFFFFFF,  // white wind streak particles
  splashDrop:     0x88DDEE,  // splash water droplets
  splashFoam:     0xCCEEEE,  // splash foam ring

  // ─── Birds / Hazards ──────────────────────────────
  birdBody:       0x333333,  // dark bird body (silhouette reads)
  birdWing:       0x444444,  // slightly lighter wing
  birdBeak:       0xFF8800,  // orange beak accent

  // ─── Collectibles ─────────────────────────────────
  ringGold:       0xFFDD44,  // golden score ring
  ringGlow:       0xFFEE88,  // ring glow aura
  boostOrb:       0x44EEFF,  // speed boost collectible
  boostGlow:      0x88FFFF,  // boost glow aura

  // ─── UI ────────────────────────────────────────────
  uiPrimary:      0x1A3A5C,  // dark navy for panels
  uiAccent:       0xFF4444,  // Red Bull red accent
  uiText:         0xFFFFFF,  // white text
  uiTextShadow:   0x000000,  // text shadow for readability
  uiSuccess:      0x44DD44,  // green success
  uiWarning:      0xFFAA22,  // amber warning
  uiDanger:       0xFF4444,  // red danger
};
```

### Palette Visual Map

```
  Sky (top to bottom):
  ┌──────────────────────────────┐
  │ #2A6FA8  skyTop (zenith)     │  ████████
  │ #4A90D9  skyMid              │  ████████
  │ #87CEEB  skyBottom           │  ████████
  │ #B8D4E8  skyHorizon          │  ████████
  ├──────────────────────────────┤
  │ #00AACC  oceanSurface        │  ████████  ← water line
  │ #0088AA  oceanMid            │  ████████
  │ #005577  oceanDeep            │  ████████
  └──────────────────────────────┘

  Objects:
  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │ Pier    │  │ Craft   │  │ Birds   │
  │ #B88855 │  │ (vivid) │  │ #333333 │
  │ warm    │  │ LOUDEST │  │ dark    │
  │ brown   │  │ colors  │  │ reads   │
  └─────────┘  └─────────┘  └─────────┘
```

---

## Craft Accent Colors

Players can choose from these bold accent colors for their craft. All are highly saturated
and designed to pop against both sky and water backgrounds.

```javascript
const CRAFT_COLORS = {
  // Primary colors
  fireRed:        0xFF3333,  // classic Red Bull craft
  royalBlue:      0x3366FF,  // bold blue
  sunYellow:      0xFFCC00,  // bright yellow

  // Secondary colors
  limeGreen:      0x44DD44,  // vivid green
  hotOrange:      0xFF7722,  // traffic orange
  electricPurple: 0xAA44FF,  // punchy purple

  // Fun / novelty
  hotPink:        0xFF44AA,  // flashy pink
  cyberCyan:      0x00DDEE,  // neon cyan
  mintGreen:      0x44DDAA,  // minty fresh
  coralRed:       0xFF6655,  // warm coral

  // Neutrals (for contrast builds)
  cleanWhite:     0xF0F0F0,  // bright white (not pure, reads better)
  stormGray:      0x667788,  // cool gray with blue tint

  // Special / unlockable
  goldChrome:     0xDDAA22,  // metallic gold look
  skyPaint:       0x4488CC,  // sky-matching camouflage (stealth!)
  rainbowBase:    0xFF4444,  // base for animated rainbow shift
  nightBlack:     0x222233,  // dark with blue tint (not pure black)
};
```

### Color Selection Rules

- **Minimum 2 colors per craft**: body color + accent color for wings/details
- **Body is always the LOUDEST**: most saturated, largest surface area
- **Wings can match or contrast**: either same color family or complementary
- **Decorations use the accent**: flags, stickers, trim in the second color
- **Never pure black (0x000000)**: always tint with a hue so toon shading reads
- **Never pure white (0xFFFFFF)**: use 0xF0F0F0 or tinted white

---

## Lighting Rigs

### Flight / Outdoor Rig

The primary lighting setup used during gameplay. Simulates a bright sunny day over water.

```javascript
/**
 * Sets up the outdoor flight lighting rig.
 * Use this for the main gameplay flight sequence.
 *
 * Light hierarchy:
 *   Sun (DirectionalLight) ─── main illumination, hard shadows
 *   Sky fill (HemisphereLight) ─── sky blue top, ocean teal bottom
 *   Ambient (AmbientLight) ─── prevents pure black shadows
 */
function setupFlightLighting(scene) {
  // ─── Sun: main directional light ───
  const sun = new THREE.DirectionalLight(0xFFF5E0, 1.2);
  sun.position.set(30, 50, 20);  // high and to the right
  sun.castShadow = true;

  // Shadow map settings
  sun.shadow.mapSize.width = 1024;
  sun.shadow.mapSize.height = 1024;
  sun.shadow.camera.near = 1;
  sun.shadow.camera.far = 100;
  sun.shadow.camera.left = -30;
  sun.shadow.camera.right = 30;
  sun.shadow.camera.top = 30;
  sun.shadow.camera.bottom = -30;
  sun.shadow.bias = -0.001;

  scene.add(sun);

  // ─── Hemisphere: sky-blue top, ocean-teal bottom ───
  const hemi = new THREE.HemisphereLight(
    0x87CEEB,  // sky color (top)
    0x006688,  // ocean bounce color (bottom)
    0.5        // moderate intensity
  );
  scene.add(hemi);

  // ─── Ambient: minimal fill to prevent pure black ───
  const ambient = new THREE.AmbientLight(0x404060, 0.3);
  scene.add(ambient);

  return { sun, hemi, ambient };
}
```

```
  Lighting Diagram (flight/outdoor):

        SUN ☀️ (0xFFF5E0, 1.2)
         \
          \  30,50,20
           \
            \
  ══════════╗═══════════════════  ← pier level
             ╲   craft
              ╲  ┌───┐
               ╲ │   │
                 └───┘
  ~~~~~~~~~~~~~~~~~~~~~~~~~~  ← water level (y=0)

  ↑ HemisphereLight ↑
  sky blue (0x87CEEB) on top faces
  ocean teal (0x006688) on bottom faces

  + AmbientLight (0x404060, 0.3) everywhere
```

### Workshop Rig

Used in the craft-building screen. Warmer, more focused, like a garage workshop.

```javascript
/**
 * Sets up workshop lighting for the craft builder.
 * Warm key light + cool fill + subtle rim for depth.
 */
function setupWorkshopLighting(scene) {
  // ─── Key light: warm spotlight from above-right ───
  const key = new THREE.SpotLight(0xFFE8CC, 1.5);
  key.position.set(5, 8, 5);
  key.target.position.set(0, 1, 0); // aim at craft center
  key.angle = Math.PI / 4;
  key.penumbra = 0.3;
  key.castShadow = true;
  key.shadow.mapSize.width = 1024;
  key.shadow.mapSize.height = 1024;
  scene.add(key);
  scene.add(key.target);

  // ─── Fill light: cool blue from left ───
  const fill = new THREE.DirectionalLight(0x8888CC, 0.4);
  fill.position.set(-5, 3, 2);
  scene.add(fill);

  // ─── Rim light: white from behind for edge definition ───
  const rim = new THREE.DirectionalLight(0xFFFFFF, 0.6);
  rim.position.set(0, 4, -5);
  scene.add(rim);

  // ─── Ambient: low fill ───
  const ambient = new THREE.AmbientLight(0x333344, 0.3);
  scene.add(ambient);

  return { key, fill, rim, ambient };
}
```

```
  Lighting Diagram (workshop):

              KEY (warm spot)
               │
               │ 5,8,5
               ▼
  FILL ───→  ┌─────────┐  ←─── RIM
  (cool blue) │  CRAFT  │  (white, behind)
  -5,3,2     │  being   │  0,4,-5
              │  built   │
              └─────────┘
                   │
              ─────┴─────  (floor/turntable)
```

### Title Screen Rig

Dramatic sunset lighting for the title/menu screen.

```javascript
/**
 * Sets up dramatic sunset lighting for title screen.
 * Warm orange sun low on horizon, cool blue fill, dramatic mood.
 */
function setupTitleLighting(scene) {
  // ─── Low sun: warm orange from horizon ───
  const sun = new THREE.DirectionalLight(0xFF9944, 1.5);
  sun.position.set(50, 8, 0);  // low on horizon
  sun.castShadow = true;
  sun.shadow.mapSize.width = 1024;
  sun.shadow.mapSize.height = 1024;
  scene.add(sun);

  // ─── Sky fill: deep blue overhead ───
  const skyFill = new THREE.HemisphereLight(
    0x334488,  // deep blue sky (sunset)
    0x442200,  // warm ground bounce
    0.4
  );
  scene.add(skyFill);

  // ─── Backlight: golden rim on craft silhouette ───
  const backlight = new THREE.DirectionalLight(0xFFCC66, 0.8);
  backlight.position.set(-30, 10, 0);
  scene.add(backlight);

  // ─── Ambient: very low for drama ───
  const ambient = new THREE.AmbientLight(0x222233, 0.2);
  scene.add(ambient);

  return { sun, skyFill, backlight, ambient };
}
```

```
  Lighting Diagram (title screen):

  BACKLIGHT ←──  ┌─────────┐  ──→ LOW SUN (orange)
  (golden rim)    │  CRAFT  │     at horizon level
  -30,10,0       │silhouette│     50,8,0
                  └─────────┘
                       │
  ~~~~~~~~~~~~~~~~~~~~~│~~~~~  ← water with sunset reflection
                       │
           ↑ HemisphereLight ↑
           deep blue top / warm bottom

  Mood: dramatic sunset, craft dark against bright sky
```

### Sunset Sky Palette Override

When using the title screen rig, override the sky colors:

```javascript
const SUNSET_SKY_PALETTE = {
  skyTop:     0x1A2A55,  // deep navy overhead
  skyMid:     0x445588,  // dark blue-purple
  skyBottom:  0xFF8844,  // warm orange at horizon
  skyHorizon: 0xFFCC88,  // golden glow
};
```

---

## Environment Rendering

### Sky Gradient

The sky is a vertex-colored inverted sphere. See the main SKILL.md for full implementation.

**Key color transitions (from top to bottom):**

```
  Zenith (y = 500):     #2A6FA8  deep saturated blue
       │
       │  smooth blend
       ▼
  Upper (y ~ 325):      #4A90D9  mid blue
       │
       │  smooth blend
       ▼
  Lower (y ~ 225):      #87CEEB  light sky blue
       │
       │  smooth blend
       ▼
  Horizon (y ~ 0):      #B8D4E8  hazy pale blue
       │
       │  (below horizon, not normally visible)
       ▼
  Below (y = -500):     #87CEEB  matches horizon to avoid seam
```

**Rules:**
- Sky sphere radius should be 500 (large enough that parallax is imperceptible)
- Use `MeshBasicMaterial` with `vertexColors: true` -- sky must NOT respond to lights
- Set `fog: false` on the sky material so fog doesn't wash it out
- Set `depthWrite: false` and `renderOrder: -100` so everything draws in front

### Water Surface

See the full toon water implementation in `SKILL.md`. Key color rules:

```
  Wave crest / foam:    #CCEEEE  (oceanFoam)   ← brightest, white-ish
  Surface:              #00AACC  (oceanSurface) ← vivid teal
  Mid-depth:            #0088AA  (oceanMid)     ← darker teal
  Deep:                 #005577  (oceanDeep)    ← darkest, greenish-blue
```

**Water color must always contrast with sky color.** The sky uses pure blues (#2A6FA8 range)
while water uses blue-greens / teals (#0088AA range). This prevents the horizon from being
an indistinguishable blur.

### Fog & Atmospheric Perspective

Use exponential fog to fade distant objects toward the sky horizon color:

```javascript
// Subtle atmospheric perspective fog
scene.fog = new THREE.FogExp2(
  0xB8D4E8,  // skyHorizon color -- objects fade toward this
  0.003       // very gentle density
);

// IMPORTANT: Exclude sky and water from fog
// Sky: set material.fog = false
// Water: set material.fog = false (or handle in custom shader)
```

**Fog Rules:**
- Fog color must match `skyHorizon` (#B8D4E8) for outdoor scenes
- Density should be very low (0.002 - 0.005) -- this is a clear sunny day
- Fog creates depth without blocking visibility of gameplay elements
- Near objects (0-20m): full vivid color
- Mid objects (20-50m): slightly faded
- Far objects (50m+): noticeably desaturated, blending toward sky

### Cloud Coloring

Clouds use MeshToonMaterial for consistent shading but are colored to feel soft:

```javascript
// Cloud material (no outline!)
const cloudMat = createToonMaterial(0xF0F4FF, { steps: 3 });

// Shadow-facing puffs in a cloud cluster can use:
const cloudShadowMat = createToonMaterial(0xB8C8DD, { steps: 3 });
```

**Cloud Rules:**
- Sun-facing puffs: `cloudWhite` (#F0F4FF) -- almost white with blue tint
- Shadow puffs: `cloudShadow` (#B8C8DD) -- muted blue-gray
- NO outlines on clouds -- they should feel soft and atmospheric
- Clouds are NOT interactive (no collision, no gameplay effect)
- Place at y = 20-60 (above typical flight range of 5-30)

---

## Color Rules by Object

### Priority Hierarchy

```
  MOST SATURATED ──────────────────── LEAST SATURATED
  ┌──────────┬──────────┬──────────┬──────────┬──────────┐
  │  Player  │Collecti- │   Pier   │  Crowd   │  Sky /   │
  │  Craft   │  bles    │  / Ramp  │  / Birds │  Water   │
  │          │          │          │          │(environ) │
  │ LOUDEST  │ Glowing  │  Warm    │Background│  Calm    │
  │ boldest  │ pulsing  │  grounded│  muted   │ backdrop │
  └──────────┴──────────┴──────────┴──────────┴──────────┘
```

### Per-Object Color Rules

**Player Craft:**
- The MOST colorful object in any frame
- Uses bold saturated colors from `CRAFT_COLORS` palette
- Minimum 2 colors (body + accent)
- Must contrast against both sky AND water
- If craft is blue, wings/accent must be contrasting (orange, yellow, red)

**Collectible Rings:**
- Golden (#FFDD44) with emissive glow
- Pulsing additive sprite behind for fake bloom
- Should feel "inviting" -- warm, glowing, rewarding

**Pier / Ramp:**
- Warm wood brown (#B88855) -- feels grounded, structural
- Dark planks (#8A6633) for subtle variation (not a texture, just alternating faces)
- Metal accents (#888899) for bolts, rails
- Red ramp surface (#CC4444) with yellow caution stripes (#FFCC00)
- Contrasts against cool water below

**Birds / Hazards:**
- Dark body (#333333) for strong silhouette
- Orange beak (#FF8800) as recognizable detail
- Must read instantly as "threat" against sky
- Dark color ensures outline is visible even against dark sky

**Crowd:**
- Warm skin tone base (#FFCC88)
- Varied shirt colors pulled from crowd palette
- Confetti uses the full rainbow confetti array
- Muted compared to craft -- crowd is background detail

**Water:**
- Cool blue-green tones (teals, cyans)
- Must be VISUALLY DISTINCT from sky blues
- Foam white on wave crests
- Gentle animated color shifts, not static

**Sky:**
- Pure blues (no green/teal tint -- that's water's territory)
- Gradient from dark saturated at top to light/hazy at bottom
- Horizon band is pale and slightly warm

---

## Atmospheric Perspective

Atmospheric perspective makes near objects vivid and far objects muted, creating depth.

### Distance-Based Color Shifting

```javascript
/**
 * Adjusts material color based on distance from camera.
 * Blends toward sky horizon color at distance.
 *
 * @param {THREE.Material} material - Material to adjust
 * @param {number} distance - Distance from camera
 * @param {number} maxDistance - Distance at which full fade occurs
 */
function applyAtmosphericPerspective(material, distance, maxDistance = 80) {
  const horizonColor = new THREE.Color(0xB8D4E8);
  const t = Math.min(distance / maxDistance, 1.0);

  // Use a power curve for more natural falloff
  const fade = Math.pow(t, 1.5);

  // Store original color if not already saved
  if (!material.userData.originalColor) {
    material.userData.originalColor = material.color.clone();
  }

  // Blend toward horizon
  material.color.copy(material.userData.originalColor).lerp(horizonColor, fade * 0.4);
}
```

### Depth Zones

```
  ┌─────────────┬──────────┬─────────────────────────────────┐
  │ Zone        │ Distance │ Color Treatment                 │
  ├─────────────┼──────────┼─────────────────────────────────┤
  │ Foreground  │ 0 - 15m  │ Full vivid palette colors       │
  │ Midground   │ 15 - 40m │ Slightly desaturated (10-20%)   │
  │ Background  │ 40 - 80m │ Noticeably muted (30-40% fade)  │
  │ Far BG      │ 80m+     │ Mostly sky color (fog handles)  │
  └─────────────┴──────────┴─────────────────────────────────┘
```

### Visual Effect

```
  Camera ──────────────────────────────────► Distance

  ████████  ▓▓▓▓▓▓▓▓  ░░░░░░░░  ........
  VIVID     slightly   muted      faded
  full      faded      toward     almost
  color     colors     horizon    invisible

  0m        15m        40m        80m+
```

**Implementation note:** The fog system (`FogExp2`) handles most of this automatically. The
`applyAtmosphericPerspective()` function is for cases where you want finer control on
specific objects, or where the fog density isn't enough for the desired effect.

### Rules

- The player's own craft is ALWAYS at full vivid color (never apply atmospheric fade to it)
- The pier fades only slightly since it's the launch reference point
- Birds should still be identifiable at midground distances
- Clouds beyond 50m can be very faint (this looks natural)
- The water surface doesn't fade -- it's always at the camera's feet
