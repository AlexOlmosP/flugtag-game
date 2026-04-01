# Toon Shading Reference

Complete technical reference for the Flugtag game's toon/cel shading pipeline. Every solid
game object -- craft, pier, birds, crowd, collectibles -- passes through this system.

---

## Table of Contents

- [Overview](#overview)
- [MeshToonMaterial Fundamentals](#meshtoonmaterial-fundamentals)
- [Gradient Map Textures](#gradient-map-textures)
- [Inverted Hull Outline System](#inverted-hull-outline-system)
- [Outline Thickness Reference](#outline-thickness-reference)
- [Helper Functions](#helper-functions)
  - [createToonMaterial()](#createtoonmaterial)
  - [addOutline()](#addoutline)
  - [toonifyGLB()](#toonifyglb)
- [Material Assignment Decision Tree](#material-assignment-decision-tree)
- [Performance Optimization](#performance-optimization)
- [Troubleshooting](#troubleshooting)

---

## Overview

The toon shading system has three layers:

```
  ┌─────────────────────────────────────────┐
  │ Layer 3: Post-Processing (FXAA)         │  ← smooths outline jaggies
  ├─────────────────────────────────────────┤
  │ Layer 2: Inverted Hull Outlines         │  ← black silhouette behind mesh
  ├─────────────────────────────────────────┤
  │ Layer 1: MeshToonMaterial + GradientMap │  ← flat color with shadow steps
  └─────────────────────────────────────────┘
```

All three layers must be present for the correct visual result. Missing the gradient map
produces smooth shading (wrong). Missing outlines makes objects blend into each other (wrong).
Missing FXAA makes outlines look jaggy at lower resolutions.

---

## MeshToonMaterial Fundamentals

`MeshToonMaterial` is a built-in Three.js material that quantizes lighting into discrete steps
using a gradient map texture. It responds to lights in the scene but renders in flat bands
instead of smooth gradients.

### Basic Usage

```javascript
// Simplest possible toon material
const material = new THREE.MeshToonMaterial({
  color: 0xFF4444  // base color
});
// Without a gradient map, this gives 2 steps (lit/shadow).
// We always want to provide a gradient map for 3+ steps.
```

### Key Properties

| Property     | Type           | Purpose                              | Flugtag Default     |
|-------------|----------------|--------------------------------------|---------------------|
| `color`     | Color/hex      | Base surface color                   | Varies per object   |
| `gradientMap` | Texture      | Controls shadow step count/placement | 3-step (see below)  |
| `emissive`  | Color/hex      | Self-illumination color              | 0x000000 (none)     |
| `emissiveIntensity` | float | Emissive strength                    | 0                   |
| `map`       | Texture        | Albedo texture (rarely used)         | null (flat color)   |
| `side`      | Side enum      | Which faces to render                | THREE.FrontSide     |
| `transparent` | boolean      | Enable transparency                  | false                |
| `opacity`   | float          | Transparency amount                  | 1.0                 |

### What NOT to Use

```javascript
// WRONG - these materials break toon aesthetic
new THREE.MeshPhongMaterial(...)      // smooth specular shading
new THREE.MeshLambertMaterial(...)    // smooth diffuse shading
new THREE.MeshStandardMaterial(...)   // PBR (too realistic) -- except UI platforms
new THREE.MeshPhysicalMaterial(...)   // advanced PBR (way too realistic)
```

---

## Gradient Map Textures

The gradient map is a **1D texture** that maps light intensity (0=shadow, 1=fully lit) to
discrete brightness steps. Three.js samples this texture based on the dot product of the
surface normal and light direction.

### How It Works

```
  Light intensity:  0.0 ──────────────────────────── 1.0
                     │                                 │
  Gradient map:     [dark ][dark ][mid  ][mid  ][light][light]
                     │                                 │
  Result:           Hard steps instead of smooth ramp
```

### 3-Step Gradient Map (Default)

The standard for all Flugtag game objects. Produces **shadow / mid / lit** bands.

```javascript
function createGradientMap(steps = 3) {
  let colors;

  if (steps === 3) {
    // Shadow at 40%, mid at 70%, lit at 100%
    colors = new Uint8Array([102, 178, 255]);  // 0.4, 0.7, 1.0 as bytes
  } else if (steps === 5) {
    // Finer control for hero objects (craft)
    colors = new Uint8Array([77, 128, 178, 217, 255]);
  } else {
    // Generate evenly spaced steps
    colors = new Uint8Array(steps);
    for (let i = 0; i < steps; i++) {
      colors[i] = Math.round(((i + 1) / steps) * 255);
    }
  }

  const gradientMap = new THREE.DataTexture(
    colors,
    colors.length,    // width = number of steps
    1,                // height = 1 (it's a 1D lookup)
    THREE.LuminanceFormat
  );

  // CRITICAL: These filter settings create the hard steps
  gradientMap.minFilter = THREE.NearestFilter;
  gradientMap.magFilter = THREE.NearestFilter;
  gradientMap.needsUpdate = true;

  return gradientMap;
}
```

### Filter Settings Are Critical

```
  NearestFilter (CORRECT):              LinearFilter (WRONG):
  ┌────┬────┬────┐                      ┌─────────────────┐
  │dark│mid │lit │  ← hard steps        │ smooth gradient  │  ← defeats purpose
  └────┴────┴────┘                      └─────────────────┘
```

**Always** set both `minFilter` and `magFilter` to `THREE.NearestFilter`. If you see smooth
shading on a MeshToonMaterial, the filters are probably wrong.

### 5-Step Gradient Map (Hero Objects)

Use on the player craft for slightly more nuanced shading while remaining toon-like:

```javascript
const heroGradient = createGradientMap(5);
// Produces: deep shadow / shadow / mid / light / highlight
// Still clearly stepped, but slightly more refined
```

### When to Use Which

| Step Count | Use Case                                    |
|-----------|---------------------------------------------|
| 2         | Very distant/small objects (simplest)       |
| 3         | Default for all game objects                |
| 5         | Player craft, pier close-up, featured items |

---

## Inverted Hull Outline System

The inverted hull method creates outlines by rendering a **slightly larger copy** of the mesh
behind the original, using only back faces, colored solid black.

### How It Works

```
  Step 1: Original mesh        Step 2: Scaled-up back-face copy

    ┌──────┐                     ██████████
    │      │                     █┌──────┐█
    │ mesh │                     █│ mesh │█  ← black border visible
    │      │                     █│      │█     around edges
    └──────┘                     █└──────┘█
                                 ██████████
```

### Complete Outline Implementation

```javascript
/**
 * Creates an outline mesh for the given source mesh.
 * The outline is a scaled-up copy that only renders back faces in solid black.
 *
 * @param {THREE.Mesh} mesh - Source mesh to outline
 * @param {number} thickness - Outline size as fraction of mesh size (0.03 = 3%)
 * @returns {THREE.Mesh} The outline mesh (add to same parent as source)
 */
function addOutline(mesh, thickness = 0.03) {
  // Clone geometry to avoid modifying original
  const outlineGeo = mesh.geometry.clone();

  // Method: Scale normals outward for uniform thickness
  // This works better than uniform scaling for non-uniform shapes
  const positionAttr = outlineGeo.attributes.position;
  const normalAttr = outlineGeo.attributes.normal;

  if (normalAttr) {
    for (let i = 0; i < positionAttr.count; i++) {
      const nx = normalAttr.getX(i);
      const ny = normalAttr.getY(i);
      const nz = normalAttr.getZ(i);

      positionAttr.setX(i, positionAttr.getX(i) + nx * thickness);
      positionAttr.setY(i, positionAttr.getY(i) + ny * thickness);
      positionAttr.setZ(i, positionAttr.getZ(i) + nz * thickness);
    }
    positionAttr.needsUpdate = true;
  }

  const outlineMat = new THREE.MeshBasicMaterial({
    color: 0x000000,
    side: THREE.BackSide   // only render back faces = outline effect
  });

  const outlineMesh = new THREE.Mesh(outlineGeo, outlineMat);

  // Copy transform from source
  outlineMesh.position.copy(mesh.position);
  outlineMesh.rotation.copy(mesh.rotation);
  outlineMesh.scale.copy(mesh.scale);

  // Tag for management
  outlineMesh.userData.isOutline = true;
  outlineMesh.userData.sourceId = mesh.uuid;

  // Render outlines before the main mesh
  outlineMesh.renderOrder = mesh.renderOrder - 1;

  // Add as sibling (not child) so transforms work correctly
  if (mesh.parent) {
    mesh.parent.add(outlineMesh);
  }

  // Store reference on source for cleanup
  mesh.userData.outlineMesh = outlineMesh;

  return outlineMesh;
}
```

### Alternative: Simple Scale Method

For perfectly convex shapes (spheres, capsules), uniform scaling is simpler:

```javascript
function addOutlineSimple(mesh, thickness = 0.03) {
  const outlineMesh = mesh.clone();
  outlineMesh.material = new THREE.MeshBasicMaterial({
    color: 0x000000,
    side: THREE.BackSide
  });

  const scaleFactor = 1 + thickness * 2;
  outlineMesh.scale.multiplyScalar(scaleFactor);

  outlineMesh.userData.isOutline = true;
  mesh.parent?.add(outlineMesh);
  mesh.userData.outlineMesh = outlineMesh;

  return outlineMesh;
}
```

> **Use the normal-push method** (first version) for complex shapes like craft, pier, birds.
> The simple scale method is fine for basic shapes like collectible spheres.

---

## Outline Thickness Reference

```
┌────────────────────────┬──────────┬──────────────────────────────────┐
│ Object Category        │ Thickness│ Reasoning                        │
├────────────────────────┼──────────┼──────────────────────────────────┤
│ Player craft (body)    │ 0.04     │ Hero object, needs to POP        │
│ Craft wings/propeller  │ 0.03     │ Important but secondary          │
│ Craft decorations      │ 0.02     │ Detail level, thinner            │
│ Birds / hazards        │ 0.02     │ Must read as threat at distance  │
│ Collectible rings      │ 0.02     │ Important gameplay element       │
│ Pier structure         │ 0.015    │ Large scenery, moderate outline  │
│ Ramp surface           │ 0.015    │ Part of pier, same weight        │
│ Crowd figures          │ 0.015    │ Background, don't dominate       │
│ Distant scenery        │ 0.01     │ Minimal, just enough definition  │
│ Clouds                 │ NONE     │ Soft atmospheric, no hard edges  │
│ Water surface          │ NONE     │ Infinite plane, outline meaningless│
│ Sky dome               │ NONE     │ Background, never outlined       │
│ Particles/effects      │ NONE     │ Ephemeral, would look wrong      │
│ UI/HUD elements        │ CSS only │ Use CSS border, not 3D outline   │
└────────────────────────┴──────────┴──────────────────────────────────┘
```

---

## Helper Functions

### createToonMaterial()

One-stop function for creating a properly configured toon material:

```javascript
/**
 * Creates a MeshToonMaterial with correct gradient map and settings.
 *
 * @param {number|string} color - Hex color value
 * @param {Object} options - Additional options
 * @param {number} options.steps - Gradient steps (default 3)
 * @param {number} options.emissive - Emissive color (default 0x000000)
 * @param {number} options.emissiveIntensity - Emissive strength (default 0)
 * @param {boolean} options.transparent - Enable transparency (default false)
 * @param {number} options.opacity - Opacity value (default 1.0)
 * @param {number} options.side - Face culling (default FrontSide)
 * @returns {THREE.MeshToonMaterial}
 */
function createToonMaterial(color, options = {}) {
  const {
    steps = 3,
    emissive = 0x000000,
    emissiveIntensity = 0,
    transparent = false,
    opacity = 1.0,
    side = THREE.FrontSide
  } = options;

  const gradientMap = createGradientMap(steps);

  return new THREE.MeshToonMaterial({
    color: color,
    gradientMap: gradientMap,
    emissive: emissive,
    emissiveIntensity: emissiveIntensity,
    transparent: transparent,
    opacity: opacity,
    side: side
  });
}

// Usage examples:
const craftBody   = createToonMaterial(0xFF4444, { steps: 5 });
const craftWing   = createToonMaterial(0x4488FF, { steps: 3 });
const pierWood    = createToonMaterial(0xB88855);
const birdBody    = createToonMaterial(0x333333, { steps: 3 });
const collectible = createToonMaterial(0xFFDD44, {
  emissive: 0xFFDD44,
  emissiveIntensity: 0.3
});
```

### addOutline()

See the [Inverted Hull Outline System](#inverted-hull-outline-system) section above for the
complete implementation.

### toonifyGLB()

Pipeline for converting loaded GLB models to match the toon art style:

```javascript
/**
 * Converts all materials in a loaded GLTF model to toon-shaded materials
 * and adds outlines to all meshes. Call this immediately after loading any GLB.
 *
 * @param {Object} gltf - The loaded GLTF object from GLTFLoader
 * @param {Object} options - Configuration options
 * @param {number} options.outlineThickness - Default outline thickness (0.03)
 * @param {number} options.gradientSteps - Default gradient steps (3)
 * @param {boolean} options.addOutlines - Whether to add outlines (true)
 * @param {Object} options.outlineOverrides - Map of mesh name -> thickness override
 * @returns {THREE.Group} The toonified model
 */
function toonifyGLB(gltf, options = {}) {
  const {
    outlineThickness = 0.03,
    gradientSteps = 3,
    addOutlines: shouldAddOutlines = true,
    outlineOverrides = {}
  } = options;

  const model = gltf.scene;

  // Shared gradient map for performance
  const sharedGradient = createGradientMap(gradientSteps);

  model.traverse((child) => {
    if (!child.isMesh) return;

    // --- Step 1: Convert material ---
    const oldMat = child.material;

    // Determine if this mesh should skip toon conversion
    const name = (child.name || '').toLowerCase();
    const skipToon = name.includes('particle') ||
                     name.includes('glow') ||
                     name.includes('trail') ||
                     name.includes('water') ||
                     name.includes('sky');

    if (skipToon) {
      // Keep as MeshBasicMaterial for effects
      child.material = new THREE.MeshBasicMaterial({
        color: oldMat.color || 0xFFFFFF,
        transparent: oldMat.transparent || false,
        opacity: oldMat.opacity || 1.0
      });
      return; // No outline for skipped meshes
    }

    // Extract color from original material
    const baseColor = oldMat.color ? oldMat.color.clone() : new THREE.Color(0xCCCCCC);

    // Create toon replacement
    const toonMat = new THREE.MeshToonMaterial({
      color: baseColor,
      gradientMap: sharedGradient,
      side: THREE.FrontSide
    });

    // Preserve texture map if present (rare in our style, but possible)
    if (oldMat.map) {
      toonMat.map = oldMat.map;
    }

    child.material = toonMat;

    // --- Step 2: Enable shadow receiving ---
    child.castShadow = true;
    child.receiveShadow = true;

    // --- Step 3: Add outline ---
    if (shouldAddOutlines) {
      const thickness = outlineOverrides[child.name] || outlineThickness;
      addOutline(child, thickness);
    }
  });

  return model;
}

// Usage with GLTFLoader:
const loader = new THREE.GLTFLoader();
loader.load('craft.glb', (gltf) => {
  const craft = toonifyGLB(gltf, {
    outlineThickness: 0.04,
    gradientSteps: 5,
    outlineOverrides: {
      'wing_left': 0.03,
      'wing_right': 0.03,
      'propeller': 0.02,
      'decoration_flag': 0.02
    }
  });
  scene.add(craft);
});
```

### Toonification Pipeline Diagram

```
  GLB File Loaded
       │
       ▼
  ┌──────────────────────┐
  │ Traverse all meshes  │
  └──────────┬───────────┘
             │
       For each mesh:
             │
       ┌─────┴─────┐
       │ Is it an   │
       │ effect?    │──── YES ──→ Convert to MeshBasicMaterial, skip outline
       │ (particle, │
       │  glow, etc)│
       └─────┬──────┘
             │ NO
             ▼
  ┌──────────────────────┐
  │ Extract base color   │
  │ from original mat    │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ Create MeshToonMat   │
  │ + shared gradient map│
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ Enable shadows       │
  │ (cast + receive)     │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ Add inverted hull    │
  │ outline              │
  │ (check overrides)    │
  └──────────┬───────────┘
             │
             ▼
       Done! Add to scene.
```

---

## Material Assignment Decision Tree

When building objects procedurally or deciding how to shade something:

```
  "What material should this object use?"
       │
       ├─── Is it SKY?
       │     └─→ MeshBasicMaterial (vertex colors, no lighting)
       │
       ├─── Is it WATER?
       │     └─→ ShaderMaterial (custom toon water, see SKILL.md)
       │
       ├─── Is it a CLOUD?
       │     └─→ MeshToonMaterial, 3-step, NO outline
       │
       ├─── Is it a PARTICLE / TRAIL / GLOW?
       │     └─→ MeshBasicMaterial or PointsMaterial
       │         + transparent + additive blending
       │         + depthWrite: false
       │         + NO outline
       │
       ├─── Is it a UI PLATFORM (menus floating in 3D)?
       │     └─→ MeshStandardMaterial (subtle reflections OK)
       │         + NO outline (use CSS-styled HTML overlay instead)
       │
       ├─── Is it the PLAYER CRAFT?
       │     └─→ MeshToonMaterial, 5-step gradient
       │         + outline at 0.04 (body) / 0.03 (wings) / 0.02 (details)
       │
       ├─── Is it a BIRD / HAZARD?
       │     └─→ MeshToonMaterial, 3-step
       │         + outline at 0.02
       │
       ├─── Is it PIER / RAMP / SCENERY?
       │     └─→ MeshToonMaterial, 3-step
       │         + outline at 0.015
       │
       └─── Is it a COLLECTIBLE?
             └─→ MeshToonMaterial, 3-step
                 + emissive glow (emissiveIntensity: 0.3)
                 + outline at 0.02
                 + additive glow sprite behind (fake bloom)
```

---

## Performance Optimization

### Distance-Based Outline Culling

Outlines are invisible at distance. Disable them to save draw calls:

```javascript
/**
 * Updates outline visibility based on camera distance.
 * Call once per frame.
 *
 * @param {THREE.Camera} camera
 * @param {THREE.Scene} scene
 * @param {number} maxDistance - Distance to disable outlines (default 30)
 */
function updateOutlineVisibility(camera, scene, maxDistance = 30) {
  scene.traverse((child) => {
    if (child.userData.isOutline) {
      const distance = camera.position.distanceTo(child.position);
      child.visible = distance < maxDistance;
    }
  });
}
```

### LOD System for Toon Objects

Switch gradient map complexity and outline presence based on distance:

```javascript
function updateToonLOD(mesh, cameraDistance) {
  if (cameraDistance < 10) {
    // Close: full detail
    mesh.material.gradientMap = createGradientMap(5);
    if (mesh.userData.outlineMesh) mesh.userData.outlineMesh.visible = true;
  } else if (cameraDistance < 30) {
    // Medium: standard detail
    mesh.material.gradientMap = createGradientMap(3);
    if (mesh.userData.outlineMesh) mesh.userData.outlineMesh.visible = true;
  } else {
    // Far: minimal detail
    mesh.material.gradientMap = createGradientMap(2);
    if (mesh.userData.outlineMesh) mesh.userData.outlineMesh.visible = false;
  }
}
```

### Performance Budget

```
┌──────────────────────┬────────────┬────────────────────────┐
│ System               │ Draw Calls │ Notes                  │
├──────────────────────┼────────────┼────────────────────────┤
│ Player craft + outlines │ 6-10    │ Multiple parts         │
│ Pier + outlines      │ 4-8       │ Few large meshes       │
│ Birds (up to 8)      │ 16        │ Mesh + outline each    │
│ Collectibles (up to 6)│ 12-18   │ Mesh + outline + glow  │
│ Clouds (up to 12)    │ 12-36    │ Multi-sphere, no outline│
│ Water                │ 1         │ Single shader mesh     │
│ Sky                  │ 1         │ Single sphere          │
│ Particles            │ 2-4       │ Wind + thermals        │
├──────────────────────┼────────────┼────────────────────────┤
│ TOTAL TARGET         │ < 80      │ Comfortable for mobile │
└──────────────────────┴────────────┴────────────────────────┘
```

---

## Troubleshooting

### Objects look smooth instead of toon-shaded

**Cause:** Gradient map filter is wrong or gradient map is missing.

```javascript
// CHECK: Is the gradient map assigned?
console.log(mesh.material.gradientMap); // Should not be null

// CHECK: Are filters set to Nearest?
console.log(mesh.material.gradientMap.minFilter === THREE.NearestFilter); // true
console.log(mesh.material.gradientMap.magFilter === THREE.NearestFilter); // true
```

### Outlines are not visible

**Cause:** Outline mesh may be inside the original mesh, or back-face culling is inverted.

```javascript
// CHECK: Is outline material using BackSide?
console.log(outlineMesh.material.side === THREE.BackSide); // true

// CHECK: Is thickness too small?
// Try temporarily setting thickness to 0.1 to see if it appears

// CHECK: Is outline mesh in the scene?
console.log(outlineMesh.parent); // Should not be null
```

### Outlines flicker or z-fight

**Cause:** Outline and source mesh are too close in depth.

```javascript
// FIX: Set renderOrder to ensure outline draws first
outlineMesh.renderOrder = mesh.renderOrder - 1;

// FIX: Or use polygonOffset on the outline material
outlineMesh.material.polygonOffset = true;
outlineMesh.material.polygonOffsetFactor = 1;
outlineMesh.material.polygonOffsetUnits = 1;
```

### Toon materials are completely black

**Cause:** No lights in the scene. MeshToonMaterial requires at least one light source.

```javascript
// MINIMUM lighting for toon materials:
scene.add(new THREE.AmbientLight(0x404040, 0.5));
scene.add(new THREE.DirectionalLight(0xFFFFFF, 1.0));
```

### GLB model looks wrong after toonification

**Cause:** Original model may have custom shaders, multiple materials per mesh, or unusual
vertex attributes.

```javascript
// DEBUG: Log what the original model contains
gltf.scene.traverse((child) => {
  if (child.isMesh) {
    console.log(`Mesh: ${child.name}`);
    console.log(`  Material: ${child.material.type}`);
    console.log(`  Color: #${child.material.color?.getHexString()}`);
    console.log(`  Has map: ${!!child.material.map}`);
    console.log(`  Has normals: ${!!child.geometry.attributes.normal}`);
  }
});

// FIX: Ensure normals exist (required for toon shading)
if (!child.geometry.attributes.normal) {
  child.geometry.computeVertexNormals();
}
```

### Outline thickness looks inconsistent across objects

**Cause:** The normal-push method produces thickness relative to the mesh's local space. If
meshes have non-uniform scaling, outlines will look uneven.

```javascript
// FIX: Apply world matrix to geometry before adding outline
child.updateWorldMatrix(true, false);
child.geometry.applyMatrix4(child.matrixWorld);
child.position.set(0, 0, 0);
child.rotation.set(0, 0, 0);
child.scale.set(1, 1, 1);
// Then add outline
addOutline(child, 0.03);
```

### Performance is poor with many outlined objects

```javascript
// Optimization 1: Share outline material across all outlines
const SHARED_OUTLINE_MAT = new THREE.MeshBasicMaterial({
  color: 0x000000,
  side: THREE.BackSide
});

// Optimization 2: Merge outline geometries for static objects
// (pier, scenery that never moves)
const mergedGeo = THREE.BufferGeometryUtils.mergeBufferGeometries(
  staticOutlineGeometries
);
const mergedOutline = new THREE.Mesh(mergedGeo, SHARED_OUTLINE_MAT);
scene.add(mergedOutline);

// Optimization 3: Distance culling (see Performance section above)
```
