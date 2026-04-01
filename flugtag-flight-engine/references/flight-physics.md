# Flight Physics Reference

Complete physics model for the Flugtag Flight Engine. This is a simplified 2D
arcade flight model tuned for fun, not realism. The goal is floaty, satisfying
gliding that rewards skillful pitch control.

---

## Table of Contents

1. [2D Flight Physics Model](#2d-flight-physics-model)
2. [Force Diagram](#force-diagram)
3. [Lift Equation](#lift-equation)
4. [Drag Equation](#drag-equation)
5. [Gravity](#gravity)
6. [Angle of Attack](#angle-of-attack)
7. [Stall Mechanics](#stall-mechanics)
8. [Terminal Velocity](#terminal-velocity)
9. [Craft Stats to Physics Constants](#craft-stats-to-physics-constants)
10. [Thermal Interaction Model](#thermal-interaction-model)
11. [Wind Gust Model](#wind-gust-model)
12. [Bird Collision Model](#bird-collision-model)
13. [Ground Effect Model](#ground-effect-model)
14. [Altitude-Based Air Density](#altitude-based-air-density)
15. [Energy Model](#energy-model)
16. [Complete FlightPhysics Class](#complete-flightphysics-class)
17. [Tuning Guide](#tuning-guide)
18. [Common Issues and Fixes](#common-issues-and-fixes)

---

## 2D Flight Physics Model

The game simulates flight in a 2D side-view plane (x = horizontal distance,
y = altitude). All forces are computed in this 2D space.

**Why simplified?** Real aerodynamics involves Reynolds numbers, induced drag,
aspect ratios, and dozens of coefficients. For an arcade game we need:

- Predictable behavior that players can learn
- A satisfying "float" feeling
- Clear cause-and-effect between pitch input and flight path
- Enough depth that skilled players fly farther

**Core principle**: The craft is always losing energy to drag. The player's job
is to manage pitch angle to trade altitude for distance efficiently. Thermals
and ground effect provide opportunities to recover energy.

**Coordinate system**:
- Origin: base of ramp at water level
- +x: horizontal distance from ramp (increases rightward)
- +y: altitude above water (increases upward)
- Angles: 0 degrees = horizontal right, positive = counter-clockwise (nose up)

---

## Force Diagram

```
                    LIFT (perpendicular to velocity)
                      ^
                     /
                    /
                   /
    - - - - - - -+=============> VELOCITY DIRECTION
                /  \  craft
               /    \
              /      v
          DRAG       GRAVITY (always straight down)
     (opposes        F = m * g
      velocity)

    Where:
      Angle of Attack (AoA) = pitch - velocity angle
      Lift acts PERPENDICULAR to velocity, rotated 90 degrees
      Drag acts OPPOSITE to velocity direction
```

The four forces acting on the craft at any moment:

1. **Gravity** -- always points straight down, constant magnitude
2. **Lift** -- perpendicular to velocity vector, magnitude depends on speed and AoA
3. **Drag** -- opposes velocity vector, magnitude depends on speed
4. **External** -- thermals (upward), wind (horizontal), boost (forward)

---

## Lift Equation

```
L = 0.5 * rho * v^2 * S * CL * sin(2 * AoA)
```

| Symbol | Description                     | Unit    |
|--------|---------------------------------|---------|
| L      | Lift force magnitude            | Newtons |
| rho    | Air density                     | kg/m^3  |
| v      | Airspeed (magnitude of velocity)| m/s     |
| S      | Wing area                       | m^2     |
| CL     | Lift coefficient (from stats)   | -       |
| AoA    | Angle of attack                 | radians |

**Why `sin(2 * AoA)` instead of linear CL?**

In real aviation, lift varies roughly linearly with AoA up to stall. We use
`sin(2 * AoA)` because it:

- Naturally peaks at 45 degrees AoA (unreachable due to stall)
- Drops to zero at 0 and 90 degrees (which makes intuitive sense)
- Creates a smooth, learnable curve
- Feels good in gameplay -- small pitch changes have proportional effect

```javascript
// Lift force calculation
function calculateLift(airDensity, speed, wingArea, liftCoeff, aoaRad) {
  return 0.5 * airDensity * speed * speed * wingArea * liftCoeff * Math.sin(2 * aoaRad);
}
```

### Lift Direction

Lift acts perpendicular to the velocity vector, rotated 90 degrees counter-clockwise:

```javascript
const velocityAngle = Math.atan2(vy, vx); // radians
const liftAngle = velocityAngle + Math.PI / 2; // perpendicular

const liftFx = liftMagnitude * Math.cos(liftAngle);
const liftFy = liftMagnitude * Math.sin(liftAngle);
```

This means:
- When flying level (velocity horizontal), lift points straight up
- When diving (velocity angled down), lift has a forward component
- When climbing (velocity angled up), lift has a backward component

---

## Drag Equation

```
D = 0.5 * rho * v^2 * S * CD
```

| Symbol | Description                     | Unit    |
|--------|---------------------------------|---------|
| D      | Drag force magnitude            | Newtons |
| rho    | Air density                     | kg/m^3  |
| v      | Airspeed                        | m/s     |
| S      | Wing area                       | m^2     |
| CD     | Drag coefficient (from stats)   | -       |

Drag always opposes velocity:

```javascript
function calculateDrag(airDensity, speed, wingArea, dragCoeff, vx, vy) {
  const dragMagnitude = 0.5 * airDensity * speed * speed * wingArea * dragCoeff;

  // Oppose velocity direction
  if (speed < 0.01) return { fx: 0, fy: 0 };
  return {
    fx: -dragMagnitude * (vx / speed),
    fy: -dragMagnitude * (vy / speed),
  };
}
```

**Key insight**: Drag increases with the square of speed. Going twice as fast
means four times the drag. This naturally limits maximum speed and creates the
energy management challenge.

---

## Gravity

The simplest force -- constant downward pull.

```
Fg = m * g
```

| Symbol | Description      | Value   |
|--------|------------------|---------|
| m      | Craft mass       | kg      |
| g      | Gravity constant | -9.8 m/s^2 |

```javascript
const gravityForce = { fx: 0, fy: mass * GRAVITY }; // GRAVITY = -9.8
```

Gravity is the relentless enemy. Without lift, the craft falls. The entire
gameplay tension comes from generating enough lift to counteract gravity.

---

## Angle of Attack

Angle of Attack (AoA) is the difference between where the craft is pointed
(pitch) and where it is actually going (velocity direction).

```
AoA = pitch - velocityAngle
```

```
              pitch = 10 deg (where nose points)
             /
            /
    -------+==========> velocity direction = -5 deg
                         (where craft is actually going)

    AoA = 10 - (-5) = 15 degrees
```

```javascript
function calculateAoA(pitch, vx, vy) {
  const velocityAngleDeg = Math.atan2(vy, vx) * (180 / Math.PI);
  return pitch - velocityAngleDeg;
}
```

### AoA Effects

| AoA Range     | Effect                                     |
|---------------|--------------------------------------------|
| -10 to 0 deg  | Negative lift (nose down dive)             |
| 0 to 10 deg   | Moderate lift, low drag -- efficient glide |
| 10 to 20 deg  | Strong lift, increasing drag               |
| 20+ deg       | STALL -- lift drops dramatically            |

The sweet spot for efficient gliding is **5 to 12 degrees AoA**. Players learn
to maintain this zone for maximum distance.

---

## Stall Mechanics

When AoA exceeds the stall angle (20 degrees), airflow separates from the wing
and lift drops sharply.

```javascript
function applyStall(baseLiftCoeff, aoaDeg) {
  const stallAngle = 20; // degrees

  if (Math.abs(aoaDeg) <= stallAngle) {
    return baseLiftCoeff; // No stall, full lift
  }

  // Progressive lift loss beyond stall angle
  const excess = Math.abs(aoaDeg) - stallAngle;
  const stallFactor = Math.max(0.1, 1.0 - excess * 0.08);
  return baseLiftCoeff * stallFactor;
}
```

### Stall behavior

```
Lift Coefficient vs AoA:

CL ^
   |        /\
   |       /  \
   |      /    \  <-- stall point at 20 deg
   |     /      \
   |    /        \____
   |   /               \___ (reduced but not zero)
   |  /
   | /
   +----------------------------> AoA (degrees)
   0   5   10  15  20  25  30
```

**Game feel**: When stalled, the craft's nose drops, it loses altitude quickly,
and the player must pitch down to regain airspeed and exit the stall. This
creates tense moments and a learnable skill.

Visual cues during stall:
- Craft shakes/wobbles
- Warning indicator on HUD
- Wind trail particles become turbulent
- Pitch indicator turns red

---

## Terminal Velocity

Maximum speeds are capped to prevent physics explosions and maintain game feel.

```javascript
const TERMINAL = {
  VX_MAX:   40.0,  // m/s horizontal (forward)
  VX_MIN:   -5.0,  // m/s horizontal (backward -- shouldn't happen often)
  VY_MAX:   30.0,  // m/s vertical (upward)
  VY_MIN:  -30.0,  // m/s vertical (downward / falling)
};
```

**Analytical terminal velocity** (falling straight down, no lift):

```
v_terminal = sqrt(2 * m * g / (rho * S * CD))
```

For a typical craft (m=80kg, S=4m^2, CD=0.19):
```
v_terminal = sqrt(2 * 80 * 9.8 / (1.225 * 4 * 0.19))
           = sqrt(1568 / 0.931)
           = sqrt(1684)
           ~ 41 m/s
```

We cap at 30 m/s falling speed to keep it manageable.

---

## Craft Stats to Physics Constants

Complete mapping from the 1-5 stat scale to physics values.

| Stat   | Value | Lift Coeff | Mass (kg) | Wing Area (m^2) | Drag Coeff | Style Mult | Ramp Penalty | Boost Fuel (s) |
|--------|-------|------------|-----------|------------------|------------|------------|--------------|----------------|
| Lift 1 | Min   | 0.30       | --        | 3.0              | --         | --         | --           | --             |
| Lift 2 |       | 0.40       | --        | 3.5              | --         | --         | --           | --             |
| Lift 3 | Mid   | 0.50       | --        | 4.0              | --         | --         | --           | --             |
| Lift 4 |       | 0.60       | --        | 4.5              | --         | --         | --           | --             |
| Lift 5 | Max   | 0.70       | --        | 5.0              | --         | --         | --           | --             |
| Wt 1   | Min   | --         | 50        | --               | 0.15       | --         | 1.00         | 3.0            |
| Wt 2   |       | --         | 65        | --               | 0.17       | --         | 0.95         | 2.6            |
| Wt 3   | Mid   | --         | 80        | --               | 0.19       | --         | 0.90         | 2.2            |
| Wt 4   |       | --         | 95        | --               | 0.21       | --         | 0.85         | 1.8            |
| Wt 5   | Max   | --         | 110       | --               | 0.23       | --         | 0.80         | 1.4            |
| Sty 1  | Min   | --         | --        | --               | --         | 1.00       | --           | --             |
| Sty 2  |       | --         | --        | --               | --         | 1.25       | --           | --             |
| Sty 3  | Mid   | --         | --        | --               | --         | 1.50       | --           | --             |
| Sty 4  |       | --         | --        | --               | --         | 1.75       | --           | --             |
| Sty 5  | Max   | --         | --        | --               | --         | 2.00       | --           | --             |

### Formulas

```javascript
function statsToPhysics(lift, weight, style) {
  return {
    baseLiftCoeff:   0.3 + (lift - 1) * 0.1,
    mass:            50 + (weight - 1) * 15,
    wingArea:        3.0 + (lift - 1) * 0.5,
    dragCoeff:       0.15 + (weight - 1) * 0.02,
    styleMultiplier: 1.0 + (style - 1) * 0.25,
    rampSpeedPenalty: 1.0 - (weight - 1) * 0.05,
    boostFuel:       3.0 - (weight - 1) * 0.4,
  };
}
```

### Archetype Analysis

**The Glider** (Lift 5, Weight 1, Style 1):
- CL=0.70, mass=50kg, wingArea=5.0, CD=0.15
- Extremely floaty, great lift-to-drag ratio
- Fast on the ramp (no weight penalty)
- Low score multiplier -- pure distance build

**The Brick** (Lift 1, Weight 5, Style 1):
- CL=0.30, mass=110kg, wingArea=3.0, CD=0.23
- Barely generates lift, heavy drag
- Slow on ramp (20% penalty)
- Falls like a rock -- comedy build for short distances

**The Showboat** (Lift 1, Weight 1, Style 5):
- CL=0.30, mass=50kg, wingArea=3.0, CD=0.15
- Light but poor lift -- short flights
- 2.0x score multiplier compensates
- High risk, high reward if distance is decent

**The Balanced** (Lift 3, Weight 3, Style 3):
- CL=0.50, mass=80kg, wingArea=4.0, CD=0.19
- Solid all-around performer
- 1.5x multiplier, moderate flight distance
- Forgiving for new players

---

## Thermal Interaction Model

Thermals are invisible columns of rising warm air that apply an upward force
to the craft. They are the primary way to extend flight distance.

### Thermal Geometry

Each thermal is a vertical cylinder defined by:
- **x position**: horizontal center on the flight path
- **radius**: how wide the column is (5-15 meters)
- **strength**: upward acceleration at center (3-8 m/s^2)
- **minY / maxY**: vertical extent (typically 0-30m)

```
        maxY (30m)
         |       |
         | warm  |
         | air   |
         | rises |
         |   ^   |
         |   ^   |
         |   ^   |
        minY (0m)
    <-- radius -->
         center (x)
```

### Force Calculation

```javascript
class Thermal {
  constructor(x, radius, strength) {
    this.x = x;
    this.radius = radius;     // meters
    this.strength = strength; // m/s^2 upward acceleration
    this.minY = 0;
    this.maxY = 30;
  }

  /**
   * Returns upward acceleration (m/s^2) at the given position.
   * Falls off quadratically from center to edge.
   */
  getUpwardAccel(craftX, craftY) {
    // Outside horizontal bounds
    const dx = Math.abs(craftX - this.x);
    if (dx > this.radius) return 0;

    // Outside vertical bounds
    if (craftY < this.minY || craftY > this.maxY) return 0;

    // Quadratic falloff from center
    const normalizedDist = dx / this.radius; // 0 at center, 1 at edge
    const falloff = 1.0 - normalizedDist * normalizedDist;

    return this.strength * falloff;
  }

  /**
   * Apply thermal force to flight state.
   */
  applyToState(state, dt) {
    const accel = this.getUpwardAccel(state.px, state.py);
    if (accel > 0) {
      state.vy += accel * dt;
      return true; // Craft is in thermal
    }
    return false;
  }
}
```

### Thermal Spawning

Thermals are pre-placed along the flight path at semi-random intervals:

```javascript
function generateThermals(courseLength) {
  const thermals = [];
  let x = 30; // First thermal at 30m from ramp

  while (x < courseLength) {
    const radius = 5 + Math.random() * 10;   // 5-15m radius
    const strength = 3 + Math.random() * 5;   // 3-8 m/s^2

    thermals.push(new Thermal(x, radius, strength));

    // Next thermal 30-80m away
    x += 30 + Math.random() * 50;
  }

  return thermals;
}
```

### Visual Indicators

Thermals are partially visible to give skilled players an advantage:
- Faint upward-flowing particles (subtle white dots rising)
- Slight heat shimmer effect (vertex displacement on nearby geometry)
- Birds often circle near thermals (visual hint)

---

## Wind Gust Model

Wind gusts are time-limited horizontal forces that push the craft forward
(tailwind) or backward (headwind).

```javascript
class WindGust {
  constructor(startTime, duration, strength) {
    this.startTime = startTime;
    this.duration = duration;     // 1-3 seconds
    this.strength = strength;     // m/s^2, positive=tailwind, negative=headwind
    this.active = true;
  }

  /**
   * Returns horizontal acceleration at the given time.
   * Uses a sine envelope for smooth ramp-up and ramp-down.
   */
  getAcceleration(currentTime) {
    const elapsed = currentTime - this.startTime;
    if (elapsed < 0 || elapsed > this.duration) {
      this.active = false;
      return 0;
    }

    // Sine envelope: ramp up, peak, ramp down
    const t = elapsed / this.duration;
    const envelope = Math.sin(t * Math.PI);
    return this.strength * envelope;
  }

  applyToState(state, currentTime, dt) {
    const accel = this.getAcceleration(currentTime);
    state.vx += accel * dt;
  }
}
```

### Wind Gust Scheduling

```javascript
function scheduleWindGusts(flightDuration) {
  const gusts = [];
  let t = 2.0; // First gust after 2 seconds of flight

  while (t < flightDuration) {
    const duration = 1.0 + Math.random() * 2.0;

    // 60% chance tailwind, 40% chance headwind
    const isTailwind = Math.random() < 0.6;
    const strength = (isTailwind ? 1 : -1) * (2 + Math.random() * 4);

    gusts.push(new WindGust(t, duration, strength));
    t += 3 + Math.random() * 5; // 3-8 seconds between gusts
  }

  return gusts;
}
```

### Visual Cues

- Tailwind: leaves/papers blow from left to right across screen
- Headwind: debris blows from right to left
- Wind indicator arrow on HUD
- Cloud movement speed changes

---

## Bird Collision Model

Bird flocks are obstacles that reduce speed and cause pitch wobble on collision.

```javascript
class BirdFlock {
  constructor(x, y, birdCount) {
    this.x = x;
    this.y = y;
    this.radius = 3;              // collision radius in meters
    this.birdCount = birdCount || 5;
    this.hit = false;
    this.scattered = false;

    // Each bird is a small offset from center (for rendering)
    this.birds = [];
    for (let i = 0; i < this.birdCount; i++) {
      this.birds.push({
        offsetX: (Math.random() - 0.5) * this.radius * 2,
        offsetY: (Math.random() - 0.5) * this.radius,
        flapPhase: Math.random() * Math.PI * 2,
        flapSpeed: 3 + Math.random() * 2,
      });
    }
  }

  checkCollision(craftX, craftY) {
    if (this.hit) return false;
    const dx = craftX - this.x;
    const dy = craftY - this.y;
    const dist = Math.sqrt(dx * dx + dy * dy);
    return dist < this.radius;
  }

  /**
   * Apply collision effect to flight state.
   * Speed reduction: 20%
   * Pitch wobble: random +/- 15 degrees
   */
  applyCollision(state) {
    if (this.hit) return;
    this.hit = true;
    this.scattered = true;

    // Speed reduction
    state.vx *= 0.80;
    state.vy *= 0.80;

    // Random pitch wobble
    state.pitch += (Math.random() - 0.5) * 15;

    return {
      speedLoss: 0.20,
      cameraShake: { intensity: 0.2, duration: 0.3 },
      featherBurst: true, // trigger feather particle effect
    };
  }

  /**
   * Update bird positions (for rendering).
   * After scatter, birds fly away in random directions.
   */
  update(dt, time) {
    if (this.scattered) {
      // Birds scatter outward
      for (const bird of this.birds) {
        bird.offsetX += bird.offsetX * 3 * dt;
        bird.offsetY += bird.offsetY * 3 * dt;
      }
    } else {
      // Normal flocking -- gentle bobbing
      for (const bird of this.birds) {
        bird.offsetY += Math.sin(time * bird.flapSpeed + bird.flapPhase) * 0.5 * dt;
      }
    }
  }
}
```

---

## Ground Effect Model

When flying close to the water surface (below 3 meters altitude), the craft
receives a lift bonus due to ground effect. This encourages risky low flying.

```javascript
function getGroundEffectMultiplier(altitude) {
  const GROUND_EFFECT_HEIGHT = 3.0;
  const GROUND_EFFECT_BONUS = 1.5; // 50% more lift at water level

  if (altitude >= GROUND_EFFECT_HEIGHT || altitude <= 0) {
    return 1.0; // No ground effect
  }

  // Linear interpolation: strongest at altitude=0, fades to 1.0 at 3m
  const factor = 1.0 - (altitude / GROUND_EFFECT_HEIGHT);
  return 1.0 + (GROUND_EFFECT_BONUS - 1.0) * factor;
}
```

### Ground Effect Diagram

```
Altitude  | Ground Effect Multiplier
  3.0m    |  1.00x (no bonus)
  2.5m    |  1.08x
  2.0m    |  1.17x
  1.5m    |  1.25x
  1.0m    |  1.33x
  0.5m    |  1.42x
  0.0m    |  1.50x (maximum bonus)
```

**Game strategy**: Skilled players can "skim" the water surface to maintain flight
longer, but risk splashing prematurely if they dip too low. This creates an
exciting risk/reward dynamic.

Visual indicators:
- Water spray particles when below 2m
- Ripple on water surface below craft
- HUD "LOW" warning indicator

---

## Altitude-Based Air Density

Above 25 meters, the air is thinner (game abstraction), reducing lift.

```javascript
function getAirDensity(altitude) {
  const BASE_DENSITY = 1.225; // kg/m^3
  const HIGH_ALT_THRESHOLD = 25.0;
  const HIGH_ALT_PENALTY = 0.7; // 30% less lift above threshold

  if (altitude <= HIGH_ALT_THRESHOLD) {
    return BASE_DENSITY;
  }

  // Gradual reduction above threshold
  const excess = altitude - HIGH_ALT_THRESHOLD;
  const factor = Math.max(HIGH_ALT_PENALTY, 1.0 - excess * 0.01);
  return BASE_DENSITY * factor;
}
```

This discourages unrealistic climbing and keeps the gameplay in the
interesting altitude band of 0-25 meters where thermals and ground effect
create decision-making opportunities.

---

## Energy Model

Understanding the energy trade-offs helps with tuning and debugging.

### Kinetic Energy
```
KE = 0.5 * m * v^2
```

### Potential Energy
```
PE = m * g * h    (where g is positive 9.8 here)
```

### Total Mechanical Energy
```
E_total = KE + PE = 0.5 * m * v^2 + m * g * h
```

### Energy Budget

At launch:
```
E_launch = 0.5 * m * v_launch^2 + m * g * h_ramp
```

For a typical craft (m=80kg, v_launch=12 m/s, h_ramp=10m):
```
E_launch = 0.5 * 80 * 144 + 80 * 9.8 * 10
         = 5760 + 7840
         = 13600 Joules
```

Energy is continuously lost to drag. Thermals add energy. The flight ends
when all energy has been converted to heat (drag) and the craft is at y=0.

```javascript
function calculateEnergy(state) {
  const speed = Math.sqrt(state.vx * state.vx + state.vy * state.vy);
  const ke = 0.5 * state.cp.mass * speed * speed;
  const pe = state.cp.mass * 9.8 * Math.max(0, state.py);
  return { kinetic: ke, potential: pe, total: ke + pe };
}
```

This is useful for debugging: if total energy is increasing without thermals
or boost, there is a physics bug.

---

## Complete FlightPhysics Class

```javascript
class FlightPhysics {
  constructor(craftStats) {
    // Derive physics constants from craft stats
    const lift   = craftStats.lift   || 3;
    const weight = craftStats.weight || 3;
    const style  = craftStats.style  || 3;

    this.baseLiftCoeff   = 0.3 + (lift - 1) * 0.1;
    this.mass            = 50 + (weight - 1) * 15;
    this.wingArea        = 3.0 + (lift - 1) * 0.5;
    this.dragCoeff       = 0.15 + (weight - 1) * 0.02;
    this.styleMultiplier = 1.0 + (style - 1) * 0.25;
    this.rampSpeedPenalty = 1.0 - (weight - 1) * 0.05;
    this.boostFuel       = 3.0 - (weight - 1) * 0.4;

    // Constants
    this.GRAVITY            = -9.8;
    this.BASE_AIR_DENSITY   = 1.225;
    this.RAMP_HEIGHT        = 10.0;
    this.RAMP_ANGLE         = 15.0;
    this.MAX_PITCH          = 30.0;
    this.PITCH_SPEED        = 60.0;
    this.STALL_ANGLE        = 20.0;
    this.GROUND_EFFECT_H    = 3.0;
    this.GROUND_EFFECT_MULT = 1.5;
    this.HIGH_ALT_THRESH    = 25.0;
    this.HIGH_ALT_PENALTY   = 0.7;
    this.TERMINAL_VX        = 40.0;
    this.TERMINAL_VY_DOWN   = -30.0;
    this.TERMINAL_VY_UP     = 30.0;
    this.BOOST_FORCE        = 200; // Newtons

    // State
    this.px = 0;
    this.py = this.RAMP_HEIGHT;
    this.vx = 0;
    this.vy = 0;
    this.pitch = 0;
    this.targetPitch = 0;
    this.boostActive = false;
    this.currentBoostFuel = this.boostFuel;

    // Tracking
    this.maxAltitude = this.py;
    this.distance = 0;
    this.flightTime = 0;
    this.inGroundEffect = false;
    this.isStalling = false;
  }

  /**
   * Set initial launch velocity from ramp.
   */
  launch(launchSpeed) {
    const angleRad = this.RAMP_ANGLE * (Math.PI / 180);
    this.vx = launchSpeed * Math.cos(angleRad);
    this.vy = launchSpeed * Math.sin(angleRad);
    this.px = 0;
    this.py = this.RAMP_HEIGHT;
  }

  /**
   * Set target pitch from input (-1 to +1 mapped to -MAX_PITCH to +MAX_PITCH).
   */
  setTargetPitch(input) {
    this.targetPitch = input * this.MAX_PITCH;
  }

  /**
   * Main physics update. Call once per frame.
   * @param {number} dt - Delta time in seconds (capped externally)
   * @param {Thermal[]} thermals - Active thermal objects
   * @param {WindGust[]} gusts - Active wind gust objects
   * @param {number} gameTime - Current game time for gust evaluation
   * @returns {string} Current state: 'FLIGHT' or 'SPLASH'
   */
  update(dt, thermals = [], gusts = [], gameTime = 0) {
    this.flightTime += dt;

    // --- Pitch interpolation ---
    const pitchDiff = this.targetPitch - this.pitch;
    const maxChange = this.PITCH_SPEED * dt;
    this.pitch += Math.max(-maxChange, Math.min(maxChange, pitchDiff));
    this.pitch = this._clamp(this.pitch, -this.MAX_PITCH, this.MAX_PITCH);

    // --- Speed and angles ---
    const speed = Math.sqrt(this.vx * this.vx + this.vy * this.vy);
    const velocityAngleDeg = Math.atan2(this.vy, this.vx) * (180 / Math.PI);
    const aoa = this.pitch - velocityAngleDeg;
    const aoaRad = aoa * (Math.PI / 180);
    const velocityAngleRad = Math.atan2(this.vy, this.vx);

    // --- Air density at altitude ---
    const airDensity = this._getAirDensity(this.py);

    // --- Lift coefficient with modifiers ---
    let liftCoeff = this.baseLiftCoeff;

    // Stall
    this.isStalling = Math.abs(aoa) > this.STALL_ANGLE;
    if (this.isStalling) {
      const excess = Math.abs(aoa) - this.STALL_ANGLE;
      liftCoeff *= Math.max(0.1, 1.0 - excess * 0.08);
    }

    // Ground effect
    this.inGroundEffect = this.py > 0 && this.py < this.GROUND_EFFECT_H;
    if (this.inGroundEffect) {
      const factor = 1.0 + (this.GROUND_EFFECT_MULT - 1.0)
                         * (1.0 - this.py / this.GROUND_EFFECT_H);
      liftCoeff *= factor;
    }

    // --- Lift force ---
    const liftMag = 0.5 * airDensity * speed * speed
                  * this.wingArea * liftCoeff * Math.sin(2 * aoaRad);
    const liftAngleRad = velocityAngleRad + Math.PI / 2;
    const liftFx = liftMag * Math.cos(liftAngleRad);
    const liftFy = liftMag * Math.sin(liftAngleRad);

    // --- Drag force ---
    const dragMag = 0.5 * airDensity * speed * speed
                  * this.wingArea * this.dragCoeff;
    const dragFx = speed > 0.01 ? -dragMag * (this.vx / speed) : 0;
    const dragFy = speed > 0.01 ? -dragMag * (this.vy / speed) : 0;

    // --- Gravity ---
    const gravFy = this.mass * this.GRAVITY;

    // --- Boost ---
    let boostFx = 0;
    if (this.boostActive && this.currentBoostFuel > 0) {
      boostFx = this.BOOST_FORCE;
      this.currentBoostFuel = Math.max(0, this.currentBoostFuel - dt);
    }

    // --- Environmental forces ---
    let envFx = 0;
    let envFy = 0;

    // Thermals
    for (const thermal of thermals) {
      const upAccel = thermal.getUpwardAccel(this.px, this.py);
      envFy += upAccel * this.mass; // Convert acceleration to force
    }

    // Wind gusts
    for (const gust of gusts) {
      const gustAccel = gust.getAcceleration(gameTime);
      envFx += gustAccel * this.mass;
    }

    // --- Net force and acceleration ---
    const totalFx = liftFx + dragFx + boostFx + envFx;
    const totalFy = liftFy + dragFy + gravFy + envFy;
    const ax = totalFx / this.mass;
    const ay = totalFy / this.mass;

    // --- Integrate ---
    this.vx += ax * dt;
    this.vy += ay * dt;

    // Clamp velocities
    this.vx = this._clamp(this.vx, -5, this.TERMINAL_VX);
    this.vy = this._clamp(this.vy, this.TERMINAL_VY_DOWN, this.TERMINAL_VY_UP);

    this.px += this.vx * dt;
    this.py += this.vy * dt;

    // --- Update tracking ---
    this.distance = Math.max(this.distance, this.px);
    this.maxAltitude = Math.max(this.maxAltitude, this.py);

    // --- Splash check ---
    if (this.py <= 0) {
      this.py = 0;
      return 'SPLASH';
    }

    return 'FLIGHT';
  }

  /**
   * Get current speed magnitude.
   */
  getSpeed() {
    return Math.sqrt(this.vx * this.vx + this.vy * this.vy);
  }

  /**
   * Get current total energy (for debug).
   */
  getEnergy() {
    const speed = this.getSpeed();
    const ke = 0.5 * this.mass * speed * speed;
    const pe = this.mass * 9.8 * Math.max(0, this.py);
    return { kinetic: ke, potential: pe, total: ke + pe };
  }

  _getAirDensity(altitude) {
    if (altitude <= this.HIGH_ALT_THRESH) return this.BASE_AIR_DENSITY;
    const excess = altitude - this.HIGH_ALT_THRESH;
    const factor = Math.max(this.HIGH_ALT_PENALTY, 1.0 - excess * 0.01);
    return this.BASE_AIR_DENSITY * factor;
  }

  _clamp(val, min, max) {
    return Math.max(min, Math.min(max, val));
  }
}
```

---

## Tuning Guide

The physics model has many knobs. Here is how to adjust for game feel.

### Making Flights Feel Floaty and Fun

The default constants are tuned for arcade fun, not realism. If flights feel
too realistic (short, punishing), adjust these:

| Adjustment                         | Parameter           | Direction |
|------------------------------------|---------------------|-----------|
| Longer flights                     | `baseLiftCoeff`     | Increase  |
| Longer flights                     | `dragCoeff`         | Decrease  |
| More forgiving pitch               | `STALL_ANGLE`       | Increase  |
| Slower falling                     | `GRAVITY`           | Reduce magnitude (-7 instead of -9.8) |
| More responsive controls           | `PITCH_SPEED`       | Increase  |
| Bigger ground effect zone          | `GROUND_EFFECT_H`   | Increase  |
| Stronger ground effect             | `GROUND_EFFECT_MULT`| Increase  |
| More thermal boost                 | `Thermal.strength`  | Increase  |
| Longer thermal zones               | `Thermal.radius`    | Increase  |

### Target Flight Distances

For a well-tuned game, target these distances for a mid-stat craft (3/3/3):

| Player Skill | Expected Distance |
|--------------|-------------------|
| First try    | 30 - 60m          |
| Learning     | 60 - 120m         |
| Competent    | 120 - 250m        |
| Skilled      | 250 - 500m        |
| Expert       | 500m+             |

### Quick Tuning Process

1. Set all stats to 3/3/3
2. Launch at 70% power (not perfect)
3. Hold pitch at +5 degrees (gentle nose-up)
4. Should glide approximately 80-120m with no thermals
5. If too short: reduce `dragCoeff` or increase `baseLiftCoeff`
6. If too long: increase `dragCoeff`
7. Add thermals and test: skilled use should double distance
8. Test extreme builds (5/1/1 and 1/5/1)
9. 5/1/1 should fly 2-3x farther than 3/3/3
10. 1/5/1 should barely clear 30m

---

## Common Issues and Fixes

### Craft nosedives immediately after launch

**Cause**: Lift is too weak relative to gravity, or launch angle is too low.

**Fix**:
- Increase `baseLiftCoeff` globally
- Increase `RAMP_ANGLE` (try 18-20 degrees)
- Increase initial launch speed
- Check that `sin(2 * aoaRad)` is positive at launch -- the initial pitch
  should match the ramp angle

### Craft flies forever (never lands)

**Cause**: Lift overcomes gravity at equilibrium speed.

**Fix**:
- Increase `dragCoeff` to bleed speed faster
- Decrease `baseLiftCoeff`
- Ensure no accidental energy injection (check thermal forces, ground effect)
- Reduce `GROUND_EFFECT_MULT` if craft "surfs" the water surface indefinitely
- Add a maximum flight time failsafe (e.g., 60 seconds)

### Craft oscillates wildly (porpoising)

**Cause**: Pitch changes too fast, or lift response is too aggressive.

**Fix**:
- Reduce `PITCH_SPEED` for smoother pitch changes
- Add pitch damping (reduce pitch speed when oscillation is detected)
- Smooth the `sin(2 * aoaRad)` response curve
- Add a small artificial stability term that resists pitch changes

### Ground effect causes infinite surfing

**Cause**: At low altitude, ground effect lift exactly matches gravity.

**Fix**:
- Reduce `GROUND_EFFECT_MULT` (try 1.3 instead of 1.5)
- Reduce `GROUND_EFFECT_H` (try 2m instead of 3m)
- Add slight additional drag near water surface

### Physics explosion (NaN or infinite values)

**Cause**: Division by zero, uncapped forces, or dt spike.

**Fix**:
- Always cap dt to 1/30 second (0.033)
- Check speed > 0.01 before dividing by speed for drag direction
- Clamp all velocities to terminal values every frame
- Add NaN checks: `if (isNaN(vx)) vx = 0;`

### Thermals are overpowered

**Cause**: Thermal strength too high, or thermals are too close together.

**Fix**:
- Reduce `Thermal.strength` range to 2-5 m/s^2
- Increase minimum spacing between thermals to 50m
- Add thermal "cooldown" -- reduce effectiveness if craft stays too long
- Make thermal strength proportional to altitude (weaker near ground)

### Stall feels too punishing

**Cause**: Lift drops too fast beyond stall angle.

**Fix**:
- Increase `STALL_ANGLE` (25 degrees instead of 20)
- Reduce the stall penalty factor (0.05 instead of 0.08 per degree)
- Add a brief "stall warning" grace period before full stall
- Make stall recovery faster (pitch down = quick lift recovery)

### Bird collisions are frustrating

**Cause**: Too frequent, or speed penalty is too harsh.

**Fix**:
- Reduce bird flock density
- Reduce speed penalty from 20% to 10-15%
- Make bird flocks more visible (larger, more contrast)
- Give a brief invulnerability window after each hit
- Add subtle audio/visual warning before bird collision
