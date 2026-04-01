---
name: flugtag-flight-engine
description: >
  Flight gameplay engine for Red Bull Flugtag HTML game. Covers ramp launch,
  gliding mechanics, flight physics, pitch controls, wind system, splash landing,
  distance scoring, game loop, ramp run phase, air controls, thermals, updrafts,
  angle of attack, altitude tracking, flight camera, power meter, environmental
  interactions, craft stat integration, HUD overlay, and scoring system.
---

# Flugtag Flight Engine

## Game Design Summary

Red Bull Flugtag meets arcade flight physics. The core gameplay loop:

1. **Build** a flying machine in the Workshop (craft builder skill)
2. **Run** down a ramp -- rapidly tap/click to build speed on a power meter
3. **Launch** off the ramp edge into the air
4. **Glide** by controlling pitch -- balance lift vs drag to fly as far as possible
5. **Splash** into the water when altitude hits zero
6. **Score** based on distance traveled, with style multiplier bonuses

The tone is cartoonish and exaggerated. Craft should feel floaty, physics should reward
skillful pitch control, and every run should feel like a spectacle. Think Tiny Wings
crossed with a paper airplane simulator.

---

## Controls

### Desktop
| Input           | Action                          |
|-----------------|---------------------------------|
| Up Arrow / W    | Pitch nose up                   |
| Down Arrow / S  | Pitch nose down                 |
| Space           | Boost (limited fuel)            |
| Rapid Click     | Build power during ramp run     |

### Mobile
| Input                  | Action                          |
|------------------------|---------------------------------|
| Touch upper half       | Pitch nose up                   |
| Touch lower half       | Pitch nose down                 |
| Device tilt (optional) | Pitch via accelerometer         |
| Rapid tap              | Build power during ramp run     |

### Ramp Phase
- Rapid tap or rapid click fills the power meter
- Tap frequency determines power accumulation rate
- A "perfect zone" window at the ramp edge grants a launch bonus

---

## Architecture Overview

Single HTML file artifact. Uses Three.js r128 with a toon-shaded cartoon art style.
GLB model loading with full procedural geometry fallbacks for every visual element.

**Craft data** is read from `window.storage` under the key `currentCraft`:

```javascript
// Expected craft data structure
const craft = JSON.parse(window.storage?.getItem('currentCraft') || '{}');
// {
//   name: "The Flying Banana",
//   stats: { lift: 3, weight: 2, style: 4 },  // each 1-5
//   model: "banana-plane",                      // GLB asset key or null
//   colors: { primary: "#FFD700", secondary: "#8B4513" }
// }
```

The game renders as a **side-scrolling view** during flight (camera follows craft
from the side, like Tiny Wings or Angry Birds). The ramp phase uses a closer camera
angle, and the splash zooms in for impact.

---

## Game Phases

```
+------------+     +-----------+     +--------+     +--------+     +--------+     +---------+
|  LOADING   | --> | RAMP_RUN  | --> | LAUNCH | --> | FLIGHT | --> | SPLASH | --> | RESULTS |
+------------+     +-----------+     +--------+     +--------+     +--------+     +---------+
     |                  |                 |              |              |              |
  Load craft       Tap to build      Brief 0.5s     Full physics   Water impact   Score calc
  Init scene       speed on meter    transition     Pitch control  Particles      Medal award
  Load assets      Power meter UI    Calc initial   Env forces     Craft bobs     Share/retry
  Setup lights     Crowd cheering    velocity       Camera follow  Distance lock  Leaderboard
```

### Phase Details

**LOADING**
- Parse craft data from `window.storage`
- Initialize Three.js scene, renderer, lights
- Build environment (water plane, sky gradient, pier/ramp, clouds)
- Load or generate craft 3D model
- Transition to RAMP_RUN when ready

**RAMP_RUN**
- Camera positioned close to the ramp, slightly behind craft
- Team of characters pushes the craft down the ramp
- Player rapidly taps/clicks to fill the power meter
- Speed increases based on tap frequency and craft weight
- Crowd cheering audio scales with power level
- Duration: length of ramp at current speed (typically 2-4 seconds)
- Ends when craft reaches ramp edge

**LAUNCH**
- Brief 0.5-second cinematic moment
- Camera pulls back to flight view
- Initial velocity calculated from ramp speed and ramp angle
- Craft leaves the ramp edge
- Crowd reacts (cheer or gasp based on launch speed)

**FLIGHT**
- Full 2D flight physics simulation
- Player controls pitch (nose up/down)
- Environmental forces: thermals, wind gusts, bird collisions
- Camera follows craft in side-scrolling view
- HUD displays altitude, distance, speed, pitch guide
- Continues until craft altitude reaches 0 (water level)

**SPLASH**
- Triggered when craft y-position <= 0
- Splash particle effect (size proportional to impact speed)
- Camera zooms to impact point
- Craft bobs on water surface
- Distance is locked and displayed prominently
- Brief pause (1.5 seconds) before transition

**RESULTS**
- Final distance displayed with animation (counting up)
- Style multiplier applied
- Bonuses tallied (perfect launch, thermal chains, etc.)
- Medal awarded (Bronze/Silver/Gold/Platinum thresholds)
- Retry and share buttons

---

## Integration with Craft Builder

The craft builder skill produces craft with three stats (1-5 scale). Each stat
directly maps to flight physics parameters:

| Stat   | Effect on Gameplay                                         |
|--------|------------------------------------------------------------|
| Lift   | Base lift coefficient: `0.3 + (lift - 1) * 0.1`           |
| Weight | Mass in kg: `50 + (weight - 1) * 15`                      |
| Style  | Score multiplier: `1.0 + (style - 1) * 0.25`              |

### Derived Physics Constants

```javascript
function deriveCraftPhysics(stats) {
  const lift   = stats.lift   || 3;
  const weight = stats.weight || 3;
  const style  = stats.style  || 3;

  return {
    // Lift coefficient (higher = more lift per unit speed)
    baseLiftCoeff: 0.3 + (lift - 1) * 0.1,       // Range: 0.3 - 0.7

    // Mass in kg (heavier = harder to launch, falls faster)
    mass: 50 + (weight - 1) * 15,                 // Range: 50 - 110 kg

    // Effective wing area in m^2
    wingArea: 3.0 + (lift - 1) * 0.5,             // Range: 3.0 - 5.0 m^2

    // Drag coefficient (heavier craft = more frontal area)
    dragCoeff: 0.15 + (weight - 1) * 0.02,        // Range: 0.15 - 0.23

    // Score multiplier
    styleMultiplier: 1.0 + (style - 1) * 0.25,    // Range: 1.0 - 2.0x

    // Max ramp speed penalty from weight
    rampSpeedPenalty: 1.0 - (weight - 1) * 0.05,  // Range: 1.0 - 0.80

    // Boost fuel (lighter craft = more boost)
    boostFuel: 3.0 - (weight - 1) * 0.4,          // Range: 3.0 - 1.4 seconds
  };
}
```

### Stat Trade-offs

- **High Lift, Low Weight**: Floaty glider -- easy to keep aloft but slow, less momentum
- **Low Lift, High Weight**: Cannonball -- fast launch, nosedives quickly
- **Balanced**: Mid-range flight, forgiving controls
- **High Style**: Lower flight stats but big score multiplier -- risky high-reward build

---

## Three.js Flight Scene Hierarchy

```
Scene
  |
  +-- AmbientLight (soft fill, intensity 0.6)
  +-- DirectionalLight (sun, intensity 0.9, cast shadows)
  |
  +-- SkyGradient (large sphere or plane, gradient blue-to-white)
  |
  +-- CloudLayer[]
  |     +-- Cloud_0 (cluster of spheres, toon material)
  |     +-- Cloud_1
  |     +-- Cloud_2 ... (object pool, ~20 clouds)
  |
  +-- WaterPlane
  |     +-- AnimatedWaterMesh (plane with vertex animation, toon blue)
  |     +-- FoamParticles (near shore)
  |
  +-- RampGroup
  |     +-- Platform (pier structure, wooden planks)
  |     +-- RampSurface (angled plane with rails)
  |     +-- CrowdGroup
  |     |     +-- Spectator[] (simple capsule people, waving animation)
  |     +-- Banner (Red Bull Flugtag sign)
  |     +-- Flags[] (animated cloth sim or vertex wobble)
  |
  +-- EnvironmentGroup
  |     +-- ThermalIndicator[] (shimmering air columns, subtle particles)
  |     +-- BirdFlock[] (simple animated V-shapes)
  |     +-- BoatGroup[] (small boats on water, spectators)
  |     +-- DistanceMarker[] (buoys every 25m with distance labels)
  |     +-- IslandGroup[] (distant scenery)
  |
  +-- CraftGroup
  |     +-- CraftModel (loaded GLB or procedural mesh)
  |     +-- WindTrailParticles (particle system behind craft)
  |     +-- PitchIndicator (small arrow showing pitch angle)
  |     +-- BoostFlame (particle effect when boost active)
  |
  +-- SplashEffects (object pool)
  |     +-- SplashRing[] (expanding torus geometries)
  |     +-- WaterDroplet[] (small spheres with gravity)
  |     +-- SprayParticles
  |
  +-- HUDGroup (rendered as HTML overlay, not 3D)
  +-- Camera (OrthographicCamera or PerspectiveCamera in side view)
```

---

## Flight Physics Model

### Constants

```javascript
const PHYSICS = {
  GRAVITY:       -9.8,    // m/s^2
  AIR_DENSITY:    1.225,  // kg/m^3 at sea level
  RAMP_HEIGHT:   10.0,    // meters above water
  RAMP_ANGLE:    15.0,    // degrees
  MAX_PITCH:     30.0,    // degrees (clamp)
  PITCH_SPEED:   60.0,    // degrees per second
  STALL_ANGLE:   20.0,    // degrees -- lift drops dramatically
  GROUND_EFFECT_HEIGHT: 3.0,  // meters -- bonus lift near water
  GROUND_EFFECT_MULT:   1.5,  // lift multiplier in ground effect
  HIGH_ALT_THRESHOLD:  25.0,  // meters -- reduced lift above this
  HIGH_ALT_PENALTY:     0.7,  // lift multiplier above threshold
  TERMINAL_VX:         40.0,  // m/s horizontal cap
  TERMINAL_VY:        -30.0,  // m/s vertical cap (falling)
};
```

### Flight State

```javascript
class FlightState {
  constructor(craftPhysics) {
    this.cp = craftPhysics; // from deriveCraftPhysics()

    // Position (meters, origin = ramp edge base)
    this.px = 0;
    this.py = PHYSICS.RAMP_HEIGHT;

    // Velocity (m/s)
    this.vx = 0;
    this.vy = 0;

    // Pitch angle (degrees, positive = nose up)
    this.pitch = 0;
    this.targetPitch = 0;

    // Boost
    this.boostFuel = this.cp.boostFuel;
    this.boosting = false;

    // Stats
    this.maxAltitude = this.py;
    this.distance = 0;
    this.thermalChain = 0;
    this.inThermal = false;
  }
}
```

### Core Update Function

```javascript
function updateFlight(state, dt) {
  const cp = state.cp;
  const speed = Math.sqrt(state.vx * state.vx + state.vy * state.vy);

  // --- Pitch interpolation ---
  const pitchDiff = state.targetPitch - state.pitch;
  const maxChange = PHYSICS.PITCH_SPEED * dt;
  state.pitch += Math.max(-maxChange, Math.min(maxChange, pitchDiff));
  state.pitch = clamp(state.pitch, -PHYSICS.MAX_PITCH, PHYSICS.MAX_PITCH);

  // --- Angle of attack ---
  const velocityAngle = Math.atan2(state.vy, state.vx) * (180 / Math.PI);
  const aoa = state.pitch - velocityAngle; // angle of attack in degrees
  const aoaRad = aoa * (Math.PI / 180);
  const pitchRad = state.pitch * (Math.PI / 180);

  // --- Lift force ---
  let liftCoeff = cp.baseLiftCoeff;

  // Stall: if AoA exceeds stall angle, lift drops sharply
  if (Math.abs(aoa) > PHYSICS.STALL_ANGLE) {
    const excess = Math.abs(aoa) - PHYSICS.STALL_ANGLE;
    liftCoeff *= Math.max(0.1, 1.0 - excess * 0.08);
  }

  // Ground effect: bonus lift near water
  if (state.py < PHYSICS.GROUND_EFFECT_HEIGHT && state.py > 0) {
    const factor = 1.0 + (PHYSICS.GROUND_EFFECT_MULT - 1.0)
                       * (1.0 - state.py / PHYSICS.GROUND_EFFECT_HEIGHT);
    liftCoeff *= factor;
  }

  // High altitude penalty
  if (state.py > PHYSICS.HIGH_ALT_THRESHOLD) {
    liftCoeff *= PHYSICS.HIGH_ALT_PENALTY;
  }

  // Lift = 0.5 * rho * v^2 * S * CL * sin(2 * aoa)
  const liftMag = 0.5 * PHYSICS.AIR_DENSITY * speed * speed
                * cp.wingArea * liftCoeff * Math.sin(2 * aoaRad);

  // Lift acts perpendicular to velocity
  const liftAngle = velocityAngle + 90; // perpendicular
  const liftAngleRad = liftAngle * (Math.PI / 180);
  const liftFx = liftMag * Math.cos(liftAngleRad);
  const liftFy = liftMag * Math.sin(liftAngleRad);

  // --- Drag force ---
  const dragMag = 0.5 * PHYSICS.AIR_DENSITY * speed * speed
                * cp.wingArea * cp.dragCoeff;

  // Drag opposes velocity
  const dragFx = speed > 0.01 ? -dragMag * (state.vx / speed) : 0;
  const dragFy = speed > 0.01 ? -dragMag * (state.vy / speed) : 0;

  // --- Gravity ---
  const gravFy = cp.mass * PHYSICS.GRAVITY;

  // --- Boost ---
  let boostFx = 0;
  if (state.boosting && state.boostFuel > 0) {
    boostFx = 200; // Newtons forward
    state.boostFuel -= dt;
    if (state.boostFuel < 0) state.boostFuel = 0;
  }

  // --- Net force and acceleration ---
  const ax = (liftFx + dragFx + boostFx) / cp.mass;
  const ay = (liftFy + dragFy + gravFy) / cp.mass;

  // --- Integrate velocity ---
  state.vx += ax * dt;
  state.vy += ay * dt;

  // Clamp terminal velocity
  state.vx = clamp(state.vx, -5, PHYSICS.TERMINAL_VX);
  state.vy = clamp(state.vy, PHYSICS.TERMINAL_VY, 30);

  // --- Integrate position ---
  state.px += state.vx * dt;
  state.py += state.vy * dt;

  // --- Update stats ---
  state.distance = Math.max(state.distance, state.px);
  state.maxAltitude = Math.max(state.maxAltitude, state.py);

  // --- Check splash ---
  if (state.py <= 0) {
    state.py = 0;
    return 'SPLASH';
  }

  return 'FLIGHT';
}

function clamp(val, min, max) {
  return Math.max(min, Math.min(max, val));
}
```

---

## Ramp Run Mechanics

### Power Meter

```javascript
class PowerMeter {
  constructor(weightPenalty) {
    this.power = 0;           // 0 to 1
    this.tapTimes = [];       // timestamps of recent taps
    this.weightPenalty = weightPenalty; // 0.80 to 1.0
    this.decayRate = 0.3;     // power lost per second when not tapping
    this.maxTapWindow = 1.0;  // seconds to consider for tap frequency
  }

  tap(time) {
    this.tapTimes.push(time);
    // Keep only recent taps
    this.tapTimes = this.tapTimes.filter(t => time - t < this.maxTapWindow);

    // Calculate tap frequency (taps per second)
    const freq = this.tapTimes.length / this.maxTapWindow;

    // Add power based on frequency (6+ taps/sec = max gain)
    const gain = Math.min(freq / 6.0, 1.0) * 0.15;
    this.power = Math.min(1.0, this.power + gain);
  }

  update(dt) {
    // Decay power if not tapping
    this.power = Math.max(0, this.power - this.decayRate * dt);
  }

  getLaunchSpeed() {
    const BASE_MAX_SPEED = 15.0; // m/s
    return BASE_MAX_SPEED * this.power * this.weightPenalty;
  }

  isPerfect() {
    return this.power >= 0.9;
  }
}
```

### Launch Calculation

```javascript
function calculateLaunch(powerMeter, rampAngleDeg) {
  let speed = powerMeter.getLaunchSpeed();
  const perfect = powerMeter.isPerfect();

  // Perfect launch bonus
  if (perfect) {
    speed *= 1.10; // +10% speed
  }

  const angleRad = rampAngleDeg * (Math.PI / 180);
  return {
    vx: speed * Math.cos(angleRad),
    vy: speed * Math.sin(angleRad),
    perfect: perfect,
  };
}
```

---

## Pitch Controls

### Input Handling

```javascript
class PitchController {
  constructor() {
    this.pitchInput = 0; // -1 (nose down) to +1 (nose up)
    this.useAccelerometer = false;
    this._setupInputs();
  }

  _setupInputs() {
    // Keyboard
    const keys = {};
    window.addEventListener('keydown', e => { keys[e.code] = true; });
    window.addEventListener('keyup',   e => { keys[e.code] = false; });

    this._keys = keys;

    // Touch
    this._touchY = null;
    const canvas = document.querySelector('canvas');

    canvas.addEventListener('touchstart', e => {
      const touch = e.touches[0];
      const midY = window.innerHeight / 2;
      this._touchY = touch.clientY < midY ? 1 : -1;
    });

    canvas.addEventListener('touchend', () => {
      this._touchY = null;
    });

    // Accelerometer (optional, mobile)
    if (window.DeviceOrientationEvent) {
      window.addEventListener('deviceorientation', e => {
        if (this.useAccelerometer && e.beta !== null) {
          // beta: -180 to 180, tilt forward/back
          // Map -30 to +30 degrees of tilt to -1 to +1
          this._accelInput = clamp(e.beta / 30, -1, 1);
        }
      });
    }
  }

  update() {
    this.pitchInput = 0;

    // Keyboard
    if (this._keys['ArrowUp'] || this._keys['KeyW']) {
      this.pitchInput = 1;
    } else if (this._keys['ArrowDown'] || this._keys['KeyS']) {
      this.pitchInput = -1;
    }

    // Touch overrides keyboard
    if (this._touchY !== null) {
      this.pitchInput = this._touchY;
    }

    // Accelerometer overrides all if enabled
    if (this.useAccelerometer && this._accelInput !== undefined) {
      this.pitchInput = this._accelInput;
    }
  }

  getTargetPitch() {
    return this.pitchInput * PHYSICS.MAX_PITCH;
  }
}
```

---

## Camera System

```javascript
class FlightCamera {
  constructor(camera) {
    this.camera = camera;
    this.target = { x: 0, y: 10, z: 0 };
    this.offset = { x: -15, y: 5, z: 30 }; // side-scroll offset
    this.lerpSpeed = 3.0;
    this.phase = 'RAMP_RUN';
  }

  setPhase(phase) {
    this.phase = phase;
    switch (phase) {
      case 'RAMP_RUN':
        this.offset = { x: -5, y: 3, z: 15 };  // Close behind
        this.lerpSpeed = 2.0;
        break;
      case 'LAUNCH':
        this.offset = { x: -10, y: 5, z: 25 };  // Pull back
        this.lerpSpeed = 1.5;
        break;
      case 'FLIGHT':
        this.offset = { x: -15, y: 5, z: 30 };  // Wide side view
        this.lerpSpeed = 3.0;
        break;
      case 'SPLASH':
        this.offset = { x: -8, y: 3, z: 18 };   // Zoom to impact
        this.lerpSpeed = 4.0;
        break;
      case 'RESULTS':
        this.offset = { x: -20, y: 8, z: 35 };  // Pull way back
        this.lerpSpeed = 1.0;
        break;
    }
  }

  update(craftPos, dt) {
    // Target follows craft
    this.target.x = craftPos.x;
    this.target.y = Math.max(craftPos.y, 2); // Don't go below water

    // Smooth lerp to desired position
    const desiredX = this.target.x + this.offset.x;
    const desiredY = this.target.y + this.offset.y;
    const desiredZ = this.offset.z;

    const t = 1.0 - Math.exp(-this.lerpSpeed * dt);
    this.camera.position.x += (desiredX - this.camera.position.x) * t;
    this.camera.position.y += (desiredY - this.camera.position.y) * t;
    this.camera.position.z += (desiredZ - this.camera.position.z) * t;

    // Always look at craft
    this.camera.lookAt(this.target.x, this.target.y, 0);
  }
}
```

---

## Environmental Interactions

### Thermals

Rising columns of warm air that push the craft upward. Visible as shimmering
distortion and faint upward particle streams.

```javascript
class Thermal {
  constructor(x, radius, strength) {
    this.x = x;             // horizontal position (meters)
    this.radius = radius;   // meters (typically 5-15)
    this.strength = strength; // upward force in m/s^2 (typically 3-8)
    this.minY = 0;
    this.maxY = 30;         // thermals don't extend above 30m
  }

  getForce(craftX, craftY) {
    const dx = Math.abs(craftX - this.x);
    if (dx > this.radius) return 0;
    if (craftY < this.minY || craftY > this.maxY) return 0;

    // Stronger at center, falls off toward edge
    const falloff = 1.0 - (dx / this.radius);
    return this.strength * falloff * falloff;
  }
}
```

### Wind Gusts

Horizontal force bursts that push the craft forward or backward. Announced by
visual cues (leaves, papers blowing).

```javascript
class WindGust {
  constructor(startTime, duration, strength) {
    this.startTime = startTime;
    this.duration = duration;       // seconds (1-3)
    this.strength = strength;       // m/s^2 (positive = tailwind, negative = headwind)
  }

  getForce(currentTime) {
    const elapsed = currentTime - this.startTime;
    if (elapsed < 0 || elapsed > this.duration) return 0;

    // Ramp up, sustain, ramp down
    const t = elapsed / this.duration;
    const envelope = Math.sin(t * Math.PI); // smooth bell curve
    return this.strength * envelope;
  }
}
```

### Bird Flocks

Collision with birds causes speed loss and craft wobble.

```javascript
class BirdFlock {
  constructor(x, y) {
    this.x = x;
    this.y = y;
    this.radius = 3; // collision radius
    this.hit = false;
  }

  checkCollision(craftX, craftY) {
    if (this.hit) return false;
    const dx = craftX - this.x;
    const dy = craftY - this.y;
    if (Math.sqrt(dx*dx + dy*dy) < this.radius) {
      this.hit = true;
      return true;
    }
    return false;
  }

  applyEffect(state) {
    // Reduce speed by 20%
    state.vx *= 0.80;
    state.vy *= 0.80;
    // Apply random wobble to pitch
    state.pitch += (Math.random() - 0.5) * 15;
  }
}
```

### Altitude Zones

| Zone | Altitude  | Effect                                          |
|------|-----------|-------------------------------------------------|
| Low  | 0 - 8m   | Ground effect bonus (extra lift near water)     |
| Mid  | 8 - 25m  | Best thermals, normal flight                    |
| High | 25m+     | Reduced lift (thin air), no thermals            |

---

## Splash and Water Landing

### Splash Detection and Effects

```javascript
function triggerSplash(state, scene) {
  const impactSpeed = Math.abs(state.vy);
  const splashScale = clamp(impactSpeed / 20, 0.3, 1.0);

  // Lock distance
  const finalDistance = state.distance;

  // Create splash particles
  const splashCount = Math.floor(20 + splashScale * 40);
  for (let i = 0; i < splashCount; i++) {
    const angle = (Math.PI * 2 * i) / splashCount + (Math.random() - 0.5) * 0.3;
    const speed = (3 + Math.random() * 8) * splashScale;
    createWaterDroplet(scene, state.px, 0, {
      vx: Math.cos(angle) * speed,
      vy: Math.abs(Math.sin(angle)) * speed * 1.5,
      size: 0.1 + Math.random() * 0.2 * splashScale,
      life: 0.5 + Math.random() * 1.0,
    });
  }

  // Create expanding splash ring
  createSplashRing(scene, state.px, 0, splashScale);

  // Camera shake proportional to impact
  triggerCameraShake(0.3 * splashScale, 0.5);

  return {
    distance: finalDistance,
    impactSpeed: impactSpeed,
    splashScale: splashScale,
  };
}

function createWaterDroplet(scene, x, y, params) {
  const geo = new THREE.SphereGeometry(params.size, 6, 6);
  const mat = new THREE.MeshToonMaterial({ color: 0x4488CC });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(x, y, 0);
  mesh.userData = {
    vx: params.vx,
    vy: params.vy,
    life: params.life,
    age: 0,
  };
  scene.add(mesh);
  return mesh;
}

function createSplashRing(scene, x, y, scale) {
  const geo = new THREE.TorusGeometry(0.5, 0.1, 8, 16);
  const mat = new THREE.MeshToonMaterial({
    color: 0xAADDFF,
    transparent: true,
    opacity: 0.8,
  });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(x, y + 0.1, 0);
  mesh.rotation.x = Math.PI / 2;
  mesh.userData = {
    expandRate: 5 * scale,
    fadeRate: 1.5,
    age: 0,
  };
  scene.add(mesh);
  return mesh;
}
```

### Craft Bobbing on Water

```javascript
function updateCraftBobbing(craftMesh, time) {
  craftMesh.position.y = Math.sin(time * 2) * 0.15;
  craftMesh.rotation.z = Math.sin(time * 1.5) * 0.05;
}
```

---

## HUD Layout

```
+---------------------------------------------------------------------+
|  [ALT]                                                    [BOOST]   |
|  |##|  25m                                               [====  ]   |
|  |##|                                                               |
|  |##|              DISTANCE: 127.4m                                 |
|  |##|                                                               |
|  |  |                                                               |
|  |  |                                                               |
|  |  |  0m                                                           |
|                                                                     |
|           SPEED: 12.3 m/s          PITCH: +8.2 deg                  |
|                                                                     |
|  [Pitch Guide]                                                      |
|       ^  Nose Up                                                    |
|      --- Level                                                      |
|       v  Nose Down                                                  |
+---------------------------------------------------------------------+
```

### HUD Elements

```javascript
function createHUD() {
  const hud = document.createElement('div');
  hud.id = 'flight-hud';
  hud.innerHTML = `
    <div class="hud-altitude">
      <div class="alt-bar"><div class="alt-fill"></div></div>
      <span class="alt-label">0m</span>
    </div>
    <div class="hud-distance">0.0m</div>
    <div class="hud-speed">0.0 m/s</div>
    <div class="hud-pitch">
      <div class="pitch-indicator"></div>
    </div>
    <div class="hud-boost">
      <div class="boost-bar"><div class="boost-fill"></div></div>
    </div>
  `;
  document.body.appendChild(hud);
  return hud;
}

function updateHUD(hud, state) {
  const altFill = hud.querySelector('.alt-fill');
  const altLabel = hud.querySelector('.alt-label');
  const distLabel = hud.querySelector('.hud-distance');
  const speedLabel = hud.querySelector('.hud-speed');
  const pitchIndicator = hud.querySelector('.pitch-indicator');
  const boostFill = hud.querySelector('.boost-fill');

  const altPercent = clamp(state.py / 40, 0, 1) * 100;
  altFill.style.height = altPercent + '%';
  altLabel.textContent = state.py.toFixed(1) + 'm';

  distLabel.textContent = state.distance.toFixed(1) + 'm';

  const speed = Math.sqrt(state.vx * state.vx + state.vy * state.vy);
  speedLabel.textContent = speed.toFixed(1) + ' m/s';

  // Pitch indicator rotation
  pitchIndicator.style.transform = `rotate(${-state.pitch}deg)`;

  // Boost bar
  const boostPercent = (state.boostFuel / state.cp.boostFuel) * 100;
  boostFill.style.width = boostPercent + '%';
}
```

---

## Scoring System

### Primary Score

```
finalScore = distance * styleMultiplier + bonuses
```

### Bonuses

| Bonus             | Condition                          | Value          |
|-------------------|------------------------------------|----------------|
| Perfect Launch    | Power meter >= 90% at launch       | +50 points     |
| Thermal Chain x2  | Hit 2+ thermals in sequence        | +25 per chain  |
| Thermal Chain x3  | Hit 3+ thermals in sequence        | +50 per chain  |
| Low Rider         | Fly below 3m for 5+ seconds       | +30 points     |
| Sky High          | Reach altitude above 30m           | +40 points     |
| Smooth Landing    | Impact speed below 5 m/s           | +20 points     |

### Medal Thresholds

| Medal    | Score Requirement |
|----------|-------------------|
| None     | < 100             |
| Bronze   | 100 - 249         |
| Silver   | 250 - 499         |
| Gold     | 500 - 999         |
| Platinum | 1000+             |

### Score Calculation

```javascript
function calculateScore(state, bonuses) {
  const baseScore = state.distance;
  const styleMultiplier = state.cp.styleMultiplier;

  let bonusTotal = 0;
  const bonusList = [];

  if (bonuses.perfectLaunch) {
    bonusTotal += 50;
    bonusList.push({ name: 'Perfect Launch', value: 50 });
  }
  if (bonuses.thermalChain >= 3) {
    const val = 50 * (bonuses.thermalChain - 2);
    bonusTotal += val;
    bonusList.push({ name: `Thermal Chain x${bonuses.thermalChain}`, value: val });
  } else if (bonuses.thermalChain >= 2) {
    bonusTotal += 25;
    bonusList.push({ name: 'Thermal Chain x2', value: 25 });
  }
  if (bonuses.lowRiderTime >= 5.0) {
    bonusTotal += 30;
    bonusList.push({ name: 'Low Rider', value: 30 });
  }
  if (state.maxAltitude >= 30) {
    bonusTotal += 40;
    bonusList.push({ name: 'Sky High', value: 40 });
  }
  if (bonuses.impactSpeed < 5) {
    bonusTotal += 20;
    bonusList.push({ name: 'Smooth Landing', value: 20 });
  }

  const finalScore = Math.round(baseScore * styleMultiplier + bonusTotal);

  let medal = 'none';
  if (finalScore >= 1000) medal = 'platinum';
  else if (finalScore >= 500) medal = 'gold';
  else if (finalScore >= 250) medal = 'silver';
  else if (finalScore >= 100) medal = 'bronze';

  return { finalScore, baseScore, styleMultiplier, bonusTotal, bonusList, medal };
}
```

---

## Implementation Checklist

- [ ] HTML boilerplate with Three.js r128 CDN import
- [ ] Toon shader materials (MeshToonMaterial with gradient maps)
- [ ] Scene setup: renderer, camera, lights, sky gradient
- [ ] Water plane with animated vertex shader
- [ ] Pier/ramp geometry (procedural wooden structure)
- [ ] Crowd spectators (simple capsule meshes with wave animation)
- [ ] Craft loading from `window.storage` or fallback procedural craft
- [ ] Power meter UI (HTML overlay for ramp phase)
- [ ] Tap/click tracking for power accumulation
- [ ] Launch velocity calculation with perfect bonus
- [ ] Flight physics engine (lift, drag, gravity, AoA)
- [ ] Pitch control system (keyboard, touch, accelerometer)
- [ ] Camera system with per-phase behavior and smooth lerp
- [ ] Thermal system (spawning, visual indicators, force application)
- [ ] Wind gust system (random events, visual cues)
- [ ] Bird flock system (spawning, collision, speed penalty)
- [ ] Distance markers / buoys on water
- [ ] Cloud layer (parallax scrolling)
- [ ] HUD overlay (altitude bar, distance counter, speed, pitch, boost)
- [ ] Splash particle effects (droplets, rings, spray)
- [ ] Craft water bobbing animation
- [ ] Score calculation with bonuses and medal assignment
- [ ] Results screen (animated score reveal, medal, retry button)
- [ ] Object pooling for particles, clouds, environment elements
- [ ] Mobile-responsive layout and touch controls
- [ ] Sound effects hooks (crowd cheer, wind, splash, boost)
- [ ] Performance: maintain 60fps target, LOD for distant objects

---

## Test Prompts

1. **Basic flight test**: "Create a Flugtag flight game where I tap to build speed on a ramp, launch off, and glide over water. Use Three.js with toon shading. Show distance traveled."

2. **Physics tuning test**: "Build the Flugtag game but make the flight feel floaty and fun -- not realistic. The craft should glide gracefully and respond smoothly to pitch controls."

3. **Full feature test**: "Build the complete Flugtag flight engine with ramp run power meter, launch mechanics, pitch-controlled gliding with thermals and wind gusts, splash landing with particles, and score with bonuses."

4. **Mobile test**: "Create a mobile-friendly Flugtag game where I tap to build ramp speed, then tilt my phone to control pitch during flight. Touch-friendly HUD."

5. **Integration test**: "Build the Flugtag flight engine that reads craft stats from window.storage. Lift affects glide distance, weight affects ramp speed, style affects score multiplier."

6. **Visual polish test**: "Create the Flugtag flight scene with a cartoon pier, animated water, parallax clouds, distance buoys, and a dramatic splash effect when landing."

7. **Scoring test**: "Build the Flugtag game with full scoring: distance times style multiplier, plus bonuses for perfect launch, thermal chains, low flying, high altitude, and smooth landing. Show medals."
