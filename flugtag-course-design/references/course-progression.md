# Course Progression Reference

## Table of Contents

- [Distance-Based Element Density Tables](#distance-based-element-density-tables)
- [Course Configuration Profiles](#course-configuration-profiles)
- [CourseManager Class](#coursemanager-class)
- [Adaptive Difficulty](#adaptive-difficulty)
- [Procedural Generation with Seeded Randomness](#procedural-generation-with-seeded-randomness)
- [Distance Milestone System](#distance-milestone-system)
- [Medal Thresholds](#medal-thresholds)

---

## Distance-Based Element Density Tables

### Thermal Density (thermals per 100 m)

| Distance Range | Pier | Cliff | Rooftop | Mountain |
|---------------|------|-------|---------|----------|
| 0-50 m | 0 (intro zone) | 0 | 0 | 0 |
| 50-100 m | 3.3 | 2.5 | 2.5 | 1.8 |
| 100-200 m | 2.8 | 2.2 | 2.2 | 1.6 |
| 200-300 m | 2.3 | 1.8 | 1.8 | 1.3 |
| 300-400 m | 2.0 | 1.5 | 1.5 | 1.1 |
| 400-500 m | 1.8 | 1.3 | 1.3 | 0.9 |
| 500 m+ | 1.6 | 1.1 | 1.1 | 0.8 |

### Bird Flock Frequency (flocks per 100 m)

| Distance Range | Pier | Cliff | Rooftop | Mountain |
|---------------|------|-------|---------|----------|
| 0-150 m | 0 | 0 | 0 | 0 |
| 150-200 m | 0.5 | 0.5 | 1.0 | 0.5 |
| 200-300 m | 1.0 | 1.0 | 2.0 | 1.5 |
| 300-400 m | 1.2 | 1.0 | 2.5 | 1.5 |
| 400-500 m | 1.5 | 1.2 | 3.0 | 2.0 |
| 500 m+ | 1.5 | 1.2 | 3.0 | 2.0 |

### Wind Gust Intensity (max force in m/s)

| Distance Range | Pier | Cliff | Rooftop | Mountain |
|---------------|------|-------|---------|----------|
| 0-200 m | 0 | 0.5 (ambient) | 0 | 1.0 (ambient) |
| 200-300 m | 1.0 | 2.0 | 1.5 | 2.5 |
| 300-400 m | 1.5 | 3.0 | 2.0 | 3.5 |
| 400-500 m | 2.0 | 3.5 | 2.5 | 4.5 |
| 500 m+ | 2.0 | 4.0 | 3.0 | 5.0 |

### Star / Bonus Placement (items per 100 m)

| Distance Range | Stars | Boosts | High-Alt Stars |
|---------------|-------|--------|----------------|
| 0-100 m | 0 | 0 | 0 |
| 100-200 m | 3 | 0 | 0 |
| 200-300 m | 3 | 1 | 1 |
| 300-400 m | 2 | 1 | 1 |
| 400-500 m | 2 | 1 | 2 |
| 500 m+ | 2 | 1 | 2 |

---

## Course Configuration Profiles

### Pier (Default)

```javascript
const PIER_CONFIG = {
  name: 'Pier',
  rampHeight: 10,
  unlockDistance: 0,  // Available at start
  theme: 'pier',

  thermals: {
    baseSpacing: 30,
    maxSpacing: 55,
    strengthRange: [4, 6],
    radiusRange: [5, 8]
  },

  wind: {
    gustSpacing: 80,
    maxForce: 2,
    ambientWind: 0,
    headwindOnset: 200
  },

  birds: {
    spacing: 80,
    flockSizeRange: [3, 4],
    introDistance: 150
  },

  turbulence: {
    onset: 400,
    maxIntensity: 0.6
  },

  clouds: {
    density: 'sparse',
    updraftChance: 0.6
  },

  groundEffect: {
    liftBonus: 0.3,
    dragReduction: 0.15
  }
};
```

### Cliff

```javascript
const CLIFF_CONFIG = {
  name: 'Cliff',
  rampHeight: 20,
  unlockDistance: 200,
  theme: 'coastal',

  thermals: {
    baseSpacing: 40,
    maxSpacing: 65,
    strengthRange: [3, 5],
    radiusRange: [4, 7]
  },

  wind: {
    gustSpacing: 30,
    maxForce: 4,
    ambientWind: -1, // Constant 1 m/s headwind
    headwindOnset: 100
  },

  birds: {
    spacing: 80,
    flockSizeRange: [3, 5],
    introDistance: 150
  },

  turbulence: {
    onset: 300,
    maxIntensity: 0.8
  },

  clouds: {
    density: 'moderate',
    updraftChance: 0.4
  },

  groundEffect: {
    liftBonus: 0.3,
    dragReduction: 0.15
  }
};
```

### Rooftop

```javascript
const ROOFTOP_CONFIG = {
  name: 'Rooftop',
  rampHeight: 15,
  unlockDistance: 350,
  theme: 'urban',

  thermals: {
    baseSpacing: 40,
    maxSpacing: 65,
    strengthRange: [3, 5],
    radiusRange: [4, 7]
  },

  wind: {
    gustSpacing: 50,
    maxForce: 3,
    ambientWind: 0, // Variable, handled by ambient gust system
    headwindOnset: 200
  },

  birds: {
    spacing: 30,
    flockSizeRange: [3, 6],
    introDistance: 100 // Birds appear earlier on rooftop
  },

  turbulence: {
    onset: 350,
    maxIntensity: 0.7
  },

  clouds: {
    density: 'sparse',
    updraftChance: 0.5
  },

  groundEffect: {
    liftBonus: 0.3,
    dragReduction: 0.15
  }
};
```

### Mountain

```javascript
const MOUNTAIN_CONFIG = {
  name: 'Mountain',
  rampHeight: 30,
  unlockDistance: 500,
  theme: 'alpine',

  thermals: {
    baseSpacing: 55,
    maxSpacing: 80,
    strengthRange: [2, 4],
    radiusRange: [3, 6]
  },

  wind: {
    gustSpacing: 35,
    maxForce: 5,
    ambientWind: -2, // Constant 2 m/s headwind
    headwindOnset: 50
  },

  birds: {
    spacing: 50,
    flockSizeRange: [3, 5],
    introDistance: 150
  },

  turbulence: {
    onset: 200,
    maxIntensity: 1.0
  },

  clouds: {
    density: 'dense',
    updraftChance: 0.3 // Most clouds are turbulent here
  },

  groundEffect: {
    liftBonus: 0.2,  // Reduced due to thin mountain air
    dragReduction: 0.10
  }
};

const COURSE_CONFIGS = {
  pier: PIER_CONFIG,
  cliff: CLIFF_CONFIG,
  rooftop: ROOFTOP_CONFIG,
  mountain: MOUNTAIN_CONFIG
};
```

---

## CourseManager Class

The `CourseManager` orchestrates all element managers and handles the overall
course lifecycle.

```javascript
class CourseManager {
  constructor(courseId, seed) {
    this.config = COURSE_CONFIGS[courseId];
    this.seed = seed || Date.now();

    // Initialize sub-managers
    this.thermalManager = new ThermalManager(this.seed, this.config);
    this.windGustManager = new WindGustManager(this.seed, this.config);
    this.birdFlockManager = new BirdFlockManager(this.seed, this.config);
    this.turbulenceSystem = new TurbulenceSystem(this.seed);
    this.cloudManager = new CloudManager(this.seed, this.config);
    this.collectibleManager = new CollectibleManager(this.seed, this.config);
    this.milestoneManager = new MilestoneManager();

    // State
    this.currentDistance = 0;
    this.maxDistanceReached = 0;
    this.activeElements = new Set();
    this.introSchedule = this._buildIntroSchedule();
  }

  _buildIntroSchedule() {
    // Gate when each element type becomes available
    return {
      thermals: 50,
      collectibles: 100,
      birds: this.config.birds.introDistance,
      windGusts: this.config.wind.headwindOnset,
      clouds: 300,
      turbulence: this.config.turbulence.onset
    };
  }

  /**
   * Main update loop. Called every frame.
   */
  update(dt, craft, cameraX, time) {
    this.currentDistance = craft.x;
    this.maxDistanceReached = Math.max(this.maxDistanceReached, craft.x);

    // Unlock element types based on distance
    this._updateActiveElements();

    // Generate elements ahead of the camera
    if (this.activeElements.has('thermals')) {
      this.thermalManager.generateUpTo(cameraX);
      this.thermalManager.update(dt, craft, cameraX);
    }

    if (this.activeElements.has('windGusts')) {
      this.windGustManager.generateUpTo(cameraX);
      this.windGustManager.update(dt, craft, cameraX);
    }

    if (this.activeElements.has('birds')) {
      this.birdFlockManager.generateUpTo(cameraX);
      this.birdFlockManager.update(dt, craft, cameraX);
    }

    if (this.activeElements.has('turbulence')) {
      this.turbulenceSystem.apply(dt, craft, time);
    }

    if (this.activeElements.has('clouds')) {
      this.cloudManager.update(dt, craft, cameraX);
    }

    // Collectibles are always active after their intro distance
    if (this.activeElements.has('collectibles')) {
      this.collectibleManager.generateUpTo(cameraX);
      this.collectibleManager.update(dt, craft, cameraX);
    }

    // Apply ambient wind (constant headwind on some courses)
    if (this.config.wind.ambientWind !== 0) {
      craft.vx += this.config.wind.ambientWind * dt;
    }

    // Apply altitude-based physics
    this._applyAltitudeEffects(dt, craft);

    // Check milestones
    this.milestoneManager.check(craft.x);
  }

  _updateActiveElements() {
    for (const [element, introDistance] of Object.entries(this.introSchedule)) {
      if (this.currentDistance >= introDistance) {
        this.activeElements.add(element);
      }
    }
  }

  _applyAltitudeEffects(dt, craft) {
    // Thin air above 25m
    const liftMult = getLiftMultiplier(craft.y);
    craft.currentLiftMultiplier = liftMult;

    // Ground effect below 3m
    if (craft.y < 3 && craft.y > 0) {
      const geBonus = calculateGroundEffect(craft.y, craft.baseLift);
      craft.vy += geBonus * dt;

      const reducedDrag = groundEffectDragReduction(craft.y, craft.baseDrag);
      craft.currentDrag = reducedDrag;
    } else {
      craft.currentDrag = craft.baseDrag;
    }
  }

  /**
   * Get all renderable elements for the current view.
   */
  getRenderables(cameraX, viewRange) {
    return {
      thermals: this.thermalManager.getVisibleThermals(cameraX, viewRange),
      gusts: this.windGustManager.getVisibleGusts(cameraX, viewRange),
      birds: this.birdFlockManager.getVisibleFlocks(cameraX, viewRange),
      clouds: this.cloudManager.getVisibleClouds(cameraX, viewRange),
      collectibles: this.collectibleManager.getVisible(cameraX, viewRange),
      milestones: this.milestoneManager.getVisible(cameraX, viewRange)
    };
  }

  /**
   * Reset course for a new flight attempt (new seed or same seed for replay).
   */
  reset(newSeed) {
    this.seed = newSeed || Date.now();
    this.currentDistance = 0;
    this.maxDistanceReached = 0;
    this.activeElements.clear();

    this.thermalManager = new ThermalManager(this.seed, this.config);
    this.windGustManager = new WindGustManager(this.seed, this.config);
    this.birdFlockManager = new BirdFlockManager(this.seed, this.config);
    this.turbulenceSystem = new TurbulenceSystem(this.seed);
    this.cloudManager = new CloudManager(this.seed, this.config);
    this.collectibleManager = new CollectibleManager(this.seed, this.config);
  }
}
```

---

## Adaptive Difficulty

The game adjusts element density for struggling players to keep the experience
fun without making it feel patronizing.

### Rules

| Player Best Distance | Adjustment |
|---------------------|------------|
| < 50 m | Extend intro zone to 80 m, auto-stabilize pitch for 3 s after launch |
| < 100 m | Increase early thermal density by 25%, increase thermal radius by 15% |
| < 150 m | Add one extra "rescue thermal" at the 80 m mark, low altitude |
| >= 150 m | No adjustments -- standard difficulty |

### Implementation

```javascript
function applyAdaptiveDifficulty(courseManager, playerBestDistance) {
  if (playerBestDistance >= 150) return; // No help needed

  const tm = courseManager.thermalManager;

  if (playerBestDistance < 50) {
    // Extend safe intro zone
    courseManager.introSchedule.thermals = 30; // Thermals start at 30m instead of 50m
    courseManager.launchStabilizeDuration = 3.0; // 3 seconds of auto-stabilization
  }

  if (playerBestDistance < 100) {
    // Boost early thermals
    for (const t of tm.thermals) {
      if (t.x < 150) {
        t.strength *= 1.25;
        t.radius *= 1.15;
      }
    }
  }

  if (playerBestDistance < 150) {
    // Add rescue thermal at 80m, low altitude
    tm.thermals.push({
      x: 80,
      y: 6,
      radius: 8,
      strength: 5.5,
      duration: 2.5,
      active: true,
      timer: 0,
      seed: 0
    });
    // Sort by x to maintain order
    tm.thermals.sort((a, b) => a.x - b.x);
  }
}
```

### Design Notes

- Adaptive difficulty is invisible to the player. No UI indicator, no "easy mode"
  label.
- Adjustments only affect the first 150 m of the course. Beyond that, difficulty
  is always standard.
- Once the player exceeds 150 m personal best, all adjustments are permanently
  removed for that course.

---

## Procedural Generation with Seeded Randomness

### Seeded Random Number Generator

```javascript
class SeededRandom {
  constructor(seed) {
    this.seed = seed;
    this.state = seed;
  }

  /**
   * Generate next pseudo-random number in [0, 1).
   * Uses a simple xorshift32 algorithm.
   */
  random() {
    this.state ^= this.state << 13;
    this.state ^= this.state >> 17;
    this.state ^= this.state << 5;
    return ((this.state >>> 0) / 4294967296);
  }

  /**
   * Generate random number in [min, max).
   */
  range(min, max) {
    return min + this.random() * (max - min);
  }

  /**
   * Generate random integer in [min, max].
   */
  rangeInt(min, max) {
    return Math.floor(this.range(min, max + 1));
  }

  /**
   * Get the next integer (for sub-seeding).
   */
  nextInt() {
    return (this.random() * 4294967296) >>> 0;
  }
}
```

### Seed Management

- Each flight gets a seed: `seed = Date.now()` for a new attempt.
- The seed is stored with the flight result for replay capability.
- Each sub-manager offsets the base seed to avoid correlated patterns:
  - `ThermalManager`: `seed + 0`
  - `WindGustManager`: `seed + 1000`
  - `BirdFlockManager`: `seed + 2000`
  - `CloudManager`: `seed + 3000`
  - `CollectibleManager`: `seed + 4000`

### Replay Mode

When replaying a flight, pass the same seed to `CourseManager`. The course layout
will be identical. Combined with recorded player inputs, this enables full
deterministic replay.

---

## Distance Milestone System

### Buoys (Every 50 m)

Small floating buoy markers placed on the water surface every 50 m. Each buoy
displays the distance number. Buoys serve as progress markers and give the
player a sense of how far they've traveled.

```javascript
class MilestoneManager {
  constructor() {
    this.buoyInterval = 50;       // Buoy every 50m
    this.bannerInterval = 100;    // Banner every 100m
    this.milestones = [];
    this.generatedUpTo = 0;
    this.triggeredMilestones = new Set();
  }

  generateUpTo(distance) {
    const target = distance + 200;

    while (this.generatedUpTo < target) {
      this.generatedUpTo += this.buoyInterval;

      const isBanner = this.generatedUpTo % this.bannerInterval === 0;

      this.milestones.push({
        x: this.generatedUpTo,
        type: isBanner ? 'banner' : 'buoy',
        distance: this.generatedUpTo,
        triggered: false
      });
    }
  }

  check(craftX) {
    for (const m of this.milestones) {
      if (!m.triggered && craftX >= m.x) {
        m.triggered = true;
        this.triggeredMilestones.add(m.distance);

        // Return milestone event for UI/scoring
        return {
          type: m.type,
          distance: m.distance,
          points: m.type === 'banner' ? 200 : 100
        };
      }
    }
    return null;
  }

  getVisible(cameraX, viewRange) {
    return this.milestones.filter(m =>
      m.x > cameraX - 20 &&
      m.x < cameraX + viewRange
    );
  }
}
```

### Banners (Every 100 m)

Larger celebration markers every 100 m. When the craft passes a banner:
- Brief confetti particle burst.
- Distance callout appears on screen ("100 m!", "200 m!").
- Crowd cheering audio cue.
- Score bonus awarded.

### Visual Design

- **Buoys**: Small red-and-white striped cylinders bobbing on the water surface.
  Distance number painted on the side.
- **Banners**: Stretched banner between two tall poles with the distance number.
  Celebration streamers when triggered.

---

## Medal Thresholds

### Per-Course Medals

| Course | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Pier | 75 m | 150 m | 300 m |
| Cliff | 100 m | 200 m | 400 m |
| Rooftop | 100 m | 200 m | 400 m |
| Mountain | 150 m | 300 m | 600 m |

### Medal Award Logic

```javascript
function getMedal(courseId, distance) {
  const thresholds = {
    pier:     { bronze: 75,  silver: 150, gold: 300 },
    cliff:    { bronze: 100, silver: 200, gold: 400 },
    rooftop:  { bronze: 100, silver: 200, gold: 400 },
    mountain: { bronze: 150, silver: 300, gold: 600 }
  };

  const t = thresholds[courseId];
  if (!t) return null;

  if (distance >= t.gold)   return 'gold';
  if (distance >= t.silver) return 'silver';
  if (distance >= t.bronze) return 'bronze';
  return null;
}
```

### Medal Display

- Medals are shown on the results screen after each flight.
- New medal unlocks trigger a special animation (medal drops in, shines).
- Course select screen shows the best medal earned for each course.
- All-gold achievement: unlock a secret cosmetic for the craft builder.

### Progression Requirements

To unlock the next course, the player must reach the unlock distance threshold on
any course (not necessarily earn a medal). This ensures progression is gated by
skill improvement, not medal grinding.

| Unlock | Requirement |
|--------|------------|
| Cliff | Any flight reaches 200 m |
| Rooftop | Any flight reaches 350 m |
| Mountain | Any flight reaches 500 m |
