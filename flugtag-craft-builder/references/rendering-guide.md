# Rendering Guide

Technical rendering reference for the Red Bull Flugtag craft builder workshop scene.

---

## Table of Contents

1. [Three.js Scene Setup](#threejs-scene-setup)
2. [Workshop Lighting Rig](#workshop-lighting-rig)
3. [AssetManager Class](#assetmanager-class)
4. [GLB Loading Pipeline](#glb-loading-pipeline)
5. [Craft Assembly Pipeline](#craft-assembly-pipeline)
6. [Wing Mount Positions](#wing-mount-positions)
7. [Decoration Anchor Points](#decoration-anchor-points)
8. [Placeholder Geometry Builders](#placeholder-geometry-builders)
9. [Turntable Animation](#turntable-animation)
10. [Performance Budget](#performance-budget)

---

## Three.js Scene Setup

The workshop uses a dedicated scene, renderer, camera, and controls separate from
the flight scene. Everything runs in a single `<canvas>` element.

```javascript
// --- Workshop Scene Setup ---

const workshopScene = new THREE.Scene();
workshopScene.background = new THREE.Color(0x2a2a3e); // Dark blue-grey workshop

// Renderer (shared with flight scene)
const renderer = new THREE.WebGLRenderer({
  canvas: document.getElementById('game-canvas'),
  antialias: true,
  alpha: false
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputEncoding = THREE.sRGBEncoding;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// Camera
const workshopCamera = new THREE.PerspectiveCamera(
  45,                                          // FOV
  window.innerWidth / window.innerHeight,      // Aspect
  0.1,                                         // Near
  100                                          // Far
);
workshopCamera.position.set(2.5, 2.0, 3.0);
workshopCamera.lookAt(0, 0.5, 0);

// Orbit Controls (from Three.js examples)
// Only loaded in workshop mode
const orbitControls = new THREE.OrbitControls(workshopCamera, renderer.domElement);
orbitControls.target.set(0, 0.5, 0);
orbitControls.enableDamping = true;
orbitControls.dampingFactor = 0.08;
orbitControls.enablePan = false;
orbitControls.minDistance = 1.5;
orbitControls.maxDistance = 6.0;
orbitControls.minPolarAngle = 0.3;     // Don't go below floor
orbitControls.maxPolarAngle = Math.PI / 2.1; // Don't go under
orbitControls.autoRotate = true;
orbitControls.autoRotateSpeed = 1.5;   // Gentle turntable

// Workshop floor
const floorGeo = new THREE.PlaneGeometry(10, 10);
const floorMat = new THREE.MeshToonMaterial({ color: 0x3a3a4e });
const workshopFloor = new THREE.Mesh(floorGeo, floorMat);
workshopFloor.rotation.x = -Math.PI / 2;
workshopFloor.receiveShadow = true;
workshopScene.add(workshopFloor);

// Checkerboard overlay (optional visual flair)
function createCheckerTexture(size) {
  const canvas = document.createElement('canvas');
  canvas.width = canvas.height = size;
  const ctx = canvas.getContext('2d');
  const cellSize = size / 8;
  for (let r = 0; r < 8; r++) {
    for (let c = 0; c < 8; c++) {
      ctx.fillStyle = (r + c) % 2 === 0 ? '#3a3a4e' : '#333346';
      ctx.fillRect(c * cellSize, r * cellSize, cellSize, cellSize);
    }
  }
  const tex = new THREE.CanvasTexture(canvas);
  tex.wrapS = tex.wrapT = THREE.RepeatWrapping;
  tex.repeat.set(4, 4);
  return tex;
}
workshopFloor.material.map = createCheckerTexture(256);

// Handle resize
window.addEventListener('resize', () => {
  workshopCamera.aspect = window.innerWidth / window.innerHeight;
  workshopCamera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

---

## Workshop Lighting Rig

Three-light setup optimized for showing off chunky toon-shaded models.

```javascript
// --- Workshop Lighting ---

// 1. Ambient fill -- soft warm base
const ambientLight = new THREE.AmbientLight(0xFFEEDD, 0.4);
workshopScene.add(ambientLight);

// 2. Key light -- warm spotlight from upper-right-front
const keyLight = new THREE.SpotLight(0xFFDDAA, 1.2);
keyLight.position.set(3, 5, 3);
keyLight.angle = Math.PI / 5;
keyLight.penumbra = 0.4;
keyLight.castShadow = true;
keyLight.shadow.mapSize.width = 1024;
keyLight.shadow.mapSize.height = 1024;
keyLight.shadow.camera.near = 1;
keyLight.shadow.camera.far = 15;
keyLight.target.position.set(0, 0.5, 0);
workshopScene.add(keyLight);
workshopScene.add(keyLight.target);

// 3. Fill light -- cool blue from upper-left
const fillLight = new THREE.DirectionalLight(0x88AAFF, 0.5);
fillLight.position.set(-2, 3, -1);
workshopScene.add(fillLight);

// 4. Rim light -- white from behind to separate model from background
const rimLight = new THREE.DirectionalLight(0xFFFFFF, 0.6);
rimLight.position.set(0, 2, -4);
workshopScene.add(rimLight);

// Optional: point light under the craft for dramatic uplighting
const underLight = new THREE.PointLight(0x4466FF, 0.3, 3);
underLight.position.set(0, -0.2, 0);
workshopScene.add(underLight);
```

### Lighting reference diagram

```
        Key (warm)
        *
       / \
      /   \     Rim (white)
     /     \       *
    /  Craft \    /
   /  [=====] \ /
  /     |      X
 /      |     / \
*       |    /   
Fill    |   /
(blue)  | Floor
```

---

## AssetManager Class

Centralized asset loading with caching and graceful fallback to procedural geometry.

```javascript
class AssetManager {
  constructor(basePath = 'assets/') {
    this.basePath = basePath;
    this.cache = new Map();       // key -> THREE.Object3D (cloned on access)
    this.loading = new Map();     // key -> Promise
    this.loader = new THREE.GLTFLoader();
    this.failedAssets = new Set(); // Track permanently failed loads
  }

  /**
   * Get the file path for a given category and type.
   * @param {string} category - 'fuselage', 'wings', 'propulsion', etc.
   * @param {string} type - Part type ID
   * @returns {string} Full path to GLB file
   */
  getPath(category, type) {
    // Decoration categories are prefixed with slot
    // e.g., 'deco-side', 'deco-tip', 'deco-nose', 'deco-tail'
    return `${this.basePath}flugtag-${category}-${type}.glb`;
  }

  /**
   * Load a GLB model. Returns cached clone if available.
   * Falls back to placeholder geometry on failure.
   * @param {string} category
   * @param {string} type
   * @returns {Promise<THREE.Object3D>}
   */
  async load(category, type) {
    const key = `${category}:${type}`;

    // Return cached clone immediately
    if (this.cache.has(key)) {
      return this.cache.get(key).clone();
    }

    // If already known to fail, return placeholder immediately
    if (this.failedAssets.has(key)) {
      return this._getPlaceholder(category, type);
    }

    // If currently loading, wait for it
    if (this.loading.has(key)) {
      await this.loading.get(key);
      return this.cache.has(key)
        ? this.cache.get(key).clone()
        : this._getPlaceholder(category, type);
    }

    // Start new load
    const loadPromise = this._loadGLB(category, type, key);
    this.loading.set(key, loadPromise);

    try {
      await loadPromise;
      return this.cache.get(key).clone();
    } catch (e) {
      console.warn(`AssetManager: Failed to load ${key}, using placeholder`, e);
      this.failedAssets.add(key);
      return this._getPlaceholder(category, type);
    } finally {
      this.loading.delete(key);
    }
  }

  /**
   * Internal: load and process a GLB file.
   */
  async _loadGLB(category, type, key) {
    const path = this.getPath(category, type);

    return new Promise((resolve, reject) => {
      this.loader.load(
        path,
        (gltf) => {
          const model = gltf.scene;
          // Apply toon materials to all meshes
          this._toonifyModel(model);
          // Enable shadows
          model.traverse((child) => {
            if (child.isMesh) {
              child.castShadow = true;
              child.receiveShadow = true;
            }
          });
          this.cache.set(key, model);
          resolve(model);
        },
        undefined, // onProgress
        (error) => reject(error)
      );
    });
  }

  /**
   * Convert all materials in a model to MeshToonMaterial.
   * Preserves base color and texture maps.
   */
  _toonifyModel(model) {
    model.traverse((child) => {
      if (child.isMesh && child.material) {
        const oldMat = child.material;
        const toonMat = new THREE.MeshToonMaterial({
          color: oldMat.color ? oldMat.color.clone() : new THREE.Color(0xFFFFFF),
          map: oldMat.map || null,
          side: THREE.DoubleSide
        });
        child.material = toonMat;
      }
    });
  }

  /**
   * Get placeholder geometry for a given part.
   */
  _getPlaceholder(category, type) {
    switch (category) {
      case 'fuselage':   return createPlaceholderFuselage(type);
      case 'wings':      return createPlaceholderWings(type);
      case 'propulsion': return createPlaceholderPropulsion(type);
      case 'helmet':     return createPlaceholderHelmet(type);
      case 'pilot':      return createPlaceholderPilot();
      default:
        // Decorations: deco-side, deco-tip, deco-nose, deco-tail
        if (category.startsWith('deco-')) {
          const slot = category.replace('deco-', '');
          return createPlaceholderDecoration(type, slot);
        }
        // Unknown category -- return a debug cube
        return new THREE.Mesh(
          new THREE.BoxGeometry(0.2, 0.2, 0.2),
          new THREE.MeshToonMaterial({ color: 0xFF00FF })
        );
    }
  }

  /**
   * Preload a set of assets in parallel.
   * @param {Array<{category: string, type: string}>} manifest
   */
  async preload(manifest) {
    const promises = manifest.map(({ category, type }) =>
      this.load(category, type).catch(() => {}) // Swallow errors, fallback handles it
    );
    await Promise.all(promises);
  }

  /**
   * Clear all cached assets and free GPU memory.
   */
  dispose() {
    for (const [key, model] of this.cache) {
      model.traverse((child) => {
        if (child.isMesh) {
          child.geometry?.dispose();
          if (child.material) {
            if (Array.isArray(child.material)) {
              child.material.forEach(m => { m.map?.dispose(); m.dispose(); });
            } else {
              child.material.map?.dispose();
              child.material.dispose();
            }
          }
        }
      });
    }
    this.cache.clear();
    this.loading.clear();
    this.failedAssets.clear();
  }
}

// Singleton instance
const assetManager = new AssetManager('assets/');
```

---

## GLB Loading Pipeline

The pipeline from raw file to rendered toon-shaded craft part:

```
1. LOAD         GLTFLoader.load(path) -> gltf.scene
                 |
2. TOONIFY      Traverse all meshes, swap materials to MeshToonMaterial
                 |
3. ATTACH       Add to CraftGroup at correct position/rotation
                 |
4. TINT         Apply player's color choice via tintModel()
```

### tintModel function

```javascript
/**
 * Tint all meshes in a model to the given hex color.
 * Clones materials so instances don't share tint state.
 * @param {THREE.Object3D} model
 * @param {string} hexColor - e.g. "#FF4444"
 */
function tintModel(model, hexColor) {
  const color = new THREE.Color(hexColor);
  model.traverse((child) => {
    if (child.isMesh && child.material) {
      // Only tint white-ish base meshes (avoid tinting accent parts)
      const baseColor = child.material.color;
      if (baseColor && baseColor.r > 0.9 && baseColor.g > 0.9 && baseColor.b > 0.9) {
        child.material = child.material.clone();
        child.material.color.copy(color);
      }
    }
  });
}
```

> **Important**: The tint function only affects meshes with a near-white base color
> (r,g,b > 0.9). This preserves accent colors on non-tintable sub-meshes (e.g., the
> bathtub's gold feet, the aviator helmet's goggles).

---

## Craft Assembly Pipeline

The `assembleCraft` function builds the complete 3D scene graph from a craft data model.

```javascript
/**
 * Assemble a complete craft from the craft data model.
 * Loads all parts, positions them, applies colors, returns a THREE.Group.
 *
 * @param {Object} craft - The craft data model (see SKILL.md)
 * @param {AssetManager} assets - The asset manager instance
 * @returns {Promise<THREE.Group>} The assembled craft group
 */
async function assembleCraft(craft, assets) {
  const craftGroup = new THREE.Group();
  craftGroup.name = 'CraftGroup';

  // --- 1. Fuselage ---
  const fuselage = await assets.load('fuselage', craft.fuselage.type);
  fuselage.name = 'Fuselage';
  tintModel(fuselage, craft.fuselage.color);
  craftGroup.add(fuselage);

  // --- 2. Wings ---
  const wings = await assets.load('wings', craft.wings.type);
  wings.name = 'Wings';
  const wingMount = WING_MOUNT_POSITIONS[craft.fuselage.type] || { x: 0, y: 0.5, z: 0 };
  wings.position.set(wingMount.x, wingMount.y, wingMount.z);
  tintModel(wings, craft.wings.color);
  craftGroup.add(wings);

  // --- 3. Propulsion ---
  if (craft.propulsion.type !== 'none' && craft.propulsion.enabled) {
    const prop = await assets.load('propulsion', craft.propulsion.type);
    prop.name = 'Propulsion';
    const propMount = PROPULSION_MOUNT[craft.fuselage.type] || { x: -0.7, y: 0.3, z: 0 };
    prop.position.set(propMount.x, propMount.y, propMount.z);
    craftGroup.add(prop);
  }

  // --- 4. Decorations ---
  for (const deco of craft.decorations) {
    if (!deco.enabled) continue;
    const slotCategory = 'deco-' + deco.slot.replace('fuselage_', '').replace('wing_', '');
    const decoModel = await assets.load(slotCategory, deco.type);
    decoModel.name = 'Deco_' + deco.slot;

    const anchor = DECORATION_ANCHORS[craft.fuselage.type]?.[deco.slot]
      || DECORATION_ANCHORS._default[deco.slot];
    decoModel.position.set(anchor.x, anchor.y, anchor.z);
    if (anchor.rotation) {
      decoModel.rotation.set(
        anchor.rotation.x || 0,
        anchor.rotation.y || 0,
        anchor.rotation.z || 0
      );
    }
    craftGroup.add(decoModel);
  }

  // --- 5. Pilot ---
  const pilot = await assets.load('pilot', 'body');
  pilot.name = 'PilotBody';
  const seatOffset = PILOT_SEAT_OFFSETS[craft.fuselage.type] || { x: 0, y: 0.45, z: 0 };
  pilot.position.set(seatOffset.x, seatOffset.y, seatOffset.z);
  craftGroup.add(pilot);

  // --- 6. Helmet ---
  const helmet = await assets.load('helmet', craft.pilot.helmet);
  helmet.name = 'Helmet';
  // Position relative to pilot head
  helmet.position.set(
    seatOffset.x,
    seatOffset.y + 0.7, // Head is 0.7m above pilot body origin
    seatOffset.z
  );
  tintModel(helmet, craft.pilot.helmetColor);
  craftGroup.add(helmet);

  return craftGroup;
}
```

### Updating a single part (swap without full rebuild)

```javascript
/**
 * Swap a single part in an existing craft group.
 * More efficient than full reassembly for real-time editor updates.
 *
 * @param {THREE.Group} craftGroup - Existing assembled craft
 * @param {string} partName - Scene node name ('Fuselage', 'Wings', 'Propulsion', etc.)
 * @param {THREE.Object3D} newModel - New model to replace it with
 */
function swapPart(craftGroup, partName, newModel) {
  const existing = craftGroup.getObjectByName(partName);
  if (existing) {
    // Preserve position/rotation from old part
    newModel.position.copy(existing.position);
    newModel.rotation.copy(existing.rotation);
    newModel.scale.copy(existing.scale);
    craftGroup.remove(existing);
    // Dispose old geometry/materials
    existing.traverse((child) => {
      if (child.isMesh) {
        child.geometry?.dispose();
        child.material?.dispose();
      }
    });
  }
  newModel.name = partName;
  craftGroup.add(newModel);
}
```

---

## Wing Mount Positions

Wing mount Y-position varies per fuselage to look correct. The wings group is
centered at origin, so we offset its position to align with the fuselage.

```javascript
const WING_MOUNT_POSITIONS = {
  bathtub:       { x: 0, y: 0.50, z: 0 },
  shopping_cart: { x: 0, y: 0.65, z: -0.05 },
  rubber_duck:   { x: 0, y: 0.55, z: 0.1 },
  piano:         { x: 0, y: 0.60, z: 0 },
  boat:          { x: 0, y: 0.45, z: 0 },
  rocket:        { x: 0, y: 0.50, z: -0.1 }
};
```

| Fuselage | Wing Y | Notes |
|----------|--------|-------|
| `bathtub` | 0.50 | Wings sit on rim of tub |
| `shopping_cart` | 0.65 | Higher -- wings above basket top |
| `rubber_duck` | 0.55 | Mid-body, just above the waterline |
| `piano` | 0.60 | Wings emerge from near the open lid |
| `boat` | 0.45 | Low -- wings near gunwale |
| `rocket` | 0.50 | Mid-body, below the nose cone |

### Propulsion mount positions

```javascript
const PROPULSION_MOUNT = {
  bathtub:       { x: -0.65, y: 0.35, z: 0 },
  shopping_cart: { x: -0.55, y: 0.40, z: 0 },
  rubber_duck:   { x: -0.75, y: 0.30, z: 0 },
  piano:         { x: -0.80, y: 0.40, z: 0 },
  boat:          { x: -0.85, y: 0.25, z: 0 },
  rocket:        { x: -0.95, y: 0.35, z: 0 }
};
```

### Pilot seat offsets

```javascript
const PILOT_SEAT_OFFSETS = {
  bathtub:       { x: 0, y: 0.45, z: 0 },
  shopping_cart: { x: 0, y: 0.50, z: -0.1 },
  rubber_duck:   { x: 0, y: 0.50, z: 0.1 },
  piano:         { x: 0, y: 0.55, z: 0 },
  boat:          { x: 0, y: 0.35, z: 0 },
  rocket:        { x: 0, y: 0.40, z: 0.2 }
};
```

---

## Decoration Anchor Points

Each fuselage type has anchor points for the 4 decoration slots. A `_default`
fallback is used if a fuselage-specific anchor is not defined.

```javascript
const DECORATION_ANCHORS = {
  _default: {
    fuselage_side: { x: 0, y: 0.30, z: 0.35, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.2, y: 0.50, z: 0 },      // Applied to both tips
    nose:          { x: 0.70, y: 0.30, z: 0 },
    tail:          { x: -0.70, y: 0.40, z: 0 }
  },

  bathtub: {
    fuselage_side: { x: 0, y: 0.30, z: 0.35, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.2, y: 0.50, z: 0 },
    nose:          { x: 0.65, y: 0.35, z: 0 },
    tail:          { x: -0.65, y: 0.40, z: 0 }
  },

  shopping_cart: {
    fuselage_side: { x: 0, y: 0.40, z: 0.30, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.1, y: 0.65, z: 0 },
    nose:          { x: 0.55, y: 0.45, z: 0 },
    tail:          { x: -0.55, y: 0.50, z: 0 }
  },

  rubber_duck: {
    fuselage_side: { x: 0.1, y: 0.30, z: 0.45, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.2, y: 0.55, z: 0 },
    nose:          { x: 0.75, y: 0.50, z: 0 },   // On the beak
    tail:          { x: -0.70, y: 0.35, z: 0 }
  },

  piano: {
    fuselage_side: { x: 0, y: 0.35, z: 0.45, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.3, y: 0.60, z: 0 },
    nose:          { x: 0.80, y: 0.40, z: 0 },
    tail:          { x: -0.80, y: 0.45, z: 0 }
  },

  boat: {
    fuselage_side: { x: 0, y: 0.25, z: 0.35, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.2, y: 0.45, z: 0 },
    nose:          { x: 0.85, y: 0.30, z: 0 },    // Bow
    tail:          { x: -0.85, y: 0.30, z: 0 }     // Stern
  },

  rocket: {
    fuselage_side: { x: 0.1, y: 0.35, z: 0.30, rotation: { y: Math.PI / 2 } },
    wing_tip:      { x: 1.1, y: 0.50, z: 0 },
    nose:          { x: 0.95, y: 0.50, z: 0 },    // Nose cone tip
    tail:          { x: -0.90, y: 0.40, z: 0 }     // Between tail fins
  }
};
```

### Mirroring wing tip decorations

Wing tip decorations are duplicated: one at the left tip, one at the right tip
(mirrored on X axis).

```javascript
function attachWingTipDecos(craftGroup, decoModel, fuselageType) {
  const anchor = DECORATION_ANCHORS[fuselageType]?.wing_tip
    || DECORATION_ANCHORS._default.wing_tip;

  // Left tip
  const left = decoModel.clone();
  left.position.set(-anchor.x, anchor.y, anchor.z);
  left.name = 'Deco_wing_tip_left';
  craftGroup.add(left);

  // Right tip (mirrored)
  const right = decoModel.clone();
  right.position.set(anchor.x, anchor.y, anchor.z);
  right.scale.x = -1; // Mirror
  right.name = 'Deco_wing_tip_right';
  craftGroup.add(right);
}
```

---

## Placeholder Geometry Builders

Complete set of procedural fallback geometry for when GLB assets are unavailable.

### Fuselage placeholders

```javascript
function createPlaceholderFuselage(type) {
  const group = new THREE.Group();
  const white = new THREE.MeshToonMaterial({ color: 0xFFFFFF });

  switch (type) {
    case 'bathtub': {
      const body = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.5, 0.6), white.clone());
      body.position.y = 0.25;
      group.add(body);
      // Claw feet
      const footMat = new THREE.MeshToonMaterial({ color: 0xCCBB00 });
      [[-0.4, -0.2], [-0.4, 0.2], [0.4, -0.2], [0.4, 0.2]].forEach(([x, z]) => {
        const foot = new THREE.Mesh(new THREE.CylinderGeometry(0.05, 0.08, 0.15, 8), footMat);
        foot.position.set(x, 0.07, z);
        group.add(foot);
      });
      // Faucet
      const faucet = new THREE.Mesh(new THREE.CylinderGeometry(0.03, 0.03, 0.2, 8), footMat);
      faucet.position.set(0.55, 0.55, 0);
      group.add(faucet);
      break;
    }

    case 'shopping_cart': {
      const basket = new THREE.Mesh(
        new THREE.BoxGeometry(1.0, 0.6, 0.5),
        new THREE.MeshToonMaterial({ color: 0xCCCCCC, wireframe: true })
      );
      basket.position.y = 0.5;
      group.add(basket);
      // Handle
      const handle = new THREE.Mesh(
        new THREE.BoxGeometry(0.05, 0.3, 0.5),
        new THREE.MeshToonMaterial({ color: 0x888888 })
      );
      handle.position.set(-0.5, 0.7, 0);
      group.add(handle);
      // Wheels
      const wheelMat = new THREE.MeshToonMaterial({ color: 0x444444 });
      [[-0.35, -0.2], [-0.35, 0.2], [0.35, -0.2], [0.35, 0.2]].forEach(([x, z]) => {
        const wheel = new THREE.Mesh(new THREE.CylinderGeometry(0.08, 0.08, 0.03, 12), wheelMat);
        wheel.rotation.x = Math.PI / 2;
        wheel.position.set(x, 0.08, z);
        group.add(wheel);
      });
      break;
    }

    case 'rubber_duck': {
      // Body
      const body = new THREE.Mesh(
        new THREE.SphereGeometry(0.5, 16, 12),
        new THREE.MeshToonMaterial({ color: 0xFFDD00 })
      );
      body.position.y = 0.4;
      body.scale.set(1.2, 1.0, 1.0);
      group.add(body);
      // Head
      const head = new THREE.Mesh(
        new THREE.SphereGeometry(0.25, 12, 10),
        new THREE.MeshToonMaterial({ color: 0xFFDD00 })
      );
      head.position.set(0.5, 0.8, 0);
      group.add(head);
      // Beak
      const beak = new THREE.Mesh(
        new THREE.ConeGeometry(0.08, 0.15, 8),
        new THREE.MeshToonMaterial({ color: 0xFF8800 })
      );
      beak.rotation.z = -Math.PI / 2;
      beak.position.set(0.75, 0.8, 0);
      group.add(beak);
      // Eyes
      const eyeMat = new THREE.MeshToonMaterial({ color: 0x111111 });
      [[0.58, 0.9, 0.12], [0.58, 0.9, -0.12]].forEach(([x, y, z]) => {
        const eye = new THREE.Mesh(new THREE.SphereGeometry(0.04, 8, 8), eyeMat);
        eye.position.set(x, y, z);
        group.add(eye);
      });
      break;
    }

    case 'piano': {
      // Body
      const body = new THREE.Mesh(new THREE.BoxGeometry(1.5, 0.4, 0.8), white.clone());
      body.position.y = 0.4;
      group.add(body);
      // Lid (angled)
      const lid = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.03, 0.75), white.clone());
      lid.position.set(0.2, 0.65, 0);
      lid.rotation.z = 0.3;
      group.add(lid);
      // Keys
      const keys = new THREE.Mesh(
        new THREE.BoxGeometry(1.2, 0.05, 0.15),
        new THREE.MeshToonMaterial({ color: 0xFFFFF0 })
      );
      keys.position.set(0, 0.62, 0.35);
      group.add(keys);
      // Legs
      const legMat = new THREE.MeshToonMaterial({ color: 0x222222 });
      [[-0.5, -0.3], [-0.5, 0.3], [0.5, 0]].forEach(([x, z]) => {
        const leg = new THREE.Mesh(new THREE.CylinderGeometry(0.04, 0.06, 0.4, 8), legMat);
        leg.position.set(x, 0.1, z);
        group.add(leg);
      });
      break;
    }

    case 'boat': {
      // Hull -- stretched box with tapered front
      const hull = new THREE.Mesh(new THREE.BoxGeometry(1.6, 0.4, 0.6), white.clone());
      hull.position.y = 0.2;
      group.add(hull);
      // Bench
      const bench = new THREE.Mesh(
        new THREE.BoxGeometry(0.1, 0.15, 0.5),
        new THREE.MeshToonMaterial({ color: 0x8B6914 })
      );
      bench.position.set(0, 0.45, 0);
      group.add(bench);
      break;
    }

    case 'rocket': {
      // Tube body
      const tube = new THREE.Mesh(
        new THREE.CylinderGeometry(0.25, 0.25, 1.5, 12),
        white.clone()
      );
      tube.rotation.z = Math.PI / 2;
      tube.position.set(0, 0.4, 0);
      group.add(tube);
      // Nose cone
      const cone = new THREE.Mesh(
        new THREE.ConeGeometry(0.25, 0.4, 12),
        new THREE.MeshToonMaterial({ color: 0xFF2200 })
      );
      cone.rotation.z = -Math.PI / 2;
      cone.position.set(0.95, 0.4, 0);
      group.add(cone);
      // Tail fins
      const finMat = new THREE.MeshToonMaterial({ color: 0xFF2200 });
      [0, Math.PI / 2, Math.PI, Math.PI * 1.5].forEach((angle, i) => {
        if (i > 2) return; // Only 3 fins
        const fin = new THREE.Mesh(new THREE.BoxGeometry(0.3, 0.25, 0.03), finMat);
        fin.position.set(-0.7, 0.4, 0);
        fin.rotation.x = angle;
        fin.position.y += Math.cos(angle) * 0.2;
        fin.position.z += Math.sin(angle) * 0.2;
        group.add(fin);
      });
      break;
    }
  }

  group.name = 'Fuselage_' + type;
  return group;
}
```

### Wing placeholders

```javascript
function createPlaceholderWings(type) {
  const group = new THREE.Group();
  const mat = new THREE.MeshToonMaterial({ color: 0xFFFFFF, side: THREE.DoubleSide });

  switch (type) {
    case 'biplane': {
      // Lower pair
      const lowerL = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.03, 0.4), mat.clone());
      lowerL.position.set(-0.9, 0, 0);
      group.add(lowerL);
      const lowerR = new THREE.Mesh(new THREE.BoxGeometry(1.2, 0.03, 0.4), mat.clone());
      lowerR.position.set(0.9, 0, 0);
      group.add(lowerR);
      // Upper pair
      const upperL = lowerL.clone(); upperL.position.y += 0.3; group.add(upperL);
      const upperR = lowerR.clone(); upperR.position.y += 0.3; group.add(upperR);
      // Struts
      const strutMat = new THREE.MeshToonMaterial({ color: 0x8B6914 });
      [[-1.3, 0], [-0.5, 0], [0.5, 0], [1.3, 0]].forEach(([x, z]) => {
        const strut = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.3, 6), strutMat);
        strut.position.set(x, 0.15, z);
        group.add(strut);
      });
      break;
    }

    case 'delta': {
      // Two triangular shapes
      const shape = new THREE.Shape();
      shape.moveTo(0, 0);
      shape.lineTo(-1.0, 0);
      shape.lineTo(-0.3, -0.7);
      shape.closePath();
      const geo = new THREE.ExtrudeGeometry(shape, { depth: 0.03, bevelEnabled: false });
      const leftWing = new THREE.Mesh(geo, mat.clone());
      leftWing.rotation.x = -Math.PI / 2;
      leftWing.position.set(-0.3, 0, 0.35);
      group.add(leftWing);
      const rightWing = leftWing.clone();
      rightWing.scale.z = -1;
      rightWing.position.z = -0.35;
      group.add(rightWing);
      break;
    }

    case 'hang_glider': {
      const leftW = new THREE.Mesh(new THREE.BoxGeometry(1.8, 0.02, 0.8), mat.clone());
      leftW.position.set(-1.1, 0, 0);
      group.add(leftW);
      const rightW = new THREE.Mesh(new THREE.BoxGeometry(1.8, 0.02, 0.8), mat.clone());
      rightW.position.set(1.1, 0, 0);
      group.add(rightW);
      // Center keel
      const keel = new THREE.Mesh(
        new THREE.CylinderGeometry(0.02, 0.02, 1.0, 6),
        new THREE.MeshToonMaterial({ color: 0xAAAAAA })
      );
      keel.rotation.z = Math.PI / 2;
      keel.position.y = -0.1;
      group.add(keel);
      break;
    }

    case 'cardboard_box': {
      const brownMat = new THREE.MeshToonMaterial({ color: 0xC4A45A, side: THREE.DoubleSide });
      const leftFlap = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.04, 0.35), brownMat.clone());
      leftFlap.position.set(-0.7, 0, 0);
      leftFlap.rotation.z = -0.1; // Slight droop
      group.add(leftFlap);
      const rightFlap = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.04, 0.35), brownMat.clone());
      rightFlap.position.set(0.7, 0, 0);
      rightFlap.rotation.z = 0.1;
      group.add(rightFlap);
      break;
    }

    case 'umbrella': {
      const domeMat = mat.clone();
      // Half-sphere domes
      const domeGeo = new THREE.SphereGeometry(0.5, 12, 8, 0, Math.PI * 2, 0, Math.PI / 2);
      const leftDome = new THREE.Mesh(domeGeo, domeMat);
      leftDome.position.set(-0.8, 0, 0);
      group.add(leftDome);
      const rightDome = new THREE.Mesh(domeGeo, domeMat.clone());
      rightDome.position.set(0.8, 0, 0);
      group.add(rightDome);
      // Shafts
      const shaftMat = new THREE.MeshToonMaterial({ color: 0x333333 });
      [[-0.8], [0.8]].forEach(([x]) => {
        const shaft = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.6, 6), shaftMat);
        shaft.position.set(x, -0.3, 0);
        group.add(shaft);
      });
      break;
    }
  }

  group.name = 'Wings_' + type;
  return group;
}
```

### Propulsion placeholders

```javascript
function createPlaceholderPropulsion(type) {
  const group = new THREE.Group();
  if (type === 'none') return group;

  switch (type) {
    case 'rubber_band': {
      // Peg
      const peg = new THREE.Mesh(
        new THREE.CylinderGeometry(0.03, 0.03, 0.3, 8),
        new THREE.MeshToonMaterial({ color: 0x8B6914 })
      );
      peg.rotation.z = Math.PI / 2;
      group.add(peg);
      // Rubber band (torus)
      const band = new THREE.Mesh(
        new THREE.TorusGeometry(0.08, 0.015, 8, 16),
        new THREE.MeshToonMaterial({ color: 0xCC8833 })
      );
      band.position.x = -0.1;
      group.add(band);
      // Propeller
      const blade = new THREE.Mesh(
        new THREE.BoxGeometry(0.02, 0.35, 0.06),
        new THREE.MeshToonMaterial({ color: 0xBB9944 })
      );
      blade.position.x = 0.17;
      blade.name = 'PropBlade';
      group.add(blade);
      break;
    }

    case 'fan': {
      // Cage
      const cage = new THREE.Mesh(
        new THREE.TorusGeometry(0.18, 0.01, 8, 24),
        new THREE.MeshToonMaterial({ color: 0xAAAAAA })
      );
      cage.rotation.y = Math.PI / 2;
      group.add(cage);
      // Blades
      for (let i = 0; i < 3; i++) {
        const blade = new THREE.Mesh(
          new THREE.BoxGeometry(0.02, 0.15, 0.05),
          new THREE.MeshToonMaterial({ color: 0x4488CC })
        );
        blade.rotation.x = (i * Math.PI * 2) / 3;
        blade.position.y = Math.sin(blade.rotation.x) * 0.08;
        blade.position.z = Math.cos(blade.rotation.x) * 0.08;
        blade.name = 'FanBlade' + i;
        group.add(blade);
      }
      break;
    }

    case 'pedal': {
      // Crank axis
      const axis = new THREE.Mesh(
        new THREE.CylinderGeometry(0.02, 0.02, 0.3, 8),
        new THREE.MeshToonMaterial({ color: 0x666666 })
      );
      group.add(axis);
      // Pedals
      const pedalMat = new THREE.MeshToonMaterial({ color: 0x333333 });
      const p1 = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.02, 0.06), pedalMat);
      p1.position.set(0, 0.15, 0.08);
      group.add(p1);
      const p2 = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.02, 0.06), pedalMat);
      p2.position.set(0, -0.15, -0.08);
      group.add(p2);
      // Chain
      const chain = new THREE.Mesh(
        new THREE.TorusGeometry(0.1, 0.008, 6, 20),
        new THREE.MeshToonMaterial({ color: 0x555555 })
      );
      chain.position.x = -0.15;
      group.add(chain);
      break;
    }
  }

  group.name = 'Propulsion_' + type;
  return group;
}
```

### Helmet placeholders

```javascript
function createPlaceholderHelmet(type) {
  const group = new THREE.Group();
  const white = new THREE.MeshToonMaterial({ color: 0xFFFFFF });

  switch (type) {
    case 'aviator': {
      const cap = new THREE.Mesh(new THREE.SphereGeometry(0.14, 12, 8), white.clone());
      cap.scale.y = 0.8;
      group.add(cap);
      // Goggles
      const goggleMat = new THREE.MeshToonMaterial({ color: 0xCC8833 });
      const goggleL = new THREE.Mesh(new THREE.CylinderGeometry(0.05, 0.05, 0.02, 12), goggleMat);
      goggleL.rotation.x = Math.PI / 2;
      goggleL.position.set(0.05, 0.06, 0.12);
      group.add(goggleL);
      const goggleR = goggleL.clone();
      goggleR.position.z = -0.12;
      group.add(goggleR);
      break;
    }

    case 'construction': {
      const dome = new THREE.Mesh(
        new THREE.SphereGeometry(0.14, 12, 8, 0, Math.PI * 2, 0, Math.PI / 2),
        white.clone()
      );
      group.add(dome);
      // Brim
      const brim = new THREE.Mesh(
        new THREE.CylinderGeometry(0.16, 0.16, 0.02, 16),
        white.clone()
      );
      brim.position.y = -0.01;
      group.add(brim);
      break;
    }

    case 'viking': {
      const bowl = new THREE.Mesh(
        new THREE.SphereGeometry(0.14, 12, 8, 0, Math.PI * 2, 0, Math.PI / 2),
        white.clone()
      );
      group.add(bowl);
      // Horns
      const hornMat = new THREE.MeshToonMaterial({ color: 0xEEDDCC });
      const hornL = new THREE.Mesh(new THREE.ConeGeometry(0.04, 0.2, 8), hornMat);
      hornL.position.set(0, 0.05, 0.14);
      hornL.rotation.x = -0.5;
      group.add(hornL);
      const hornR = hornL.clone();
      hornR.position.z = -0.14;
      hornR.rotation.x = 0.5;
      group.add(hornR);
      break;
    }

    case 'astronaut': {
      const bubble = new THREE.Mesh(
        new THREE.SphereGeometry(0.17, 14, 10),
        new THREE.MeshToonMaterial({ color: 0xFFFFFF, transparent: true, opacity: 0.5 })
      );
      group.add(bubble);
      // Gold visor
      const visor = new THREE.Mesh(
        new THREE.SphereGeometry(0.16, 12, 8, 0, Math.PI, 0.3, 0.8),
        new THREE.MeshToonMaterial({ color: 0xFFCC00 })
      );
      visor.position.z = 0.02;
      group.add(visor);
      break;
    }

    case 'chicken': {
      // Head
      const head = new THREE.Mesh(new THREE.SphereGeometry(0.16, 12, 10), white.clone());
      group.add(head);
      // Comb
      const comb = new THREE.Mesh(
        new THREE.BoxGeometry(0.03, 0.12, 0.1),
        new THREE.MeshToonMaterial({ color: 0xFF2200 })
      );
      comb.position.y = 0.18;
      group.add(comb);
      // Beak
      const beak = new THREE.Mesh(
        new THREE.ConeGeometry(0.04, 0.1, 6),
        new THREE.MeshToonMaterial({ color: 0xFF8800 })
      );
      beak.rotation.x = Math.PI / 2;
      beak.position.set(0.02, -0.02, 0.16);
      group.add(beak);
      // Wattle
      const wattle = new THREE.Mesh(
        new THREE.SphereGeometry(0.03, 8, 6),
        new THREE.MeshToonMaterial({ color: 0xFF2200 })
      );
      wattle.position.set(0, -0.1, 0.12);
      group.add(wattle);
      break;
    }
  }

  group.name = 'Helmet_' + type;
  return group;
}
```

### Pilot placeholder

```javascript
function createPlaceholderPilot() {
  const group = new THREE.Group();
  const suitColor = new THREE.MeshToonMaterial({ color: 0xFF6600 }); // Orange jumpsuit
  const skinColor = new THREE.MeshToonMaterial({ color: 0xFFCC99 });

  // Torso
  const torso = new THREE.Mesh(new THREE.BoxGeometry(0.25, 0.3, 0.15), suitColor.clone());
  torso.position.y = 0.35;
  group.add(torso);

  // Head (sphere -- helmet goes on top)
  const head = new THREE.Mesh(new THREE.SphereGeometry(0.1, 10, 8), skinColor.clone());
  head.position.y = 0.6;
  group.add(head);

  // Arms (reaching forward for handlebars)
  const armMat = suitColor.clone();
  [[-0.18, 0.35, 0.12], [0.18, 0.35, 0.12]].forEach(([x, y, z]) => {
    const arm = new THREE.Mesh(new THREE.BoxGeometry(0.08, 0.08, 0.2), armMat);
    arm.position.set(x, y, z);
    group.add(arm);
  });

  // Legs (seated, forward)
  [[0.08, 0.12, 0.15], [-0.08, 0.12, 0.15]].forEach(([x, y, z]) => {
    const leg = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.1, 0.25), suitColor.clone());
    leg.position.set(x, y, z);
    group.add(leg);
  });

  group.name = 'Pilot';
  return group;
}
```

### Decoration placeholders

```javascript
function createPlaceholderDecoration(type, slot) {
  const group = new THREE.Group();

  // Color-code by slot for visibility during development
  const slotColors = {
    side: 0xFF4488,  // Pink -- fuselage side
    tip:  0x44FF88,  // Green -- wing tips
    nose: 0x4488FF,  // Blue -- nose
    tail: 0xFFAA44   // Orange -- tail
  };
  const color = slotColors[slot] || 0xFF00FF;
  const mat = new THREE.MeshToonMaterial({ color });

  switch (type) {
    // Flat decals (side decorations)
    case 'team_banner':
    case 'sponsor_logo':
    case 'flame_paint':
    case 'number':
    case 'stars':
      const decal = new THREE.Mesh(new THREE.PlaneGeometry(0.3, 0.2), mat);
      group.add(decal);
      break;

    // Trailing decorations (wing tips)
    case 'streamers':
    case 'ribbons':
      for (let i = 0; i < 3; i++) {
        const ribbon = new THREE.Mesh(
          new THREE.BoxGeometry(0.02, 0.02, 0.3 + i * 0.1),
          mat.clone()
        );
        ribbon.position.set(0, -i * 0.03, -0.2);
        group.add(ribbon);
      }
      break;

    case 'lights':
      for (let i = 0; i < 5; i++) {
        const bulb = new THREE.Mesh(new THREE.SphereGeometry(0.02, 6, 4), mat.clone());
        bulb.position.set(i * 0.08 - 0.16, 0, 0);
        group.add(bulb);
      }
      break;

    case 'flags':
      const flag = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.08, 0.01), mat);
      flag.position.set(0, 0.1, 0);
      const pole = new THREE.Mesh(
        new THREE.CylinderGeometry(0.008, 0.008, 0.15, 6),
        new THREE.MeshToonMaterial({ color: 0x888888 })
      );
      pole.position.set(-0.06, 0.05, 0);
      group.add(flag);
      group.add(pole);
      break;

    // Nose decorations
    case 'mascot':
      const mascotBody = new THREE.Mesh(new THREE.SphereGeometry(0.12, 10, 8), mat);
      const mascotHead = new THREE.Mesh(new THREE.SphereGeometry(0.08, 8, 6), mat.clone());
      mascotHead.position.y = 0.15;
      group.add(mascotBody);
      group.add(mascotHead);
      break;

    case 'eyes':
      [[0, 0, 0.06], [0, 0, -0.06]].forEach(([x, y, z]) => {
        const eye = new THREE.Mesh(
          new THREE.SphereGeometry(0.06, 10, 8),
          new THREE.MeshToonMaterial({ color: 0xFFFFFF })
        );
        eye.position.set(x, y, z);
        group.add(eye);
        const pupil = new THREE.Mesh(
          new THREE.SphereGeometry(0.03, 8, 6),
          new THREE.MeshToonMaterial({ color: 0x111111 })
        );
        pupil.position.set(x + 0.04, y, z);
        group.add(pupil);
      });
      break;

    case 'horn':
      const hornBody = new THREE.Mesh(
        new THREE.ConeGeometry(0.04, 0.2, 8),
        new THREE.MeshToonMaterial({ color: 0xFF2222 })
      );
      hornBody.rotation.z = -Math.PI / 2;
      group.add(hornBody);
      break;

    case 'figurehead':
      const figBody = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.25, 0.08), mat);
      figBody.position.y = 0.12;
      group.add(figBody);
      break;

    // Tail decorations
    case 'tail_flag':
      const tPole = new THREE.Mesh(
        new THREE.CylinderGeometry(0.01, 0.01, 0.3, 6),
        new THREE.MeshToonMaterial({ color: 0x888888 })
      );
      tPole.position.y = 0.15;
      group.add(tPole);
      const tFlag = new THREE.Mesh(new THREE.PlaneGeometry(0.2, 0.15), mat);
      tFlag.position.set(0.1, 0.25, 0);
      group.add(tFlag);
      break;

    case 'smoke_can':
      const can = new THREE.Mesh(
        new THREE.CylinderGeometry(0.05, 0.05, 0.15, 10),
        new THREE.MeshToonMaterial({ color: 0x888888 })
      );
      group.add(can);
      break;

    case 'parachute':
      const pack = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.2, 0.08), mat);
      group.add(pack);
      break;

    case 'cape':
      const cape = new THREE.Mesh(
        new THREE.PlaneGeometry(0.4, 0.6),
        new THREE.MeshToonMaterial({ color: 0xFF2222, side: THREE.DoubleSide })
      );
      cape.rotation.x = -0.3;
      cape.position.set(-0.2, 0.1, 0);
      group.add(cape);
      break;

    default:
      const fallback = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.15, 0.15), mat);
      group.add(fallback);
  }

  group.name = 'Deco_' + slot + '_' + type;
  return group;
}
```

---

## Turntable Animation

The workshop preview auto-rotates the craft with orbit controls for interactive viewing.

```javascript
// --- Workshop Animation Loop ---

let isWorkshopActive = false;
let craftGroup = null; // Reference to the assembled craft group

function workshopAnimationLoop() {
  if (!isWorkshopActive) return;
  requestAnimationFrame(workshopAnimationLoop);

  // Update orbit controls (handles auto-rotate + damping)
  orbitControls.update();

  // Animate propulsion spinning (if present)
  if (craftGroup) {
    const propBlade = craftGroup.getObjectByName('PropBlade');
    if (propBlade) {
      propBlade.rotation.x += 0.15; // Spin speed
    }

    // Fan blades
    for (let i = 0; i < 3; i++) {
      const fanBlade = craftGroup.getObjectByName('FanBlade' + i);
      if (fanBlade) {
        fanBlade.rotation.x += 0.1;
      }
    }
  }

  renderer.render(workshopScene, workshopCamera);
}

function startWorkshop(craft) {
  isWorkshopActive = true;

  // Remove old craft if present
  if (craftGroup) {
    workshopScene.remove(craftGroup);
  }

  // Build new craft
  assembleCraft(craft, assetManager).then((group) => {
    craftGroup = group;
    workshopScene.add(craftGroup);
  });

  workshopAnimationLoop();
}

function stopWorkshop() {
  isWorkshopActive = false;
}
```

### User interaction controls

```javascript
// Stop auto-rotate when user interacts, resume after idle
let autoRotateTimeout = null;

orbitControls.addEventListener('start', () => {
  orbitControls.autoRotate = false;
  clearTimeout(autoRotateTimeout);
});

orbitControls.addEventListener('end', () => {
  autoRotateTimeout = setTimeout(() => {
    orbitControls.autoRotate = true;
  }, 3000); // Resume after 3s idle
});

// Touch support: OrbitControls handles this natively
// Just ensure the canvas has touch-action: none in CSS
```

---

## Performance Budget

Target: **60 FPS** on mid-range mobile devices (iPhone 12, Samsung Galaxy S21).

| Budget Item | Limit | Notes |
|------------|-------|-------|
| **Total triangles** | 15,000 | Craft (~8K) + workshop env (~5K) + UI (~2K) |
| **Draw calls** | 30 max | Merge geometries where possible |
| **Texture memory** | 8 MB | All textures combined |
| **Shader complexity** | MeshToonMaterial only | No custom shaders in workshop |
| **Shadow maps** | 1 spotlight, 1024x1024 | Only key light casts shadows |
| **Pixel ratio** | Cap at 2.0 | `Math.min(devicePixelRatio, 2)` |
| **Canvas resolution** | Match window | No supersampling |
| **Animation** | RAF only when active | Stop loop when leaving workshop |

### Performance tips

1. **Dispose old models**: When swapping parts, always dispose geometry and materials
2. **Clone materials**: Never share materials between instances (tinting would bleed)
3. **Use BufferGeometry**: All procedural geometry should use BufferGeometry (default in r128)
4. **Limit traverse**: Cache references to animated parts instead of searching every frame
5. **Frustum culling**: Enabled by default; workshop camera FOV keeps everything visible
6. **Object pooling**: For decorations, consider pooling meshes instead of create/dispose cycles

### Monitoring

```javascript
// Optional: FPS counter for development
let frameCount = 0;
let lastFpsTime = performance.now();

function checkFps() {
  frameCount++;
  const now = performance.now();
  if (now - lastFpsTime >= 1000) {
    const fps = frameCount;
    frameCount = 0;
    lastFpsTime = now;
    if (fps < 50) {
      console.warn(`Workshop FPS dropped to ${fps}`);
    }
  }
}
// Call checkFps() inside workshopAnimationLoop
```
