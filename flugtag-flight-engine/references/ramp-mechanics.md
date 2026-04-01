# Ramp Mechanics Reference

Complete reference for the ramp run phase of the Flugtag Flight Engine. Covers
the power meter system, speed calculation, launch trajectory, ramp types,
visual feedback, character animation, and the full RampManager class.

---

## Table of Contents

1. [Ramp Run Overview](#ramp-run-overview)
2. [Power Meter System](#power-meter-system)
3. [Speed Calculation](#speed-calculation)
4. [Launch Trajectory](#launch-trajectory)
5. [Ramp Types per Course](#ramp-types-per-course)
6. [Visual Feedback During Ramp Run](#visual-feedback-during-ramp-run)
7. [Character Animation](#character-animation)
8. [Complete RampManager Class](#complete-rampmanager-class)
9. [Tuning Guide for Ramp Feel](#tuning-guide-for-ramp-feel)

---

## Ramp Run Overview

The ramp run is a side-view scene where the player's team pushes the craft
down a ramp toward the edge overlooking water. The player rapidly taps or
clicks to build speed on a power meter.

```
  Side View:
                       TAP! TAP! TAP!
                          |
                          v
  +--------+         +-------+
  | CROWD  |   Team->| CRAFT |--\
  | CROWD  |   push  +-------+   \   Ramp surface
  | CROWD  |    >>>               \\
  +--------+                       \\
  |  PIER  |========================\\
  |  PIER  |  Platform                \
  |  PIER  |                           \______ EDGE
  |  PIER  |                                   |
  +---------+                                  |
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~WATER~~~~~
              <--- ramp length --->
```

**Sequence of events**:

1. Craft positioned at top of ramp
2. "TAP TO BUILD SPEED!" prompt appears
3. Player taps rapidly -- power meter fills
4. Craft rolls forward, team characters push alongside
5. At ramp edge: launch velocity calculated from final power
6. Transition to LAUNCH state

---

## Power Meter System

The power meter is the core interaction of the ramp phase. It rewards rapid,
sustained tapping.

### Tap Frequency Tracking

The system tracks timestamps of recent taps within a rolling window. Tap
frequency (taps per second) determines how fast power accumulates.

```javascript
class PowerMeter {
  constructor(weightPenalty) {
    this.power = 0;              // 0.0 to 1.0 (percentage full)
    this.tapTimes = [];          // Array of timestamps (seconds)
    this.weightPenalty = weightPenalty; // 0.80 to 1.00 from craft weight stat
    this.maxTapWindow = 1.0;     // Rolling window for frequency calc (seconds)
    this.decayRate = 0.3;        // Power lost per second when not tapping
    this.gainPerTap = 0.04;      // Base power gain per qualified tap
    this.perfectThreshold = 0.9; // Power level needed for "perfect" launch
    this.lastTapTime = 0;        // Timestamp of most recent tap
    this.minTapInterval = 0.05;  // Minimum 50ms between taps (debounce)
  }
```

### Power Accumulation Formula

Each tap adds power based on the current tap frequency. Higher frequency
taps add more power, encouraging sustained rapid input.

```javascript
  /**
   * Register a tap. Called from input event handlers.
   * @param {number} time - Current time in seconds
   */
  tap(time) {
    // Debounce: ignore taps that are too close together
    if (time - this.lastTapTime < this.minTapInterval) return;
    this.lastTapTime = time;

    // Add to recent taps
    this.tapTimes.push(time);

    // Remove taps outside the window
    this.tapTimes = this.tapTimes.filter(t => time - t < this.maxTapWindow);

    // Calculate tap frequency (taps per second)
    const frequency = this.tapTimes.length / this.maxTapWindow;

    // Power gain scales with frequency:
    //   1 tap/sec  -> gain = 0.04 * (1/6)  = 0.007 (slow)
    //   3 taps/sec -> gain = 0.04 * (3/6)  = 0.020 (moderate)
    //   6 taps/sec -> gain = 0.04 * (6/6)  = 0.040 (maximum)
    //   9 taps/sec -> gain = 0.04 * (9/6)  = 0.060 (bonus for fast tappers)
    //   Capped at 10 taps/sec = 0.067
    const frequencyMultiplier = Math.min(frequency / 6.0, 10.0 / 6.0);
    const gain = this.gainPerTap * frequencyMultiplier;

    this.power = Math.min(1.0, this.power + gain);
  }
```

### Power Decay When Not Tapping

Power decays linearly when the player stops tapping. This creates urgency
and prevents the player from reaching max power with slow, spaced taps.

```javascript
  /**
   * Called every frame to apply decay.
   * @param {number} dt - Delta time in seconds
   */
  update(dt) {
    // Only decay if no recent taps
    const now = performance.now() / 1000;
    const timeSinceLastTap = now - this.lastTapTime;

    if (timeSinceLastTap > 0.2) {
      // Start decaying after 200ms gap
      this.power = Math.max(0, this.power - this.decayRate * dt);
    }
  }
```

### Perfect Zone (Timing Window for Bonus)

When power exceeds 90%, the meter enters the "perfect zone." If the craft
reaches the ramp edge while in this zone, a +10% launch speed bonus is applied.

```javascript
  /**
   * Check if currently in perfect zone.
   */
  isPerfect() {
    return this.power >= this.perfectThreshold;
  }

  /**
   * Get the current power as a display percentage (0-100).
   */
  getDisplayPercent() {
    return Math.round(this.power * 100);
  }
}
```

### Visual Power Meter (Segments)

The power meter is displayed as a segmented bar that fills from bottom to top.

```
  Power Meter Visual:

  100% +=========+ <-- PERFECT ZONE (gold)
       |#########|
   90% +---------+ <-- Perfect threshold
       |#########|
   80% |#########|
       |#########|
   70% |#########|
       |#########|  <-- Green zone (good)
   60% |#########|
       |#########|
   50% |#########|
       |   ###   |  <-- Yellow zone (moderate)
   40% |   ###   |
       |   ###   |
   30% |   ###   |
       |    #    |  <-- Red zone (weak)
   20% |    #    |
       |    #    |
   10% |    #    |
       |         |
    0% +=========+
```

```javascript
function createPowerMeterUI() {
  const container = document.createElement('div');
  container.id = 'power-meter';
  container.innerHTML = `
    <div class="pm-track">
      <div class="pm-fill"></div>
      <div class="pm-perfect-zone"></div>
      <div class="pm-segments">
        ${Array.from({length: 20}, (_, i) =>
          `<div class="pm-segment" data-index="${i}"></div>`
        ).join('')}
      </div>
    </div>
    <div class="pm-label">POWER</div>
    <div class="pm-percent">0%</div>
  `;

  // Styles
  const style = document.createElement('style');
  style.textContent = `
    #power-meter {
      position: fixed;
      left: 20px;
      bottom: 20%;
      width: 50px;
      height: 300px;
      z-index: 100;
    }
    .pm-track {
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.5);
      border-radius: 8px;
      overflow: hidden;
      position: relative;
      border: 2px solid #FFF;
    }
    .pm-fill {
      position: absolute;
      bottom: 0;
      width: 100%;
      height: 0%;
      transition: height 0.05s linear;
      border-radius: 0 0 6px 6px;
    }
    .pm-perfect-zone {
      position: absolute;
      top: 0;
      width: 100%;
      height: 10%;
      background: repeating-linear-gradient(
        45deg,
        rgba(255,215,0,0.3),
        rgba(255,215,0,0.3) 5px,
        rgba(255,215,0,0.1) 5px,
        rgba(255,215,0,0.1) 10px
      );
      border-bottom: 2px dashed gold;
    }
    .pm-label {
      text-align: center;
      color: white;
      font-weight: bold;
      font-size: 14px;
      margin-top: 5px;
      text-shadow: 1px 1px 2px black;
    }
    .pm-percent {
      text-align: center;
      color: white;
      font-size: 18px;
      font-weight: bold;
      text-shadow: 1px 1px 2px black;
    }
  `;
  document.head.appendChild(style);
  document.body.appendChild(container);
  return container;
}

function updatePowerMeterUI(power, isPerfect) {
  const fill = document.querySelector('.pm-fill');
  const percent = document.querySelector('.pm-percent');
  const track = document.querySelector('.pm-track');

  // Fill height
  fill.style.height = (power * 100) + '%';

  // Color based on power level
  if (isPerfect) {
    fill.style.background = 'linear-gradient(to top, #FFD700, #FFA500)';
    track.style.borderColor = '#FFD700';
    // Pulse animation for perfect zone
    fill.style.animation = 'perfectPulse 0.3s ease-in-out infinite';
  } else if (power > 0.6) {
    fill.style.background = 'linear-gradient(to top, #44CC44, #88FF88)';
    fill.style.animation = 'none';
    track.style.borderColor = '#FFF';
  } else if (power > 0.3) {
    fill.style.background = 'linear-gradient(to top, #CCCC44, #FFFF88)';
    fill.style.animation = 'none';
    track.style.borderColor = '#FFF';
  } else {
    fill.style.background = 'linear-gradient(to top, #CC4444, #FF8888)';
    fill.style.animation = 'none';
    track.style.borderColor = '#FFF';
  }

  percent.textContent = Math.round(power * 100) + '%';
}
```

---

## Speed Calculation

### Base Max Speed from Ramp Length

The ramp's physical length determines the maximum achievable speed. Longer
ramps allow more time to build speed.

```javascript
const BASE_MAX_SPEED = 15.0; // m/s -- maximum ramp exit speed
```

### Weight Penalty Formula

Heavier craft accelerate more slowly on the ramp, resulting in a lower
maximum speed.

```
rampSpeedPenalty = 1.0 - (weight - 1) * 0.05
```

| Weight Stat | Penalty Factor | Max Achievable Speed |
|-------------|----------------|----------------------|
| 1           | 1.00           | 15.0 m/s             |
| 2           | 0.95           | 14.25 m/s            |
| 3           | 0.90           | 13.5 m/s             |
| 4           | 0.85           | 12.75 m/s            |
| 5           | 0.80           | 12.0 m/s             |

### Final Launch Speed

```
launchSpeed = BASE_MAX_SPEED * powerPercent * rampSpeedPenalty
```

```javascript
function getLaunchSpeed(powerMeter) {
  return BASE_MAX_SPEED * powerMeter.power * powerMeter.weightPenalty;
}
```

**Example**: Weight-3 craft at 80% power:
```
launchSpeed = 15.0 * 0.80 * 0.90 = 10.8 m/s
```

---

## Launch Trajectory

When the craft reaches the ramp edge, its initial flight velocity is
decomposed into horizontal and vertical components based on the ramp angle.

### Velocity Components

```
initialVx = launchSpeed * cos(rampAngle)
initialVy = launchSpeed * sin(rampAngle)
```

```
                      /|
                     / |
    launchSpeed     /  |  Vy = launchSpeed * sin(angle)
                   /   |
                  / a  |
                 /_____|
                   Vx = launchSpeed * cos(angle)

    where a = ramp angle (degrees)
```

### Perfect Launch Bonus

If the power meter is in the perfect zone (>= 90%) at the moment the craft
reaches the ramp edge, launch speed gets a 10% bonus.

```javascript
function calculateLaunch(powerMeter, rampAngleDeg) {
  let speed = getLaunchSpeed(powerMeter);
  const perfect = powerMeter.isPerfect();

  if (perfect) {
    speed *= 1.10; // +10% bonus
  }

  const angleRad = rampAngleDeg * (Math.PI / 180);

  return {
    vx: speed * Math.cos(angleRad),
    vy: speed * Math.sin(angleRad),
    speed: speed,
    perfect: perfect,
    powerPercent: powerMeter.power,
  };
}
```

### Launch Examples

| Scenario                        | Speed  | Angle | Vx     | Vy    |
|---------------------------------|--------|-------|--------|-------|
| Weight-1, 100% power, perfect   | 16.5   | 15deg | 15.94  | 4.27  |
| Weight-3, 80% power             | 10.8   | 15deg | 10.43  | 2.80  |
| Weight-3, 100% power, perfect   | 14.85  | 15deg | 14.34  | 3.84  |
| Weight-5, 60% power             | 7.2    | 15deg | 6.96   | 1.86  |
| Weight-3, 100% power, steep ramp| 13.5   | 20deg | 12.69  | 4.62  |

---

## Ramp Types per Course

Different courses feature different ramp configurations. The ramp type affects
height (starting altitude), length (time to build speed), and angle (launch
trajectory).

| Course   | Height | Length | Angle | Special Feature          | Difficulty |
|----------|--------|--------|-------|--------------------------|------------|
| Pier     | 10m    | 15m    | 12deg | Standard, wooden pier    | Easy       |
| Cliff    | 20m    | 10m    | 20deg | Steep and short, rocky   | Medium     |
| Rooftop  | 15m    | 12m    | 15deg | Urban backdrop, neon     | Medium     |
| Mountain | 30m    | 20m    | 18deg | Extreme height, snow     | Hard       |

### Ramp Configuration Objects

```javascript
const RAMP_CONFIGS = {
  pier: {
    name: 'Seaside Pier',
    height: 10,         // meters above water
    length: 15,         // ramp length in meters
    angle: 12,          // degrees
    platformWidth: 8,   // width of the launching platform
    material: 'wood',   // visual material style
    crowdSize: 30,      // number of spectator meshes
    backdrop: 'beach',  // background scene
    music: 'upbeat',    // audio theme
    description: 'Classic wooden pier. Good for beginners.',
  },

  cliff: {
    name: 'Cliff Edge',
    height: 20,
    length: 10,
    angle: 20,
    platformWidth: 6,
    material: 'stone',
    crowdSize: 20,
    backdrop: 'cliff_face',
    music: 'dramatic',
    description: 'Short steep ramp off a cliff. High launch, less speed.',
  },

  rooftop: {
    name: 'Downtown Rooftop',
    height: 15,
    length: 12,
    angle: 15,
    platformWidth: 7,
    material: 'metal',
    crowdSize: 25,
    backdrop: 'city_skyline',
    music: 'electronic',
    description: 'Urban launch pad. Balanced ramp with city views.',
  },

  mountain: {
    name: 'Mountain Summit',
    height: 30,
    length: 20,
    angle: 18,
    platformWidth: 10,
    material: 'wood_snow',
    crowdSize: 15,
    backdrop: 'snowy_peaks',
    music: 'epic',
    description: 'Extreme altitude. Long ramp, massive height advantage.',
  },
};
```

### Ramp Geometry Builder

```javascript
function buildRampGeometry(config) {
  const group = new THREE.Group();
  const angleRad = config.angle * (Math.PI / 180);

  // --- Ramp surface ---
  const rampWidth = 4;
  const rampGeo = new THREE.PlaneGeometry(config.length, rampWidth);
  const rampMat = new THREE.MeshToonMaterial({
    color: config.material === 'wood' ? 0x8B6914 :
           config.material === 'stone' ? 0x888888 :
           config.material === 'metal' ? 0xAAAAAA : 0x8B6914,
  });
  const rampMesh = new THREE.Mesh(rampGeo, rampMat);

  // Position: center of ramp, tilted at ramp angle
  const rampCenterX = -(config.length / 2) * Math.cos(angleRad);
  const rampCenterY = config.height + (config.length / 2) * Math.sin(angleRad);
  rampMesh.position.set(rampCenterX, rampCenterY, 0);
  rampMesh.rotation.z = -angleRad;
  group.add(rampMesh);

  // --- Support structure (stilts/pillars) ---
  const pillarCount = Math.ceil(config.length / 3);
  for (let i = 0; i < pillarCount; i++) {
    const t = i / (pillarCount - 1); // 0 to 1 along ramp
    const px = -(config.length * (1 - t)) * Math.cos(angleRad);
    const pillarHeight = config.height + (config.length * (1 - t)) * Math.sin(angleRad);

    const pillarGeo = new THREE.BoxGeometry(0.4, pillarHeight, 0.4);
    const pillarMat = new THREE.MeshToonMaterial({ color: 0x654321 });
    const pillar = new THREE.Mesh(pillarGeo, pillarMat);
    pillar.position.set(px, pillarHeight / 2, -1.5);
    group.add(pillar);

    // Mirror pillar on other side
    const pillar2 = pillar.clone();
    pillar2.position.z = 1.5;
    group.add(pillar2);
  }

  // --- Rails ---
  const railGeo = new THREE.CylinderGeometry(0.08, 0.08, config.length);
  const railMat = new THREE.MeshToonMaterial({ color: 0x444444 });

  const rail1 = new THREE.Mesh(railGeo, railMat);
  rail1.rotation.z = Math.PI / 2 - angleRad;
  rail1.position.set(rampCenterX, rampCenterY + 0.5, -rampWidth / 2 + 0.2);
  group.add(rail1);

  const rail2 = rail1.clone();
  rail2.position.z = rampWidth / 2 - 0.2;
  group.add(rail2);

  // --- Platform at top ---
  const platGeo = new THREE.BoxGeometry(config.platformWidth, 0.3, rampWidth + 2);
  const platMat = new THREE.MeshToonMaterial({ color: 0x8B6914 });
  const platform = new THREE.Mesh(platGeo, platMat);
  const platX = -(config.length) * Math.cos(angleRad) - config.platformWidth / 2;
  const platY = config.height + config.length * Math.sin(angleRad);
  platform.position.set(platX, platY, 0);
  group.add(platform);

  // --- Launch edge marker ---
  const edgeGeo = new THREE.BoxGeometry(0.2, 0.5, rampWidth);
  const edgeMat = new THREE.MeshToonMaterial({ color: 0xFF0000 });
  const edge = new THREE.Mesh(edgeGeo, edgeMat);
  edge.position.set(0, config.height + 0.25, 0);
  group.add(edge);

  return group;
}
```

### How Ramp Type Affects Gameplay

**Pier (Easy)**:
- Long ramp gives plenty of time to build speed
- Low angle = more horizontal velocity, less vertical
- Low height = less potential energy
- Best for learning the power meter

**Cliff (Medium)**:
- Short ramp means less time to build speed
- Steep angle = more vertical velocity
- High starting altitude = more potential energy to trade for distance
- Rewards quick tapping bursts

**Rooftop (Medium)**:
- Balanced ramp length and angle
- Medium height for decent starting energy
- Tests all-around skill

**Mountain (Hard)**:
- Very long ramp rewards sustained rapid tapping
- Extreme height gives massive potential energy
- Steeper angle means more initial climb
- Best total energy budget but harder to maintain tap frequency over longer ramp

---

## Visual Feedback During Ramp Run

### Speed Lines

As power increases, speed lines appear around the edges of the screen,
conveying acceleration.

```javascript
function updateSpeedLines(power) {
  const container = document.getElementById('speed-lines');
  if (!container) return;

  const lineCount = Math.floor(power * 10); // 0-10 lines
  const opacity = power * 0.6;

  container.style.opacity = opacity;

  // Show/hide individual speed line elements
  const lines = container.querySelectorAll('.speed-line');
  lines.forEach((line, i) => {
    line.style.display = i < lineCount ? 'block' : 'none';
    // Animate length based on power
    const length = 20 + power * 60; // 20-80% of screen
    line.style.width = length + '%';
  });
}

function createSpeedLinesUI() {
  const container = document.createElement('div');
  container.id = 'speed-lines';
  container.style.cssText = `
    position: fixed; top: 0; left: 0; width: 100%; height: 100%;
    pointer-events: none; z-index: 50; opacity: 0;
  `;

  for (let i = 0; i < 10; i++) {
    const line = document.createElement('div');
    line.className = 'speed-line';
    const yPercent = 10 + (i * 8); // Distributed vertically
    line.style.cssText = `
      position: absolute;
      right: 0;
      top: ${yPercent}%;
      width: 20%;
      height: 2px;
      background: linear-gradient(to left, rgba(255,255,255,0.8), transparent);
      display: none;
    `;
    container.appendChild(line);
  }

  document.body.appendChild(container);
  return container;
}
```

### Crowd Cheering

The crowd's animation and audio intensity scale with power level.

```javascript
function updateCrowdVolume(power) {
  // Volume: quiet murmur at 0%, roaring at 100%
  const volume = 0.1 + power * 0.9;

  // Animation speed: idle at 0%, frantic waving at 100%
  const animSpeed = 0.5 + power * 2.0;

  // Update crowd spectator meshes
  for (const spectator of game.crowdMeshes) {
    // Arm wave amplitude scales with power
    const waveAmplitude = 0.2 + power * 0.5;
    spectator.userData.waveAmplitude = waveAmplitude;
    spectator.userData.waveSpeed = animSpeed;
  }
}
```

### Camera Shake

Subtle camera shake when power exceeds 50%, increasing in intensity.

```javascript
// Called in ramp run update:
if (power > 0.5) {
  const intensity = (power - 0.5) * 0.04; // 0 to 0.02
  triggerCameraShake(intensity, 0.05); // Very short bursts
}
```

### Tap Feedback

Each tap creates a brief visual pulse on the power meter and optionally
a haptic vibration on mobile.

```javascript
function triggerTapFeedback() {
  // Visual: flash the power meter border
  const track = document.querySelector('.pm-track');
  track.style.boxShadow = '0 0 15px rgba(255,255,255,0.8)';
  setTimeout(() => {
    track.style.boxShadow = 'none';
  }, 80);

  // Visual: small "+" text that floats up
  const plus = document.createElement('div');
  plus.textContent = '+';
  plus.style.cssText = `
    position: fixed; left: 75px; bottom: 50%;
    color: #FFD700; font-size: 18px; font-weight: bold;
    pointer-events: none; z-index: 101;
    animation: tapFadeUp 0.4s ease-out forwards;
  `;
  document.body.appendChild(plus);
  setTimeout(() => plus.remove(), 400);

  // Haptic (mobile)
  if (navigator.vibrate) {
    navigator.vibrate(15);
  }
}
```

---

## Character Animation

### Team Characters

Three team members push the craft down the ramp. They are simple capsule-shaped
meshes with basic limb animation.

```javascript
function createTeamCharacters() {
  const team = [];

  for (let i = 0; i < 3; i++) {
    const character = new THREE.Group();

    // Body (capsule: cylinder + 2 spheres)
    const bodyGeo = new THREE.CylinderGeometry(0.25, 0.25, 0.8, 8);
    const bodyMat = new THREE.MeshToonMaterial({
      color: [0xFF4444, 0x4444FF, 0x44FF44][i],
    });
    const body = new THREE.Mesh(bodyGeo, bodyMat);
    body.position.y = 0.7;
    character.add(body);

    // Head (sphere)
    const headGeo = new THREE.SphereGeometry(0.2, 8, 8);
    const headMat = new THREE.MeshToonMaterial({ color: 0xFFCC99 });
    const head = new THREE.Mesh(headGeo, headMat);
    head.position.y = 1.3;
    character.add(head);

    // Arms (thin cylinders)
    const armGeo = new THREE.CylinderGeometry(0.06, 0.06, 0.5, 6);
    const armMat = new THREE.MeshToonMaterial({ color: bodyMat.color });

    const leftArm = new THREE.Mesh(armGeo, armMat);
    leftArm.position.set(-0.35, 0.9, 0);
    leftArm.rotation.z = 0.5; // Angled forward (pushing)
    character.add(leftArm);
    character.userData.leftArm = leftArm;

    const rightArm = new THREE.Mesh(armGeo, armMat);
    rightArm.position.set(0.35, 0.9, 0);
    rightArm.rotation.z = -0.5;
    character.add(rightArm);
    character.userData.rightArm = rightArm;

    // Legs (thin cylinders)
    const legGeo = new THREE.CylinderGeometry(0.07, 0.07, 0.5, 6);
    const legMat = new THREE.MeshToonMaterial({ color: 0x333366 });

    const leftLeg = new THREE.Mesh(legGeo, legMat);
    leftLeg.position.set(-0.12, 0.25, 0);
    character.add(leftLeg);
    character.userData.leftLeg = leftLeg;

    const rightLeg = new THREE.Mesh(legGeo, legMat);
    rightLeg.position.set(0.12, 0.25, 0);
    character.add(rightLeg);
    character.userData.rightLeg = rightLeg;

    // Position behind craft, staggered
    character.userData.offset = { x: -2 - i * 0.8, z: (i - 1) * 1.2 };
    character.userData.runPhase = i * (Math.PI * 2 / 3); // Offset run cycle

    team.push(character);
  }

  return team;
}

/**
 * Update team character positions and animations during ramp run.
 */
function updateTeamCharacters(rampProgress, speed) {
  const rampAngleRad = game.rampConfig.angle * (Math.PI / 180);
  const time = game.gameTime;

  for (const char of game.teamCharacters) {
    // Position on ramp (behind craft)
    const dx = rampProgress * game.rampConfig.length + char.userData.offset.x;
    const clampedDx = Math.max(0, dx);
    char.position.x = -game.rampConfig.length + clampedDx * Math.cos(rampAngleRad);
    char.position.y = game.rampConfig.height
                    + (game.rampConfig.length - clampedDx) * Math.sin(rampAngleRad);
    char.position.z = char.userData.offset.z;

    // Running animation (leg swing)
    const runSpeed = speed * 0.8; // Slightly slower than craft
    const phase = char.userData.runPhase;
    const legSwing = Math.sin(time * runSpeed * 2 + phase) * 0.4;

    char.userData.leftLeg.rotation.x = legSwing;
    char.userData.rightLeg.rotation.x = -legSwing;

    // Arm animation (pushing forward)
    const armPush = Math.sin(time * runSpeed * 2 + phase + Math.PI / 2) * 0.3;
    char.userData.leftArm.rotation.x = -0.8 + armPush; // Mostly forward
    char.userData.rightArm.rotation.x = -0.8 - armPush;

    // Body lean forward while pushing
    char.rotation.x = -0.15; // Lean forward

    // At ramp edge: stop and wave
    if (rampProgress >= 0.9) {
      const wavePhase = (rampProgress - 0.9) / 0.1; // 0 to 1
      // Transition from push to wave
      char.userData.leftArm.rotation.z = 0.5 + wavePhase * 2.0;
      char.userData.leftArm.rotation.x = Math.sin(time * 5) * 0.5 * wavePhase;
      char.rotation.x = -0.15 * (1 - wavePhase); // Stand upright
    }
  }
}
```

### Team Wave at Launch

When the craft leaves the ramp, team characters stop running and wave goodbye.

```javascript
function triggerTeamWave() {
  for (const char of game.teamCharacters) {
    // Both arms up, waving
    char.userData.leftArm.rotation.z = 2.5;
    char.userData.rightArm.rotation.z = -2.5;

    // Animate wave
    const waveAnim = () => {
      const t = game.gameTime;
      char.userData.leftArm.rotation.x = Math.sin(t * 6) * 0.4;
      char.userData.rightArm.rotation.x = Math.sin(t * 6 + Math.PI) * 0.4;
    };
    char.userData.waveCallback = waveAnim;
  }
}
```

### Crowd Spectators

Simple capsule meshes arranged behind a barrier on the pier platform.

```javascript
function createCrowdSpectators(count) {
  const crowd = [];
  const spreadX = 8; // Width of crowd area
  const spreadZ = 4; // Depth

  for (let i = 0; i < count; i++) {
    const spectator = new THREE.Group();

    // Body
    const bodyGeo = new THREE.CylinderGeometry(0.15, 0.15, 0.6, 6);
    const hue = Math.random();
    const bodyColor = new THREE.Color().setHSL(hue, 0.7, 0.5);
    const bodyMat = new THREE.MeshToonMaterial({ color: bodyColor });
    const body = new THREE.Mesh(bodyGeo, bodyMat);
    body.position.y = 0.5;
    spectator.add(body);

    // Head
    const headGeo = new THREE.SphereGeometry(0.12, 6, 6);
    const headMat = new THREE.MeshToonMaterial({ color: 0xFFCC99 });
    const head = new THREE.Mesh(headGeo, headMat);
    head.position.y = 0.95;
    spectator.add(head);

    // One waving arm
    const armGeo = new THREE.CylinderGeometry(0.04, 0.04, 0.35, 4);
    const arm = new THREE.Mesh(armGeo, bodyMat);
    arm.position.set(0.2, 0.8, 0);
    spectator.add(arm);
    spectator.userData.arm = arm;

    // Random position in crowd area
    const rampEndX = -(game.rampConfig?.length || 15);
    spectator.position.set(
      rampEndX - 3 - Math.random() * spreadX,
      game.rampConfig?.height || 10,
      -spreadZ / 2 + Math.random() * spreadZ
    );

    // Animation properties
    spectator.userData.waveAmplitude = 0.3;
    spectator.userData.waveSpeed = 1.0;
    spectator.userData.wavePhase = Math.random() * Math.PI * 2;

    crowd.push(spectator);
  }

  return crowd;
}

function updateCrowdAnimation(dt, gameTime) {
  for (const spectator of game.crowdMeshes) {
    const { arm } = spectator.userData;
    if (!arm) continue;

    const wave = Math.sin(
      gameTime * spectator.userData.waveSpeed + spectator.userData.wavePhase
    ) * spectator.userData.waveAmplitude;

    arm.rotation.z = 1.5 + wave; // Waving above head
    arm.rotation.x = wave * 0.5;
  }
}
```

---

## Complete RampManager Class

Encapsulates all ramp phase logic in a single class.

```javascript
class RampManager {
  constructor(rampConfig, craftPhysics) {
    this.config = rampConfig;
    this.craftPhysics = craftPhysics;

    // Power meter
    this.powerMeter = new PowerMeter(craftPhysics.rampSpeedPenalty);

    // Ramp progress (0 = top, 1 = edge)
    this.progress = 0;
    this.currentSpeed = 0;

    // Timing
    this.elapsed = 0;

    // Visual elements (set externally after creation)
    this.craftMesh = null;
    this.teamCharacters = null;
    this.crowdMeshes = null;
    this.rampGroup = null;
    this.powerMeterUI = null;
    this.speedLinesUI = null;

    // State
    this.launched = false;
    this.tapHandlerBound = false;

    // Callbacks
    this.onLaunch = null; // Called when craft reaches ramp edge
  }

  /**
   * Initialize visual elements and input handlers.
   * Call once after setting mesh references.
   */
  init(scene) {
    // Build ramp geometry
    this.rampGroup = buildRampGeometry(this.config);
    scene.add(this.rampGroup);

    // Create team characters
    this.teamCharacters = createTeamCharacters();
    for (const char of this.teamCharacters) {
      scene.add(char);
    }

    // Create crowd
    this.crowdMeshes = createCrowdSpectators(this.config.crowdSize);
    for (const spec of this.crowdMeshes) {
      scene.add(spec);
    }

    // Create UI
    this.powerMeterUI = createPowerMeterUI();
    this.speedLinesUI = createSpeedLinesUI();

    // Position craft at ramp top
    this._positionCraftOnRamp(0);

    // Bind input
    this._bindTapInput();
  }

  /**
   * Register tap/click input handlers.
   */
  _bindTapInput() {
    this._onTap = (e) => {
      if (this.launched) return;
      e.preventDefault();
      this.powerMeter.tap(performance.now() / 1000);
      triggerTapFeedback();
    };

    this._onKeyTap = (e) => {
      if (this.launched) return;
      if (e.code === 'Space' || e.code === 'ArrowUp' || e.code === 'KeyW') {
        e.preventDefault();
        this.powerMeter.tap(performance.now() / 1000);
        triggerTapFeedback();
      }
    };

    window.addEventListener('mousedown', this._onTap);
    window.addEventListener('touchstart', this._onTap, { passive: false });
    window.addEventListener('keydown', this._onKeyTap);
    this.tapHandlerBound = true;
  }

  /**
   * Remove input handlers.
   */
  _unbindTapInput() {
    if (!this.tapHandlerBound) return;
    window.removeEventListener('mousedown', this._onTap);
    window.removeEventListener('touchstart', this._onTap);
    window.removeEventListener('keydown', this._onKeyTap);
    this.tapHandlerBound = false;
  }

  /**
   * Position the craft mesh on the ramp at a given progress (0-1).
   */
  _positionCraftOnRamp(progress) {
    if (!this.craftMesh) return;
    const angleRad = this.config.angle * (Math.PI / 180);
    const dx = progress * this.config.length;

    this.craftMesh.position.x = -this.config.length + dx * Math.cos(angleRad);
    this.craftMesh.position.y = this.config.height
                              + (this.config.length - dx) * Math.sin(angleRad);
    this.craftMesh.rotation.z = -angleRad;
  }

  /**
   * Per-frame update during ramp run.
   * @param {number} dt - Delta time in seconds
   * @param {number} gameTime - Total elapsed game time
   * @returns {object|null} Launch data if craft reached edge, null otherwise
   */
  update(dt, gameTime) {
    if (this.launched) return null;

    this.elapsed += dt;

    // Update power meter decay
    this.powerMeter.update(dt);

    // Calculate speed from power
    this.currentSpeed = this.powerMeter.getLaunchSpeed();

    // Advance along ramp
    if (this.currentSpeed > 0.5) {
      this.progress += (this.currentSpeed / this.config.length) * dt;
    }

    // Clamp progress
    this.progress = Math.min(this.progress, 1.0);

    // Position craft on ramp
    this._positionCraftOnRamp(this.progress);

    // Animate team characters
    if (this.teamCharacters) {
      updateTeamCharacters(this.progress, this.currentSpeed);
    }

    // Crowd animation and volume
    if (this.crowdMeshes) {
      updateCrowdVolume(this.powerMeter.power);
      updateCrowdAnimation(dt, gameTime);
    }

    // Speed lines
    updateSpeedLines(this.powerMeter.power);

    // Camera shake at high power
    if (this.powerMeter.power > 0.5) {
      const intensity = (this.powerMeter.power - 0.5) * 0.04;
      triggerCameraShake(intensity, 0.05);
    }

    // Update power meter UI
    updatePowerMeterUI(this.powerMeter.power, this.powerMeter.isPerfect());

    // Check launch condition
    if (this.progress >= 1.0) {
      return this._triggerLaunch();
    }

    return null;
  }

  /**
   * Calculate and return launch data.
   */
  _triggerLaunch() {
    this.launched = true;
    this._unbindTapInput();

    const launchData = calculateLaunch(this.powerMeter, this.config.angle);

    // Trigger team wave
    if (this.teamCharacters) {
      triggerTeamWave();
    }

    // Callback
    if (this.onLaunch) {
      this.onLaunch(launchData);
    }

    return launchData;
  }

  /**
   * Clean up UI and handlers.
   */
  dispose() {
    this._unbindTapInput();

    // Remove UI elements
    const pm = document.getElementById('power-meter');
    if (pm) pm.remove();
    const sl = document.getElementById('speed-lines');
    if (sl) sl.remove();
  }

  /**
   * Reset for a new ramp run (retry).
   */
  reset() {
    this.progress = 0;
    this.currentSpeed = 0;
    this.elapsed = 0;
    this.launched = false;
    this.powerMeter.power = 0;
    this.powerMeter.tapTimes = [];
    this._positionCraftOnRamp(0);

    if (!this.tapHandlerBound) {
      this._bindTapInput();
    }
  }
}
```

---

## Tuning Guide for Ramp Feel

### Achieving Satisfying Tap Feedback

The ramp run should feel punchy and responsive. Each tap should produce
visible, audible, and (on mobile) haptic feedback.

| Tunable               | Default | Range     | Effect                           |
|------------------------|---------|-----------|----------------------------------|
| `gainPerTap`           | 0.04    | 0.02-0.08 | How much power each tap adds    |
| `decayRate`            | 0.3     | 0.1-0.5   | How fast power drains           |
| `minTapInterval`       | 0.05s   | 0.03-0.1  | Debounce window                 |
| `maxTapWindow`         | 1.0s    | 0.5-2.0   | Rolling window for frequency    |
| `perfectThreshold`     | 0.9     | 0.8-0.95  | How hard it is to reach perfect |
| `BASE_MAX_SPEED`       | 15.0    | 10-20     | Maximum possible launch speed   |

### Tuning Process

1. **Set `decayRate` first**: This determines how urgently the player must tap.
   - Too low (0.1): player can reach max power with lazy tapping
   - Too high (0.5): impossible to maintain power, feels punishing
   - Sweet spot (0.3): requires sustained effort but is achievable

2. **Set `gainPerTap` to match**: Each tap should feel like it matters.
   - At 6 taps/sec, gain should outpace decay: `6 * gainPerTap > decayRate`
   - With defaults: `6 * 0.04 = 0.24`, decay = `0.3` -- close match
   - This means 6 taps/sec barely holds steady, 8+ taps/sec gains power

3. **Test perfect zone accessibility**:
   - Most players should reach 60-80% power
   - Good players reach 85-90% (near perfect)
   - Perfect launch should be achievable but challenging
   - If too easy: increase `perfectThreshold` to 0.95
   - If too hard: decrease `decayRate` to 0.25

4. **Test weight penalty**:
   - Weight-5 craft should feel noticeably sluggish on ramp
   - But not impossible -- minimum speed should still produce a fun flight
   - Weight-1 at 60% power should roughly equal Weight-5 at 100% power

5. **Test ramp duration**:
   - Should take 2-4 seconds to cross the ramp
   - Too fast: not enough time for power meter interaction
   - Too slow: boring and tiring
   - Adjust ramp length or `BASE_MAX_SPEED` to hit this window

### Common Ramp Feel Issues

**Power meter fills too fast**:
- Reduce `gainPerTap` (try 0.025)
- Increase `decayRate` (try 0.4)

**Power meter feels unresponsive**:
- Increase `gainPerTap` (try 0.06)
- Reduce `minTapInterval` (try 0.03)
- Ensure tap feedback is immediate (no animation delay)

**Ramp feels too short/long**:
- Adjust ramp `length` in the config
- Or adjust `BASE_MAX_SPEED` to change traverse time

**Perfect launch is too easy/hard**:
- Adjust `perfectThreshold` (0.85 for easier, 0.95 for harder)
- Adjust the decay rate to make maintaining high power harder

**Weight penalty doesn't feel impactful**:
- Increase the penalty: `1.0 - (weight - 1) * 0.08` instead of `0.05`
- Or add visible weight effects (craft bounces on ramp, team strains more)

**Mobile tapping feels worse than desktop clicking**:
- Reduce `minTapInterval` for touch events (0.03s)
- Add stronger haptic feedback
- Make the tap target the entire screen, not just a button
- Consider two-thumb alternating tap pattern for faster input
