---
name: flugtag-craft-builder
description: >
  Craft builder / flying machine customizer / workshop editor for the Red Bull Flugtag game.
  Triggers: craft builder, flying machine customizer, workshop editor, craft parts,
  wings, fuselage, propulsion, decorations, pilot gear, build screen, customize screen,
  3D preview, asset loading, part picker, craft assembly, toon-shaded workshop.
---

# Flugtag Craft Builder Skill

## Context

Red Bull Flugtag is a single-file HTML flight game built with **Three.js r128**.
Players build absurd flying machines in a workshop, then launch them off a pier ramp
over water for distance and style points.

- **Art style**: Subway Surfers / Fall Guys chunky colorful toon-shaded cartoon
- **Tech stack**: Three.js r128 loaded from CDN, single `index.html`, GLB model assets with procedural geometry fallbacks
- **Persistence**: `window.storage` (wrapper around localStorage) keyed under `flugtag:*`
- **Asset pipeline**: GLB files loaded via `GLTFLoader`, auto-tinted white base meshes, procedural placeholders when GLBs are missing

## References

| Document | Purpose |
|----------|---------|
| [craft-parts-guide.md](references/craft-parts-guide.md) | Full asset catalog, dimensions, generation prompts, file naming |
| [part-stats.md](references/part-stats.md) | Stat formulas, base values, modifiers, example builds |
| [rendering-guide.md](references/rendering-guide.md) | Three.js scene setup, assembly pipeline, lighting, placeholders |

---

## Craft Data Model

This JSON contract is the single source of truth passed between the builder UI and the
flight engine. Saved to persistence and exported as `window.currentCraft`.

```json
{
  "version": 1,
  "name": "My Flying Machine",
  "fuselage": { "type": "bathtub", "color": "#FF4444" },
  "wings": { "type": "biplane", "color": "#FFFFFF" },
  "propulsion": { "type": "rubber_band", "enabled": true },
  "decorations": [
    { "type": "team_banner", "slot": "fuselage_side", "enabled": true },
    { "type": "streamers", "slot": "wing_tip", "enabled": true },
    { "type": "mascot", "slot": "nose", "enabled": false },
    { "type": "tail_flag", "slot": "tail", "enabled": true }
  ],
  "pilot": { "helmet": "aviator", "helmetColor": "#2244FF", "costume": "jumpsuit" },
  "stats": { "lift": 3, "weight": 3, "style": 4 }
}
```

### Field rules

| Field | Type | Constraints |
|-------|------|-------------|
| `version` | int | Always `1` for now |
| `name` | string | 1-24 characters |
| `fuselage.type` | string | One of the 6 fuselage IDs |
| `fuselage.color` | hex string | Any valid hex color |
| `wings.type` | string | One of the 5 wing IDs |
| `wings.color` | hex string | Any valid hex color |
| `propulsion.type` | string | One of the 4 propulsion IDs |
| `propulsion.enabled` | bool | `false` only for `"none"` |
| `decorations[]` | array | Max 4 items, one per slot |
| `decorations[].slot` | string | `fuselage_side`, `wing_tip`, `nose`, `tail` |
| `pilot.helmet` | string | One of the 5 helmet IDs |
| `pilot.helmetColor` | hex string | Any valid hex color |
| `pilot.costume` | string | Currently always `"jumpsuit"` |
| `stats.*` | int | Clamped 1-5 |

---

## Part Catalog IDs

### Fuselages (6)

| ID | Display Name | Description |
|----|-------------|-------------|
| `bathtub` | Bathtub | Classic claw-foot bathtub, medium all-rounder |
| `shopping_cart` | Shopping Cart | Wire-frame cart, light but low style |
| `rubber_duck` | Rubber Duck | Giant inflatable duck, high style low lift |
| `piano` | Grand Piano | Baby grand, heavy but stylish |
| `boat` | Rowboat | Wooden rowboat, good lift |
| `rocket` | Cardboard Rocket | Painted cardboard tube, best lift |

### Wings (5)

| ID | Display Name | Description |
|----|-------------|-------------|
| `biplane` | Biplane Wings | Dual stacked wings, great lift |
| `delta` | Delta Wings | Swept triangular, balanced |
| `hang_glider` | Hang Glider | Large fabric delta, max lift |
| `cardboard_box` | Cardboard Flaps | Flat cardboard, low lift high comedy |
| `umbrella` | Umbrellas | Two open umbrellas, decent style |

### Propulsion (4)

| ID | Display Name | Description |
|----|-------------|-------------|
| `rubber_band` | Rubber Band Motor | Wound-up propeller, small boost |
| `fan` | Desk Fan | Battery-powered fan, moderate push |
| `pedal` | Pedal Power | Bike pedals driving a prop, sustained |
| `none` | None | No propulsion, pure glide |

### Decorations

**Fuselage Side** (slot: `fuselage_side`, 5 types):

| ID | Display Name |
|----|-------------|
| `team_banner` | Team Banner |
| `sponsor_logo` | Sponsor Logo |
| `flame_paint` | Flame Paint |
| `number` | Race Number |
| `stars` | Star Stickers |

**Wing Tip** (slot: `wing_tip`, 4 types):

| ID | Display Name |
|----|-------------|
| `streamers` | Streamers |
| `ribbons` | Ribbons |
| `lights` | Fairy Lights |
| `flags` | Mini Flags |

**Nose** (slot: `nose`, 4 types):

| ID | Display Name |
|----|-------------|
| `mascot` | Team Mascot |
| `eyes` | Googly Eyes |
| `horn` | Party Horn |
| `figurehead` | Ship Figurehead |

**Tail** (slot: `tail`, 4 types):

| ID | Display Name |
|----|-------------|
| `tail_flag` | Tail Flag |
| `smoke_can` | Smoke Canister |
| `parachute` | Emergency Chute |
| `cape` | Hero Cape |

### Helmets (5)

| ID | Display Name |
|----|-------------|
| `aviator` | Aviator Goggles |
| `construction` | Hard Hat |
| `viking` | Viking Helmet |
| `astronaut` | Space Helmet |
| `chicken` | Chicken Head |

---

## Three.js Scene Hierarchy

```
Scene
 |
 +-- AmbientLight (soft warm, 0xFFEEDD, 0.4)
 +-- WorkshopSpotlight (warm key, position 3,5,3)
 +-- WorkshopFillLight (blue fill, position -2,3,-1)
 +-- WorkshopRimLight (white rim, position 0,2,-4)
 |
 +-- CraftGroup (pivot at origin, turntable rotation)
 |    |
 |    +-- FuselageModel        (position 0, 0, 0)
 |    +-- WingLeft             (position varies per fuselage)
 |    +-- WingRight            (mirrored X of WingLeft)
 |    +-- PropulsionModel      (position at fuselage rear)
 |    +-- DecorationSlots
 |    |    +-- FuselageSideDeco   (left side of fuselage)
 |    |    +-- WingTipLeftDeco    (end of left wing)
 |    |    +-- WingTipRightDeco   (end of right wing)
 |    |    +-- NoseDeco           (front of fuselage)
 |    |    +-- TailDeco           (rear of fuselage)
 |    +-- PilotBody            (seated in/on fuselage)
 |         +-- HelmetModel     (on pilot head)
 |
 +-- WorkshopFloor (large plane, checkered texture)
 +-- WorkshopProps
 |    +-- Toolbox
 |    +-- WorkBench
 |    +-- HangingLights
 |
 +-- Camera (PerspectiveCamera, 45 FOV, orbit controls)
```

---

## Editor UI Layout

```
+---------------------------------------------------------------+
|  RED BULL FLUGTAG - WORKSHOP                    [< Back] [?]  |
+---------------------------------------------------------------+
|                                                               |
|                   +---------------------+                     |
|                   |                     |                     |
|                   |    3D CRAFT         |                     |
|                   |    PREVIEW          |                     |
|                   |                     |                     |
|                   |   (turntable +      |                     |
|                   |    drag to orbit)   |                     |
|                   |                     |                     |
|                   +---------------------+                     |
|                                                               |
|  +---+---+---+---+---+---+---+---+                           |
|  |Bod|Wng|Pwr|Sid|Tip|Nos|Tal|Plt|  <-- Part picker tabs     |
|  +---+---+---+---+---+---+---+---+                           |
|                                                               |
|  +-------+ +-------+ +-------+ +-------+                     |
|  | part  | | part  | | part  | | part  |  <-- Scrollable     |
|  | thumb | | thumb | | thumb | | thumb |      part grid       |
|  | name  | | name  | | name  | | name  |                     |
|  +-------+ +-------+ +-------+ +-------+                     |
|                                                               |
|  Color: [#] [O][O][O][O][O][O][O][O]  <-- Palette swatches   |
|                                                               |
|  +-------------------------------------------+               |
|  | LFT [====------] 3    Lift coefficient    |               |
|  | WGT [======----] 4    Launch speed        |               |
|  | STY [========--] 5    Style multiplier    |               |
|  +-------------------------------------------+               |
|                                                               |
|  Craft name: [  My Flying Machine  ]                         |
|                                                               |
|           +=============================+                     |
|           ||       LAUNCH!             ||                     |
|           +=============================+                     |
+---------------------------------------------------------------+
```

### Tab mapping

| Tab | Label | Shows |
|-----|-------|-------|
| Bod | Body | 6 fuselage options |
| Wng | Wings | 5 wing options |
| Pwr | Power | 4 propulsion options |
| Sid | Side | 5 fuselage_side decorations |
| Tip | Tips | 4 wing_tip decorations |
| Nos | Nose | 4 nose decorations |
| Tal | Tail | 4 tail decorations |
| Plt | Pilot | 5 helmets + helmet color |

---

## Stats System

### Formula

```
finalStat = clamp(fuselageBase + wingMod + propulsionMod + decoBonus, 1, 5)
```

See [part-stats.md](references/part-stats.md) for complete tables.

### How stats affect gameplay

| Stat | Gameplay Effect | Formula |
|------|----------------|---------|
| **Lift** | Flight lift coefficient | `baseLift = 0.3 + (lift - 1) * 0.1` so Lift 1 = 0.3, Lift 5 = 0.7 |
| **Weight** | Launch speed penalty + in-flight drag | `speedMult = 1.0 - (weight - 1) * 0.08` so Weight 1 = 1.0, Weight 5 = 0.68 |
| **Style** | Showmanship score multiplier | `styleMult = 1.0 + (style - 1) * 0.25` so Style 1 = 1.0, Style 5 = 2.0 |

### Quick reference

- Lift 1 = paper airplane, Lift 5 = actual hang glider
- Weight 1 = featherlight, Weight 5 = grand piano with bricks
- Style 1 = boring, Style 5 = crowd goes wild

---

## Color Tinting Strategy

Fuselages and helmets are modeled/generated as **white base meshes**. Color is applied
at runtime via material tinting so players can pick any color.

```javascript
/**
 * Tint all meshes in a model to the given hex color.
 * Preserves original material properties (roughness, metalness, map).
 * @param {THREE.Object3D} model - The loaded GLB model or placeholder group
 * @param {string} hexColor - Hex color string e.g. "#FF4444"
 */
function tintModel(model, hexColor) {
  const color = new THREE.Color(hexColor);
  model.traverse((child) => {
    if (child.isMesh && child.material) {
      // Clone material so tinting one instance doesn't affect others
      child.material = child.material.clone();
      child.material.color.copy(color);
      // Keep toon shading
      if (!child.material.isMeshToonMaterial) {
        const oldMat = child.material;
        child.material = new THREE.MeshToonMaterial({
          color: color,
          map: oldMat.map || null,
          side: THREE.DoubleSide
        });
      }
    }
  });
}
```

### Which parts get tinted

| Part | Tintable | Source property |
|------|----------|---------------|
| Fuselage | Yes | `craft.fuselage.color` |
| Wings | Yes | `craft.wings.color` |
| Propulsion | No | Uses model's baked color |
| Decorations | No | Uses model's baked color |
| Helmet | Yes | `craft.pilot.helmetColor` |
| Pilot body | No | Fixed jumpsuit color per costume |

---

## Placeholder Fallback

When GLB assets are unavailable (dev mode, loading failure), generate procedural
geometry so the game is always playable.

```javascript
function createPlaceholderFuselage(type) {
  const group = new THREE.Group();
  const mat = new THREE.MeshToonMaterial({ color: 0xFFFFFF });

  switch (type) {
    case 'bathtub': {
      // Rounded box for tub body
      const body = new THREE.Mesh(
        new THREE.BoxGeometry(1.2, 0.5, 0.6),
        mat.clone()
      );
      body.position.y = 0.25;
      group.add(body);
      // Four small cylinders for feet
      for (let i = 0; i < 4; i++) {
        const foot = new THREE.Mesh(
          new THREE.CylinderGeometry(0.05, 0.08, 0.15, 8),
          new THREE.MeshToonMaterial({ color: 0xCCBB00 })
        );
        foot.position.set(
          (i % 2 === 0 ? -0.4 : 0.4),
          0.07,
          (i < 2 ? -0.2 : 0.2)
        );
        group.add(foot);
      }
      break;
    }
    case 'shopping_cart': {
      const basket = new THREE.Mesh(
        new THREE.BoxGeometry(1.0, 0.6, 0.5),
        new THREE.MeshToonMaterial({ color: 0xCCCCCC, wireframe: true })
      );
      basket.position.y = 0.5;
      group.add(basket);
      break;
    }
    case 'rubber_duck': {
      const body = new THREE.Mesh(
        new THREE.SphereGeometry(0.5, 16, 12),
        new THREE.MeshToonMaterial({ color: 0xFFDD00 })
      );
      body.position.y = 0.4;
      body.scale.set(1.2, 1.0, 1.0);
      group.add(body);
      const head = new THREE.Mesh(
        new THREE.SphereGeometry(0.25, 12, 10),
        new THREE.MeshToonMaterial({ color: 0xFFDD00 })
      );
      head.position.set(0.5, 0.8, 0);
      group.add(head);
      break;
    }
    default: {
      // Generic box fallback for piano, boat, rocket
      const box = new THREE.Mesh(
        new THREE.BoxGeometry(1.2, 0.5, 0.6),
        mat.clone()
      );
      box.position.y = 0.25;
      group.add(box);
    }
  }

  group.name = 'Fuselage_' + type;
  return group;
}

function createPlaceholderWings(type) {
  const group = new THREE.Group();
  const mat = new THREE.MeshToonMaterial({ color: 0xFFFFFF, side: THREE.DoubleSide });

  const wingGeo = (function() {
    switch (type) {
      case 'biplane':     return new THREE.BoxGeometry(2.0, 0.03, 0.5);
      case 'delta':       return new THREE.BoxGeometry(1.5, 0.03, 0.8);
      case 'hang_glider': return new THREE.BoxGeometry(2.5, 0.03, 0.7);
      case 'cardboard_box': return new THREE.BoxGeometry(1.0, 0.05, 0.4);
      case 'umbrella':    return new THREE.CircleGeometry(0.6, 12);
      default:            return new THREE.BoxGeometry(1.5, 0.03, 0.5);
    }
  })();

  // Left wing
  const left = new THREE.Mesh(wingGeo, mat.clone());
  left.position.set(-0.8, 0.5, 0);
  left.name = 'WingLeft';
  group.add(left);

  // Right wing
  const right = new THREE.Mesh(wingGeo, mat.clone());
  right.position.set(0.8, 0.5, 0);
  right.name = 'WingRight';
  group.add(right);

  // Biplane gets a second tier
  if (type === 'biplane') {
    const topL = left.clone();
    topL.position.y += 0.3;
    group.add(topL);
    const topR = right.clone();
    topR.position.y += 0.3;
    group.add(topR);
  }

  group.name = 'Wings_' + type;
  return group;
}

function createPlaceholderPropulsion(type) {
  const group = new THREE.Group();
  if (type === 'none') return group;

  const mat = new THREE.MeshToonMaterial({ color: 0x888888 });
  const prop = new THREE.Mesh(
    new THREE.CylinderGeometry(0.05, 0.05, 0.4, 8),
    mat
  );
  prop.rotation.z = Math.PI / 2;
  group.add(prop);

  // Propeller blade
  const blade = new THREE.Mesh(
    new THREE.BoxGeometry(0.02, 0.4, 0.08),
    new THREE.MeshToonMaterial({ color: 0xBB6633 })
  );
  blade.position.x = 0.22;
  group.add(blade);

  group.name = 'Propulsion_' + type;
  group.position.set(-0.7, 0.3, 0); // rear of fuselage
  return group;
}

function createPlaceholderDecoration(type, slot) {
  const group = new THREE.Group();
  const mat = new THREE.MeshToonMaterial({ color: 0xFF4488 });

  // Small primitive stand-in
  const deco = new THREE.Mesh(
    new THREE.BoxGeometry(0.15, 0.15, 0.15),
    mat
  );
  group.add(deco);
  group.name = 'Deco_' + slot + '_' + type;
  return group;
}
```

---

## Persistence

```javascript
// --- Save / Load Craft ---

const CRAFT_STORAGE_KEY = 'flugtag:craft';

const DEFAULT_CRAFT = {
  version: 1,
  name: 'My Flying Machine',
  fuselage: { type: 'bathtub', color: '#FF4444' },
  wings: { type: 'biplane', color: '#FFFFFF' },
  propulsion: { type: 'rubber_band', enabled: true },
  decorations: [
    { type: 'team_banner', slot: 'fuselage_side', enabled: true },
    { type: 'streamers', slot: 'wing_tip', enabled: true },
    { type: 'mascot', slot: 'nose', enabled: false },
    { type: 'tail_flag', slot: 'tail', enabled: true }
  ],
  pilot: { helmet: 'aviator', helmetColor: '#2244FF', costume: 'jumpsuit' },
  stats: { lift: 3, weight: 3, style: 4 }
};

function saveCraft(craft) {
  try {
    window.storage.setItem(CRAFT_STORAGE_KEY, JSON.stringify(craft));
    window.currentCraft = craft;
  } catch (e) {
    console.warn('Failed to save craft:', e);
  }
}

function loadCraft() {
  try {
    const raw = window.storage.getItem(CRAFT_STORAGE_KEY);
    if (raw) {
      const craft = JSON.parse(raw);
      if (craft && craft.version === 1) {
        window.currentCraft = craft;
        return craft;
      }
    }
  } catch (e) {
    console.warn('Failed to load craft:', e);
  }
  // Return default and persist it
  const craft = JSON.parse(JSON.stringify(DEFAULT_CRAFT));
  saveCraft(craft);
  return craft;
}

function recalcStats(craft) {
  // Import stat tables from part-stats.md
  const fStats = FUSELAGE_STATS[craft.fuselage.type];
  const wMods  = WING_MODIFIERS[craft.wings.type];
  const pMods  = PROPULSION_MODIFIERS[craft.propulsion.type];

  let lift   = fStats.lift   + wMods.lift   + pMods.lift;
  let weight = fStats.weight + wMods.weight + pMods.weight;
  let style  = fStats.style  + wMods.style  + pMods.style;

  // Decoration nudges
  craft.decorations.forEach(d => {
    if (!d.enabled) return;
    const nudge = DECORATION_NUDGES[d.slot]?.[d.type];
    if (nudge) {
      lift   += nudge.lift   || 0;
      weight += nudge.weight || 0;
      style  += nudge.style  || 0;
    }
  });

  craft.stats = {
    lift:   Math.max(1, Math.min(5, Math.round(lift))),
    weight: Math.max(1, Math.min(5, Math.round(weight))),
    style:  Math.max(1, Math.min(5, Math.round(style)))
  };

  return craft;
}

// --- Init on page load ---
// const currentCraft = loadCraft();
```

---

## Implementation Checklist

- [ ] Create HTML skeleton with workshop UI layout (tabs, preview canvas, stat bars)
- [ ] Initialize Three.js scene with workshop lighting rig
- [ ] Implement `AssetManager` with GLB loading, caching, and fallback
- [ ] Build all 6 placeholder fuselage geometries
- [ ] Build all 5 placeholder wing geometries
- [ ] Build placeholder propulsion and decoration geometries
- [ ] Implement `assembleCraft()` to build scene graph from craft data
- [ ] Implement `tintModel()` for runtime color application
- [ ] Build part picker tab UI with thumbnail grid
- [ ] Implement color palette picker with hex input
- [ ] Wire up stat bar display (LFT / WGT / STY) with animated fill
- [ ] Implement `recalcStats()` with full formula from part-stats.md
- [ ] Add turntable auto-rotation + mouse/touch orbit controls
- [ ] Implement `saveCraft()` / `loadCraft()` persistence
- [ ] Export `window.currentCraft` for flight engine consumption
- [ ] Craft name input with validation (1-24 chars)
- [ ] LAUNCH button transitions to flight scene
- [ ] Add workshop ambient sound / click SFX hooks
- [ ] Responsive layout for mobile / desktop
- [ ] Performance test: maintain 60fps with full craft assembled

---

## Test Prompts

Use these to verify the craft builder implementation:

1. **"Build the craft workshop screen"** -- Should create the full editor UI with 3D preview, tab-based part picker, color palette, stat bars, and LAUNCH button.

2. **"Add all the fuselage types with placeholder geometry"** -- Should implement `createPlaceholderFuselage()` for all 6 types (bathtub, shopping_cart, rubber_duck, piano, boat, rocket) with distinct silhouettes.

3. **"Make the part picker update the 3D preview in real-time"** -- Selecting a new wing type should immediately swap the wing model in the scene, recalculate stats, and update the stat bars.

4. **"Implement craft persistence so my build saves between sessions"** -- Should save to `flugtag:craft` on every change and reload on page load, with `window.currentCraft` always reflecting current state.

5. **"Add color picking for the fuselage and helmet"** -- Should show a palette of preset colors plus a hex input, and tint the fuselage/helmet model in real-time using `tintModel()`.

6. **"Show me a rubber duck with hang glider wings, pedal power, and a viking helmet"** -- Should be able to compose any valid combination and see it in the 3D preview with correct stat calculation.
