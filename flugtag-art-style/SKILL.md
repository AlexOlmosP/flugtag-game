---
name: flugtag-art-style
description: >
  Visual style bible for the Red Bull Flugtag HTML game. Invoke this skill when working on:
  visual style, toon shading, cel shading, outlines, color palette, lighting setup,
  materials, sky rendering, water effects, water shading, cartoony look, GLB model matching,
  post-processing, visual consistency, "doesn't look right", too realistic, too flat, too dark,
  splash effects, wind visualization, thermal columns, crowd rendering, pier appearance,
  craft colors, cloud rendering, atmospheric perspective, gradient sky, foam trails.
---

# Flugtag Art Style & Toon Shading Skill

This document is the **visual bible** for the entire Flugtag game. Every visual decision --
materials, colors, lighting, outlines, sky, water, particles, UI -- flows from the principles
defined here. All other skills (craft-builder, flight-physics, crowd-system, etc.) **defer to
this skill** for anything appearance-related. If something "doesn't look right," start here.

The target aesthetic is **bold, chunky, colorful cartoon** -- think Subway Surfers or
Overcooked but set in an outdoor sky-and-water environment with a pier launch ramp, flying
machines, cheering crowds, and open ocean below.

---

## References

| File | Purpose |
|------|---------|
| [`references/toon-shading.md`](references/toon-shading.md) | MeshToonMaterial setup, gradient maps, inverted hull outlines, GLB toonification pipeline |
| [`references/color-and-lighting.md`](references/color-and-lighting.md) | Master color palette, lighting rigs, sky/water rendering, atmospheric perspective |

---

## 7 Core Visual Principles

### 1. Bold Outlines on Everything

Every solid game object gets a visible black outline using the **inverted hull method**
(scaled-up back-face mesh behind the original).

```
┌─────────────────────────────────────────────────┐
│              Outline Thickness Rules             │
├──────────────────────┬──────────────────────────┤
│ Object               │ Thickness (% of size)    │
├──────────────────────┼──────────────────────────┤
│ Player craft         │ 3-4%  (hero object)      │
│ Craft decorations    │ 2%    (wings, props)     │
│ Birds / hazards      │ 2%    (must read clearly)│
│ Pier / ramp          │ 1.5%  (scenery)          │
│ Crowd figures        │ 1.5%  (background)       │
│ Clouds               │ 0%    (no outlines)      │
│ Water surface        │ 0%    (no outlines)      │
│ Sky                  │ 0%    (no outlines)      │
│ Particles / trails   │ 0%    (no outlines)      │
│ UI elements          │ CSS borders instead      │
└──────────────────────┴──────────────────────────┘
```

**Rule:** If it's a solid 3D object the player interacts with or looks at, it gets an outline.
If it's atmosphere, environment backdrop, or a particle, it does NOT.

### 2. Flat Color with Hard Shadow Steps

All game objects use **MeshToonMaterial** with a **3-step gradient map** producing distinct
lit / mid / shadow bands. No smooth gradients on objects. Shadows snap from one tone to the
next like a comic book panel.

```
  Light Band    Mid Band     Shadow Band
  ┌──────────┬──────────┬──────────┐
  │ 100% sat │  70% sat │  40% sat │
  │ bright   │  medium  │  dark    │
  └──────────┴──────────┴──────────┘
       ↑           ↑          ↑
    sunlit      angled     away from sun
```

### 3. Chunky Exaggerated Proportions

Nothing is to-scale. Everything is **rounder, fatter, shorter** than real life.

```
  Real proportions          Flugtag proportions
  ┌──────────────┐          ┌────────────────────┐
  │  ═══════════ │          │  ══════════════════ │  ← wings WIDER
  │    ┌────┐    │          │      ┌──────┐      │  ← fuselage STUBBIER
  │    │    │    │          │      │      │      │
  │    └────┘    │          │      └──────┘      │
  └──────────────┘          └────────────────────┘

  Pier: cartoonishly tall with thick wooden beams
  Birds: round puffy bodies, tiny wings
  Crowd: bobblehead proportions (big heads, small bodies)
```

### 4. Vibrant Saturated Colors

The palette is **loud and happy**. Sky is a rich blue, ocean is vivid teal, the pier is warm
golden-brown, and player craft are the most colorful objects in the scene.

- Sky: rich blues from deep (#2A6FA8) at zenith to light (#87CEEB) at horizon
- Water: vivid teals and cyans (#0088AA) with white foam highlights
- Pier/ramp: warm wood brown (#B88855) with red ramp surface (#CC4444)
- Craft: the MOST saturated objects -- bold primaries and secondaries
- Crowd: warm skin tones with confetti pops of color

> See [`references/color-and-lighting.md`](references/color-and-lighting.md) for the full
> palette with hex codes.

### 5. Minimal Texture Detail

Surfaces are **flat color**, not photographic textures. Wood grain is implied by slight color
variation, not a photograph. Water shimmer is animated color, not a normal map.

```
  ✗ BAD:  Photographic wood texture on pier
  ✓ GOOD: Flat warm brown with subtle darker planks (color only)

  ✗ BAD:  Realistic ocean normal map with specular highlights
  ✓ GOOD: Flat teal plane with animated color bands and foam lines

  ✗ BAD:  Detailed feather texture on birds
  ✓ GOOD: Solid color with toon shadow steps
```

### 6. Readable Silhouettes

Every object must be **instantly identifiable** from its outline shape alone, especially
against the sky backdrop.

```
  Against sky:           Against water:
  ┌─────────────────┐    ┌─────────────────┐
  │     ╔═══╗       │    │~~~~~~~~~~~~~~~~~│
  │  ═══╣   ╠═══    │    │~~╔═══╗~~~~~~~~~~│
  │     ╚═╦═╝       │    │~~║   ║~~~~~~~~~~│
  │       ║         │    │~~╚═══╝~~~~~~~~~~│
  │  (craft in sky) │    │  (craft on water)│
  └─────────────────┘    └─────────────────┘

  Craft:  distinct wing+body shape, unique per build
  Birds:  round body + flapping wings = HAZARD
  Pier:   tall vertical structure = LAUNCH POINT
  Clouds: big puffy blobs = SCENERY (no collision)
```

### 7. Consistent Scale Reference

All objects follow a fixed scale so the world feels coherent:

```
┌─────────────────────────────────────────────────┐
│              Scale Reference Chart               │
├──────────────────────┬──────────────────────────┤
│ Object               │ Approximate Size         │
├──────────────────────┼──────────────────────────┤
│ Player craft body    │ ~2m long, ~1m tall       │
│ Craft wingspan       │ ~3m (exaggerated)        │
│ Pilot figure         │ ~0.8m (chibi)            │
│ Pier height          │ ~10m above water         │
│ Pier ramp length     │ ~8m                      │
│ Birds                │ ~0.3m body               │
│ Crowd figures        │ ~0.6m (distant, chibi)   │
│ Clouds               │ ~5-15m diameter          │
│ Water tile            │ 200m x 200m minimum     │
│ Collectible rings    │ ~1.5m diameter           │
└──────────────────────┴──────────────────────────┘
```

---

## Material Decision Tree

Use this flowchart to pick the right material for ANY object:

```
                    ┌─────────────┐
                    │ What is it? │
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
      ┌──────────┐  ┌──────────┐  ┌──────────────┐
      │ Sky/Water│  │ Game Obj │  │ UI / Glow /  │
      │ Backdrop │  │ (solid)  │  │ Particles    │
      └────┬─────┘  └────┬─────┘  └──────┬───────┘
           │              │               │
           ▼              ▼               ▼
   MeshBasicMaterial  MeshToonMaterial  Depends:
   (unlit, flat       + gradient map    ├─ UI platforms → MeshStandardMaterial
    color, no         + inverted hull   ├─ Glow rings  → MeshBasicMaterial
    shadows)          outline           │                 + additive blending
                                        ├─ Particles   → PointsMaterial or
                                        │                 SpriteMaterial
                                        └─ Trails      → MeshBasicMaterial
                                                          + opacity fade
```

**Key rules:**
- `MeshToonMaterial` = default for any solid interactive/visible object
- `MeshBasicMaterial` = sky, water, and anything that should ignore lighting
- `MeshStandardMaterial` = only for UI platforms that need subtle reflections
- Never use `MeshPhongMaterial` or `MeshLambertMaterial` -- they break toon style
- Additive blending for anything that should glow (collectible rings, splash highlights)

---

## Post-Processing Pipeline

We keep post-processing **minimal** for performance in single-file HTML artifacts.

### FXAA Anti-Aliasing

Smooths the hard edges from toon shading outlines:

```javascript
// Post-processing with EffectComposer
const composer = new THREE.EffectComposer(renderer);
const renderPass = new THREE.RenderPass(scene, camera);
composer.addPass(renderPass);

const fxaaPass = new THREE.ShaderPass(THREE.FXAAShader);
fxaaPass.uniforms['resolution'].value.set(
  1 / window.innerWidth,
  1 / window.innerHeight
);
composer.addPass(fxaaPass);

// In render loop: composer.render() instead of renderer.render()
```

### CSS Vignette (Zero GPU Cost)

```css
.vignette-overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100%; height: 100%;
  pointer-events: none;
  z-index: 10;
  background: radial-gradient(
    ellipse at center,
    transparent 60%,
    rgba(0, 20, 40, 0.3) 100%
  );
}
```

### Fake Bloom on Collectibles

Instead of real bloom (expensive), scale-pulse a bright additive sprite behind collectibles:

```javascript
function createCollectibleGlow(position) {
  const glowGeo = new THREE.PlaneGeometry(2.5, 2.5);
  const glowMat = new THREE.MeshBasicMaterial({
    color: 0xFFDD44,
    transparent: true,
    opacity: 0.4,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    side: THREE.DoubleSide
  });
  const glow = new THREE.Mesh(glowGeo, glowMat);
  glow.position.copy(position);

  // Pulse animation in update loop
  glow.userData.update = (time) => {
    const pulse = 1.0 + 0.15 * Math.sin(time * 3.0);
    glow.scale.setScalar(pulse);
    glow.material.opacity = 0.3 + 0.15 * Math.sin(time * 3.0 + 1.0);
    // Always face camera
    glow.lookAt(camera.position);
  };

  return glow;
}
```

---

## Water Rendering

The water surface is a **stylized toon plane** -- NOT realistic ocean simulation. It should
look like a Ghibli movie ocean: flat color bands with gentle animation.

### Toon Water Implementation

```javascript
function createToonWater() {
  // Large plane below the pier
  const waterGeo = new THREE.PlaneGeometry(200, 200, 40, 40);
  waterGeo.rotateX(-Math.PI / 2);

  // Custom shader for animated toon water
  const waterMat = new THREE.ShaderMaterial({
    uniforms: {
      uTime:         { value: 0 },
      uDeepColor:    { value: new THREE.Color(0x005577) },
      uMidColor:     { value: new THREE.Color(0x0088AA) },
      uSurfaceColor: { value: new THREE.Color(0x00AACC) },
      uFoamColor:    { value: new THREE.Color(0xCCEEEE) },
      uSunDirection: { value: new THREE.Vector3(0.5, 0.8, 0.3).normalize() }
    },
    vertexShader: `
      uniform float uTime;
      varying vec2 vUv;
      varying float vWave;

      void main() {
        vUv = uv;

        // Gentle wave displacement
        vec3 pos = position;
        float wave1 = sin(pos.x * 0.3 + uTime * 0.8) * 0.3;
        float wave2 = sin(pos.z * 0.5 + uTime * 0.6) * 0.2;
        float wave3 = sin((pos.x + pos.z) * 0.2 + uTime * 1.1) * 0.15;
        pos.y += wave1 + wave2 + wave3;

        vWave = (wave1 + wave2 + wave3 + 0.65) / 1.3; // normalize 0-1

        gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
      }
    `,
    fragmentShader: `
      uniform vec3 uDeepColor;
      uniform vec3 uMidColor;
      uniform vec3 uSurfaceColor;
      uniform vec3 uFoamColor;
      uniform float uTime;
      varying vec2 vUv;
      varying float vWave;

      void main() {
        // Step-based color bands (toon style)
        vec3 color;
        if (vWave > 0.75) {
          color = uFoamColor;          // wave crests = foam
        } else if (vWave > 0.45) {
          color = uSurfaceColor;       // upper surface
        } else if (vWave > 0.2) {
          color = uMidColor;           // mid depth
        } else {
          color = uDeepColor;          // deep water
        }

        // Animated foam lines
        float foamLine = step(0.92, sin(vUv.x * 80.0 + uTime * 2.0) *
                                    sin(vUv.y * 60.0 + uTime * 1.5));
        color = mix(color, uFoamColor, foamLine * 0.5);

        gl_FragColor = vec4(color, 0.95);
      }
    `,
    transparent: true,
    side: THREE.DoubleSide
  });

  const water = new THREE.Mesh(waterGeo, waterMat);
  water.position.y = 0; // water level = y:0
  water.renderOrder = -1;

  water.userData.update = (time) => {
    waterMat.uniforms.uTime.value = time;
  };

  return water;
}
```

### Water Style Rules

- Water level is always `y = 0`
- Color bands shift gently over time (never static)
- Foam lines appear at wave crests
- NO reflections, NO normal maps, NO specular highlights
- Transparency ~0.95 (almost opaque with slight depth hint)
- NO outline on the water mesh

---

## Sky Rendering

The sky is a **gradient dome or background plane** -- not a skybox texture.

### Gradient Sky Implementation

```javascript
function createGradientSky() {
  const skyGeo = new THREE.SphereGeometry(500, 32, 16);
  // Flip faces inward
  skyGeo.scale(-1, 1, 1);

  // Vertex colors for gradient
  const colors = [];
  const positions = skyGeo.attributes.position;

  const topColor    = new THREE.Color(0x2A6FA8);  // deep blue zenith
  const midColor    = new THREE.Color(0x4A90D9);  // mid sky
  const bottomColor = new THREE.Color(0x87CEEB);  // light horizon
  const horizonColor = new THREE.Color(0xB8D4E8); // hazy horizon band

  for (let i = 0; i < positions.count; i++) {
    const y = positions.getY(i);
    // Normalize y from -500..500 to 0..1
    const t = (y + 500) / 1000;

    const color = new THREE.Color();
    if (t > 0.65) {
      // Upper sky: top to mid
      color.lerpColors(midColor, topColor, (t - 0.65) / 0.35);
    } else if (t > 0.45) {
      // Mid sky: bottom to mid
      color.lerpColors(bottomColor, midColor, (t - 0.45) / 0.2);
    } else {
      // Horizon band: horizon to bottom
      color.lerpColors(horizonColor, bottomColor, t / 0.45);
    }

    colors.push(color.r, color.g, color.b);
  }

  skyGeo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3));

  const skyMat = new THREE.MeshBasicMaterial({
    vertexColors: true,
    side: THREE.BackSide,
    depthWrite: false,
    fog: false
  });

  const sky = new THREE.Mesh(skyGeo, skyMat);
  sky.renderOrder = -100; // render first
  return sky;
}
```

### Cloud Rendering

Clouds are **simple billboard sprites or low-poly puffball meshes**:

```javascript
function createCloud(x, y, z, scale) {
  const group = new THREE.Group();
  const cloudColor = 0xF0F4FF;
  const shadowColor = 0xB8C8DD;

  // 3-5 overlapping spheres for puffy shape
  const puffCount = 3 + Math.floor(Math.random() * 3);
  for (let i = 0; i < puffCount; i++) {
    const radius = (0.6 + Math.random() * 0.6) * scale;
    const geo = new THREE.SphereGeometry(radius, 8, 6);
    const mat = new THREE.MeshToonMaterial({
      color: cloudColor,
      gradientMap: createGradientMap(3)
    });

    const puff = new THREE.Mesh(geo, mat);
    puff.position.set(
      (Math.random() - 0.5) * scale * 2,
      (Math.random() - 0.3) * scale * 0.5,
      (Math.random() - 0.5) * scale * 1.5
    );
    group.add(puff);
  }

  group.position.set(x, y, z);

  // Slow drift animation
  group.userData.update = (time) => {
    group.position.x += 0.003;
    if (group.position.x > 120) group.position.x = -120;
  };

  return group;
}
```

---

## Splash Effects

When a craft hits the water, there's a satisfying **cartoon splash**.

### Splash Particle System

```javascript
function createSplash(position) {
  const splashGroup = new THREE.Group();
  splashGroup.position.copy(position);
  splashGroup.position.y = 0; // water level

  const droplets = [];
  const dropletCount = 20;

  for (let i = 0; i < dropletCount; i++) {
    const size = 0.1 + Math.random() * 0.2;
    const geo = new THREE.SphereGeometry(size, 6, 4);
    const mat = new THREE.MeshBasicMaterial({
      color: 0x88DDEE,
      transparent: true,
      opacity: 0.8
    });
    const drop = new THREE.Mesh(geo, mat);

    // Radial burst pattern
    const angle = Math.random() * Math.PI * 2;
    const speed = 2 + Math.random() * 4;
    const upSpeed = 3 + Math.random() * 5;

    drop.userData.velocity = new THREE.Vector3(
      Math.cos(angle) * speed,
      upSpeed,
      Math.sin(angle) * speed
    );

    drop.position.set(0, 0.1, 0);
    splashGroup.add(drop);
    droplets.push(drop);
  }

  // Central splash column (ring that expands outward)
  const ringGeo = new THREE.RingGeometry(0.1, 0.5, 16);
  ringGeo.rotateX(-Math.PI / 2);
  const ringMat = new THREE.MeshBasicMaterial({
    color: 0xCCEEEE,
    transparent: true,
    opacity: 0.7,
    side: THREE.DoubleSide,
    depthWrite: false
  });
  const ring = new THREE.Mesh(ringGeo, ringMat);
  splashGroup.add(ring);

  let elapsed = 0;
  splashGroup.userData.update = (delta) => {
    elapsed += delta;

    // Animate droplets
    droplets.forEach(drop => {
      drop.position.add(
        drop.userData.velocity.clone().multiplyScalar(delta)
      );
      drop.userData.velocity.y -= 12 * delta; // gravity
      drop.material.opacity = Math.max(0, 0.8 - elapsed * 0.8);
      drop.scale.multiplyScalar(0.98);
    });

    // Expand ring
    const ringScale = 1 + elapsed * 6;
    ring.scale.set(ringScale, 1, ringScale);
    ring.material.opacity = Math.max(0, 0.7 - elapsed * 0.7);

    // Remove after 1.5 seconds
    if (elapsed > 1.5) {
      splashGroup.parent?.remove(splashGroup);
    }
  };

  return splashGroup;
}
```

### Splash Style Rules

- Droplets are **round spheres**, not realistic water sim
- Colors match the water palette (teals, light cyans, foam white)
- An expanding ring on the water surface sells the impact
- The whole effect lives ~1.5 seconds then cleans up
- Scale the splash with craft speed at impact (bigger speed = bigger splash)

---

## Wind & Thermal Visualization

Thermals and wind currents are invisible forces that need **visual representation** so the
player can navigate them.

### Thermal Columns

```javascript
function createThermalColumn(x, z, radius, height) {
  const group = new THREE.Group();
  group.position.set(x, 0, z);

  // Shimmering heat particles rising upward
  const particleCount = 30;
  const positions = new Float32Array(particleCount * 3);
  const sizes = new Float32Array(particleCount);

  for (let i = 0; i < particleCount; i++) {
    const angle = Math.random() * Math.PI * 2;
    const r = Math.random() * radius;
    positions[i * 3]     = Math.cos(angle) * r;
    positions[i * 3 + 1] = Math.random() * height;
    positions[i * 3 + 2] = Math.sin(angle) * r;
    sizes[i] = 2 + Math.random() * 3;
  }

  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  geo.setAttribute('size', new THREE.Float32BufferAttribute(sizes, 1));

  const mat = new THREE.PointsMaterial({
    color: 0xFFDD88,
    size: 3,
    transparent: true,
    opacity: 0.25,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    sizeAttenuation: true
  });

  const particles = new THREE.Points(geo, mat);
  group.add(particles);

  group.userData.update = (time) => {
    const posArr = geo.attributes.position.array;
    for (let i = 0; i < particleCount; i++) {
      posArr[i * 3 + 1] += 0.02; // rise
      if (posArr[i * 3 + 1] > height) {
        posArr[i * 3 + 1] = 0; // reset to bottom
      }
      // Gentle swirl
      const angle = time * 0.5 + i;
      posArr[i * 3]     += Math.sin(angle) * 0.005;
      posArr[i * 3 + 2] += Math.cos(angle) * 0.005;
    }
    geo.attributes.position.needsUpdate = true;

    // Pulse opacity
    mat.opacity = 0.15 + 0.1 * Math.sin(time * 2);
  };

  return group;
}
```

### Wind Trail Particles

```javascript
function createWindTrail(direction, bounds) {
  const particleCount = 50;
  const geo = new THREE.BufferGeometry();
  const positions = new Float32Array(particleCount * 3);

  for (let i = 0; i < particleCount; i++) {
    positions[i * 3]     = bounds.min.x + Math.random() * (bounds.max.x - bounds.min.x);
    positions[i * 3 + 1] = bounds.min.y + Math.random() * (bounds.max.y - bounds.min.y);
    positions[i * 3 + 2] = bounds.min.z + Math.random() * (bounds.max.z - bounds.min.z);
  }
  geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));

  const mat = new THREE.PointsMaterial({
    color: 0xFFFFFF,
    size: 1.5,
    transparent: true,
    opacity: 0.15,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    sizeAttenuation: true
  });

  const wind = new THREE.Points(geo, mat);

  wind.userData.update = (delta) => {
    const posArr = geo.attributes.position.array;
    const speed = direction.clone().multiplyScalar(delta * 8);
    for (let i = 0; i < particleCount; i++) {
      posArr[i * 3]     += speed.x;
      posArr[i * 3 + 1] += speed.y;
      posArr[i * 3 + 2] += speed.z;

      // Wrap around bounds
      if (posArr[i * 3]     > bounds.max.x) posArr[i * 3]     = bounds.min.x;
      if (posArr[i * 3 + 1] > bounds.max.y) posArr[i * 3 + 1] = bounds.min.y;
      if (posArr[i * 3 + 2] > bounds.max.z) posArr[i * 3 + 2] = bounds.min.z;
    }
    geo.attributes.position.needsUpdate = true;
  };

  return wind;
}
```

### Visualization Style Rules

- Thermals: warm golden (#FFDD88) upward-drifting particles, very subtle (opacity 0.15-0.25)
- Wind: white streak particles moving horizontally, even more subtle (opacity 0.1-0.2)
- Both use **additive blending** so they glow against sky
- Both use **depthWrite: false** so they never occlude game objects
- Player should *sense* these forces, not have them dominate the scene

---

## Visual Debug Checklist

Run through this checklist whenever the scene "doesn't look right":

```
Visual Quality Checklist
========================

Outlines
  [ ] Player craft has visible black outline (3-4%)
  [ ] Birds/hazards have outlines (2%)
  [ ] Pier/ramp has outlines (1.5%)
  [ ] Water has NO outline
  [ ] Sky has NO outline
  [ ] Particles have NO outline

Materials
  [ ] All solid objects use MeshToonMaterial
  [ ] Gradient map produces 3 visible shadow steps
  [ ] No smooth Phong/Lambert shading anywhere
  [ ] Water uses custom shader or MeshBasicMaterial
  [ ] Sky uses MeshBasicMaterial with vertex colors

Colors
  [ ] Craft is the most colorful/saturated object
  [ ] Sky gradient reads blue-to-light from top to bottom
  [ ] Water reads as distinct from sky (teals vs blues)
  [ ] Pier is warm brown, contrasts against cool water
  [ ] No object looks washed out or overly dark

Lighting
  [ ] Main directional light (sun) casts hard toon shadows
  [ ] Hemisphere light provides sky-blue top / ocean-teal bottom fill
  [ ] Ambient light is low enough to preserve shadow contrast
  [ ] No object is completely black (fill light prevents this)

Scale & Proportions
  [ ] Craft wingspan looks exaggerated (wider than fuselage length)
  [ ] Pier looks cartoonishly tall (~10m above water)
  [ ] Birds are small puffy shapes (~0.3m)
  [ ] All objects feel like they belong in the same cartoon world

Atmosphere
  [ ] Sky gradient sphere covers full background
  [ ] Clouds are puffy toon-shaded blobs
  [ ] Fog/distance fade is subtle (pushes toward sky color)
  [ ] Water has gentle animated wave motion
  [ ] Near objects are vivid, far objects fade slightly

Performance
  [ ] Outline meshes disabled beyond 30m distance
  [ ] Cloud count reasonable (< 15)
  [ ] Water plane resolution not excessive (40x40 is enough)
  [ ] Particle counts under 200 total
  [ ] No real-time reflections or expensive post-processing
```

---

## Test Prompts

Use these prompts to verify visual quality at each development stage:

1. **"Build the launch pier and ramp"**
   - Pier should be warm wood brown with toon shading and outlines
   - Ramp surface should be red/bold colored
   - Cartoonishly tall proportions, thick beams

2. **"Show a craft flying over water"**
   - Craft is boldly colored with visible outline against sky
   - Water below is teal with gentle animated waves
   - Sky gradient visible behind craft
   - Clear visual separation: craft / sky / water

3. **"Add bird hazards in the flight path"**
   - Birds are small, round, puffy shapes with outlines
   - Clearly identifiable as hazards against sky backdrop
   - Flapping wing animation readable even when small

4. **"Show the splash when craft hits water"**
   - Cartoon droplets burst outward and upward
   - Expanding ring on water surface
   - Colors match water palette (teals/foam white)
   - Effect is punchy and satisfying, not realistic

5. **"Visualize thermals and wind currents"**
   - Thermals: subtle golden particles rising in columns
   - Wind: faint white streaks moving horizontally
   - Neither effect dominates the scene
   - Both readable but atmospheric

6. **"Set up the flight scene with full environment"**
   - Sky gradient from deep blue to light at horizon
   - Water plane with animated waves below
   - Pier visible behind as launch point
   - Clouds scattered at various heights
   - Atmospheric perspective: distant objects fade toward sky color

7. **"Title screen with dramatic craft silhouette"**
   - Sunset lighting rig with warm orange tones
   - Craft silhouetted against bright sky
   - Water reflects sunset colors
   - Big bold title text
