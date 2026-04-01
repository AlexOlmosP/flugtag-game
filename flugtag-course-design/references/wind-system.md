# Wind System Reference

## Table of Contents

- [Thermal Column System](#thermal-column-system)
- [Wind Gust System](#wind-gust-system)
- [Turbulence System](#turbulence-system)
- [Ground Effect](#ground-effect)
- [Altitude-Based Air Density](#altitude-based-air-density)
- [Course-Specific Wind Configuration](#course-specific-wind-configuration)
- [Wind Visualization](#wind-visualization)

---

## Thermal Column System

### Data Structure

```javascript
// Single thermal instance
const thermal = {
  x: 0,           // World-space X position (meters from launch)
  y: 0,           // Center altitude of thermal column (meters)
  radius: 6,      // Radius of effect (4-8 m)
  strength: 4.5,  // Upward force in m/s (3-6)
  duration: 2.0,  // Seconds of force applied while craft is inside
  active: true,   // Whether this thermal is currently spawned
  timer: 0,       // Time craft has spent inside this thermal
  seed: 0         // Deterministic seed for visual variation
};
```

### Generation Algorithm

Thermals are placed using seeded pseudo-random generation with enforced minimum
spacing. The algorithm walks forward from the last placed thermal and picks
the next position within a range.

```javascript
class ThermalManager {
  constructor(courseSeed, courseConfig) {
    this.rng = new SeededRandom(courseSeed);
    this.config = courseConfig;
    this.thermals = [];
    this.spawnHorizon = 0;    // Furthest X we've generated to
    this.lookAhead = 200;     // Generate thermals 200m ahead of camera
  }

  /**
   * Pre-generate thermals up to the given distance.
   * Called every frame with camera X position.
   */
  generateUpTo(distanceX) {
    const target = distanceX + this.lookAhead;

    while (this.spawnHorizon < target) {
      // Spacing increases with distance: 30m at start, 60m at 500m+
      const baseSpacing = 30 + Math.min((this.spawnHorizon / 500) * 30, 30);
      const jitter = this.rng.range(-5, 5);
      const spacing = Math.max(30, baseSpacing + jitter);

      this.spawnHorizon += spacing;

      // Skip if before the element introduction threshold (50m)
      if (this.spawnHorizon < 50) continue;

      // Altitude: alternate between mid (12-20m) and low-mid (5-12m)
      const isMid = this.thermals.length % 2 === 0;
      const y = isMid
        ? this.rng.range(12, 20)
        : this.rng.range(5, 12);

      // Radius decreases with distance: 8m at start, 4m at 500m+
      const radius = Math.max(4, 8 - (this.spawnHorizon / 500) * 4);

      // Strength decreases with distance: 6 m/s at start, 3 m/s at 500m+
      const strength = Math.max(3, 6 - (this.spawnHorizon / 500) * 3);

      this.thermals.push({
        x: this.spawnHorizon,
        y: y,
        radius: radius + this.rng.range(-0.5, 0.5),
        strength: strength + this.rng.range(-0.3, 0.3),
        duration: 2.0,
        active: true,
        timer: 0,
        seed: this.rng.nextInt()
      });
    }
  }

  /**
   * Called every frame. Applies thermal forces to the craft
   * and despawns thermals that are behind the camera.
   */
  update(dt, craft, cameraX) {
    for (let i = this.thermals.length - 1; i >= 0; i--) {
      const t = this.thermals[i];

      // Despawn thermals far behind the camera
      if (t.x < cameraX - 50) {
        this.thermals.splice(i, 1);
        continue;
      }

      if (!t.active) continue;

      // Check if craft is inside thermal radius
      const dx = craft.x - t.x;
      const dy = craft.y - t.y;
      const dist = Math.sqrt(dx * dx + dy * dy);

      if (dist < t.radius) {
        // Apply upward force scaled by how centered the craft is
        const falloff = 1 - (dist / t.radius);
        craft.vy += t.strength * falloff * dt;

        // Track time spent in thermal
        t.timer += dt;
        if (t.timer >= t.duration) {
          t.active = false; // Thermal "spent" for this craft
        }
      }
    }
  }

  /**
   * Apply adaptive difficulty boost.
   * If player's best is under 100m, make early thermals 25% stronger.
   */
  applyAdaptiveDifficulty(playerBestDistance) {
    if (playerBestDistance < 100) {
      for (const t of this.thermals) {
        if (t.x < 150) {
          t.strength *= 1.25;
          t.radius *= 1.15;
        }
      }
    }
  }

  /**
   * Return thermals within visual range for rendering.
   */
  getVisibleThermals(cameraX, viewRange) {
    return this.thermals.filter(t =>
      t.active &&
      t.x > cameraX - 10 &&
      t.x < cameraX + viewRange
    );
  }
}
```

### Force Application

When the craft center is within a thermal's radius:

```
upwardForce = thermal.strength * (1 - distFromCenter / thermal.radius) * dt
craft.vy += upwardForce
```

The force falls off linearly from center to edge. A craft passing through the
center gets the full `strength` value; a craft grazing the edge gets nearly zero.

### Visual Representation

Thermals are rendered as shimmering particle columns:

- Particles rise from the base of the thermal to the top.
- Use Three.js `Points` with a custom `ShaderMaterial` for heat-haze distortion.
- Particles are semi-transparent, yellowish-white.
- The shimmer effect is visible when the thermal is within 20 m ahead of the
  camera (giving the player visual preview time).
- Particle density and brightness scale with thermal strength.

---

## Wind Gust System

### Data Structure

```javascript
const windGust = {
  startX: 0,       // X position where gust begins
  endX: 0,         // X position where gust ends
  direction: 'tailwind',  // 'tailwind' | 'headwind' | 'updraft' | 'downdraft'
  force: 2.0,      // Force magnitude in m/s
  duration: 2.0,   // How long the gust lasts (seconds)
  warningTime: 2.0,// Seconds of visual warning before gust activates
  state: 'warning', // 'warning' | 'active' | 'spent'
  timer: 0,        // Current timer within state
  altitude: { min: 0, max: 30 } // Altitude band affected
};
```

### Gust Types

| Type | Force Direction | Effect on Craft | Frequency |
|------|----------------|-----------------|-----------|
| Tailwind | +vx | Pushes forward, free distance | Common early |
| Headwind | -vx | Pushes backward, costs distance | After 200 m only |
| Updraft | +vy | Pushes up, free altitude | Moderate |
| Downdraft | -vy | Pushes down, dangerous near water | After 200 m |

### Warning System

Every gust has a 2-second warning phase before it activates:

1. **Warning phase (2 s)**: Animated wind-line particles appear in the gust zone.
   Lines point in the gust direction. Color-coded: green = helpful, red = harmful.
2. **Active phase (1-3 s)**: Full force applied to the craft. Wind lines intensify.
3. **Spent**: Gust disappears. Lines fade out over 0.5 s.

### WindGustManager Class

```javascript
class WindGustManager {
  constructor(courseSeed, courseConfig) {
    this.rng = new SeededRandom(courseSeed + 1000); // Offset seed from thermals
    this.config = courseConfig;
    this.gusts = [];
    this.spawnHorizon = 200; // No gusts before 200m (first 200m is gust-free)
    this.lookAhead = 150;
    this.maxActive = 2;      // Max simultaneous active gusts
  }

  generateUpTo(distanceX) {
    const target = distanceX + this.lookAhead;

    while (this.spawnHorizon < target) {
      // Gust spacing: 40-80m
      const spacing = this.rng.range(40, 80);
      this.spawnHorizon += spacing;

      // Determine type based on distance
      const typeRoll = this.rng.random();
      let direction;

      if (this.spawnHorizon < 200) {
        // Only helpful gusts early (should not reach here due to horizon init)
        direction = typeRoll < 0.6 ? 'tailwind' : 'updraft';
      } else if (this.spawnHorizon < 300) {
        // Mix: 60% helpful, 40% harmful
        if (typeRoll < 0.35) direction = 'tailwind';
        else if (typeRoll < 0.60) direction = 'updraft';
        else if (typeRoll < 0.80) direction = 'headwind';
        else direction = 'downdraft';
      } else {
        // 40% helpful, 60% harmful
        if (typeRoll < 0.20) direction = 'tailwind';
        else if (typeRoll < 0.40) direction = 'updraft';
        else if (typeRoll < 0.70) direction = 'headwind';
        else direction = 'downdraft';
      }

      // Force increases with distance: 1-4 m/s
      const baseForce = Math.min(1 + (this.spawnHorizon / 500) * 3, 4);
      const force = baseForce + this.rng.range(-0.3, 0.3);

      // Duration: 1-3s. Stronger gusts are shorter.
      const duration = Math.max(1, 3 - (force / 4) * 2) + this.rng.range(-0.2, 0.2);

      // Gust zone width: 15-30m
      const width = this.rng.range(15, 30);

      this.gusts.push({
        startX: this.spawnHorizon,
        endX: this.spawnHorizon + width,
        direction: direction,
        force: force,
        duration: duration,
        warningTime: 2.0,
        state: 'pending', // Not yet in warning range
        timer: 0,
        altitude: { min: 0, max: 35 }
      });
    }
  }

  update(dt, craft, cameraX) {
    let activeCount = 0;

    for (let i = this.gusts.length - 1; i >= 0; i--) {
      const g = this.gusts[i];

      // Despawn gusts far behind camera
      if (g.endX < cameraX - 50) {
        this.gusts.splice(i, 1);
        continue;
      }

      // Count active gusts
      if (g.state === 'active') activeCount++;

      // Transition from pending to warning when craft approaches
      if (g.state === 'pending' && craft.x > g.startX - 40) {
        g.state = 'warning';
        g.timer = 0;
      }

      // Warning -> Active
      if (g.state === 'warning') {
        g.timer += dt;
        if (g.timer >= g.warningTime) {
          if (activeCount < this.maxActive) {
            g.state = 'active';
            g.timer = 0;
          }
          // If max active reached, stay in warning (delayed)
        }
      }

      // Active -> Spent
      if (g.state === 'active') {
        g.timer += dt;

        // Apply force if craft is in the gust zone
        if (craft.x >= g.startX && craft.x <= g.endX &&
            craft.y >= g.altitude.min && craft.y <= g.altitude.max) {
          this._applyGustForce(g, craft, dt);
        }

        if (g.timer >= g.duration) {
          g.state = 'spent';
        }
      }
    }
  }

  _applyGustForce(gust, craft, dt) {
    switch (gust.direction) {
      case 'tailwind':
        craft.vx += gust.force * dt;
        break;
      case 'headwind':
        craft.vx -= gust.force * dt;
        break;
      case 'updraft':
        craft.vy += gust.force * dt;
        break;
      case 'downdraft':
        craft.vy -= gust.force * dt;
        break;
    }
  }

  /**
   * Return gusts in warning or active state for rendering.
   */
  getVisibleGusts(cameraX, viewRange) {
    return this.gusts.filter(g =>
      (g.state === 'warning' || g.state === 'active') &&
      g.endX > cameraX - 10 &&
      g.startX < cameraX + viewRange
    );
  }
}
```

---

## Turbulence System

Turbulence zones apply semi-random pitch perturbation to the craft. They simulate
rough, unstable air. Turbulence uses a Perlin-noise-like function to create smooth
but unpredictable wobble.

### Implementation

```javascript
class TurbulenceSystem {
  constructor(courseSeed) {
    this.seed = courseSeed + 2000;
    this.zones = []; // Generated turbulence zones
  }

  /**
   * Simple 1D noise function for turbulence.
   * Returns a value between -1 and 1.
   */
  noise(x) {
    // Simple hash-based noise (replace with proper Perlin for production)
    const n = Math.sin(x * 127.1 + this.seed) * 43758.5453;
    return (n - Math.floor(n)) * 2 - 1;
  }

  /**
   * Sample turbulence intensity at a world position.
   * Returns a pitch perturbation in radians/second.
   */
  sample(worldX, worldY, time) {
    // Turbulence only active after 400m
    if (worldX < 400) return 0;

    // Intensity ramps from 0 at 400m to max at 600m+
    const ramp = Math.min((worldX - 400) / 200, 1.0);

    // Multi-octave noise for natural feel
    const n1 = this.noise(worldX * 0.05 + time * 2.0);
    const n2 = this.noise(worldX * 0.1 + time * 3.5) * 0.5;
    const n3 = this.noise(worldY * 0.08 + time * 1.5) * 0.3;

    const combined = (n1 + n2 + n3) / 1.8; // Normalize to ~[-1, 1]

    // Max perturbation: 0.8 rad/s pitch wobble
    const maxPerturbation = 0.8;
    return combined * maxPerturbation * ramp;
  }

  /**
   * Apply turbulence to the craft each frame.
   */
  apply(dt, craft, time) {
    const perturbation = this.sample(craft.x, craft.y, time);
    craft.pitchRate += perturbation * dt;
  }
}
```

### Behavior

- Turbulence begins at 400 m and ramps to full intensity by 600 m.
- The perturbation is applied to pitch rate, not directly to velocity. This
  means the player can fight through turbulence with active pitch corrections.
- High-frequency components make the craft feel "bumpy" while low-frequency
  components create slow drifts.
- Turbulence is stronger at high altitude (cloud layer) than at mid altitude.

---

## Ground Effect

When a craft flies very close to the water surface (below 3 m altitude), it
experiences increased lift due to ground effect. This is a real aerodynamic
phenomenon and adds a risk/reward mechanic: fly low for a speed bonus, but
risk splashing.

### Formula

```javascript
function calculateGroundEffect(altitude, baseLift) {
  if (altitude >= 3.0) return 0;
  if (altitude <= 0) return 0; // Already splashed

  // Linear ramp: maximum bonus at altitude 0, zero at altitude 3
  const factor = (1 - altitude / 3.0) * 0.3;
  return factor * baseLift;
}
```

- At 0 m: +30% bonus lift (but you're about to splash).
- At 1 m: +20% bonus lift.
- At 2 m: +10% bonus lift.
- At 3 m+: No bonus.

### Speed Bonus

Ground effect also provides a slight forward speed preservation:

```javascript
function groundEffectDragReduction(altitude, baseDrag) {
  if (altitude >= 3.0) return baseDrag;
  const factor = (1 - altitude / 3.0) * 0.15;
  return baseDrag * (1 - factor); // Up to 15% drag reduction
}
```

---

## Altitude-Based Air Density

Above 25 m, the air is "thinner" and provides less lift. This discourages
indefinite climbing and creates a natural ceiling.

### Formula

```javascript
function getLiftMultiplier(altitude) {
  if (altitude <= 20) return 1.0;        // Full lift
  if (altitude >= 25) return 0.7;        // 30% reduction
  // Smooth transition from 20-25m
  const t = (altitude - 20) / 5;
  return 1.0 - (t * 0.3); // Linear interpolation from 1.0 to 0.7
}
```

### Effect on Gameplay

- A craft at 30 m altitude loses lift quickly and must pitch down to return to
  the MID zone.
- Skilled players can briefly enter the HIGH zone to collect bonus stars, then
  dive back to MID before losing too much altitude.
- The transition zone (20-25 m) acts as a gentle warning -- lift starts to fade
  before the full penalty kicks in.

---

## Course-Specific Wind Configuration

Each course has a unique wind profile that defines base values for all systems.

### Pier (Default Course)

| Parameter | Value |
|-----------|-------|
| Thermal density | High (30 m base spacing) |
| Thermal strength | 4-6 m/s |
| Wind gust frequency | Low (80 m spacing) |
| Max gust force | 2 m/s |
| Turbulence onset | 400 m |
| Bird density | Low (80 m spacing) |
| Ground effect bonus | Standard (30%) |
| Ambient wind | None (calm) |

### Cliff

| Parameter | Value |
|-----------|-------|
| Thermal density | Medium (40 m base spacing) |
| Thermal strength | 3-5 m/s |
| Wind gust frequency | High (30 m spacing) |
| Max gust force | 4 m/s |
| Turbulence onset | 300 m |
| Bird density | Low (80 m spacing) |
| Ground effect bonus | Standard (30%) |
| Ambient wind | Constant 1 m/s headwind |

### Rooftop

| Parameter | Value |
|-----------|-------|
| Thermal density | Medium (40 m base spacing) |
| Thermal strength | 3-5 m/s |
| Wind gust frequency | Medium (50 m spacing) |
| Max gust force | 3 m/s |
| Turbulence onset | 350 m |
| Bird density | High (30 m spacing) |
| Ground effect bonus | Standard (30%) |
| Ambient wind | Gusty 0-2 m/s variable |

### Mountain

| Parameter | Value |
|-----------|-------|
| Thermal density | Low (55 m base spacing) |
| Thermal strength | 2-4 m/s |
| Wind gust frequency | High (35 m spacing) |
| Max gust force | 5 m/s |
| Turbulence onset | 200 m |
| Bird density | Medium (50 m spacing) |
| Ground effect bonus | Reduced (20%) -- thin air |
| Ambient wind | Constant 2 m/s headwind + gusts |

---

## Wind Visualization

### Thermal Visuals

```javascript
function createThermalParticles(thermal) {
  const particleCount = 50;
  const geometry = new THREE.BufferGeometry();
  const positions = new Float32Array(particleCount * 3);
  const velocities = new Float32Array(particleCount * 3);

  for (let i = 0; i < particleCount; i++) {
    const angle = Math.random() * Math.PI * 2;
    const r = Math.random() * thermal.radius;
    positions[i * 3] = Math.cos(angle) * r;     // x offset
    positions[i * 3 + 1] = Math.random() * 10;  // y (height spread)
    positions[i * 3 + 2] = Math.sin(angle) * r;  // z offset (depth)

    velocities[i * 3 + 1] = thermal.strength * 0.5; // Rise speed
  }

  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

  const material = new THREE.PointsMaterial({
    size: 0.3,
    color: 0xffffcc,
    transparent: true,
    opacity: 0.3,
    blending: THREE.AdditiveBlending,
    depthWrite: false
  });

  const points = new THREE.Points(geometry, material);
  points.position.set(thermal.x, thermal.y - 5, 0);
  return { mesh: points, velocities };
}
```

### Wind Gust Visuals

Wind gusts are rendered as animated line segments that flow in the gust direction.

```javascript
function createGustLines(gust) {
  const lineCount = 20;
  const lines = [];

  for (let i = 0; i < lineCount; i++) {
    const x = gust.startX + Math.random() * (gust.endX - gust.startX);
    const y = Math.random() * 30;
    const length = 2 + Math.random() * 3;

    const geometry = new THREE.BufferGeometry();
    let dx = 0, dy = 0;

    switch (gust.direction) {
      case 'tailwind':  dx = length;  break;
      case 'headwind':  dx = -length; break;
      case 'updraft':   dy = length;  break;
      case 'downdraft': dy = -length; break;
    }

    const positions = new Float32Array([
      x, y, 0,
      x + dx, y + dy, 0
    ]);
    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

    const isHelpful = gust.direction === 'tailwind' || gust.direction === 'updraft';
    const material = new THREE.LineBasicMaterial({
      color: isHelpful ? 0x88ff88 : 0xff8888,
      transparent: true,
      opacity: gust.state === 'warning' ? 0.3 : 0.7
    });

    lines.push(new THREE.Line(geometry, material));
  }

  return lines;
}
```

### Cloud Layer Visuals

Clouds at HIGH altitude are rendered as soft, billowy Three.js meshes:

- Use `THREE.SphereGeometry` clusters with toon-shaded white material.
- Clouds drift slowly in the wind direction (1-2 m/s).
- On entry, apply a brief screen-space blur or fog effect to simulate reduced
  visibility.
- Updraft clouds glow faintly green at the base; turbulence clouds have a
  slight grey-purple tint.
