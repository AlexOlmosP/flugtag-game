# Scoring System Reference

## Table of Contents

- [Primary Score: Distance](#primary-score-distance)
- [Style Multiplier](#style-multiplier)
- [Performance Bonuses](#performance-bonuses)
- [Total Score Formula](#total-score-formula)
- [Crowd Reaction System](#crowd-reaction-system)
- [Medal System](#medal-system)
- [High Score Persistence](#high-score-persistence)
- [ScoringManager Class](#scoringmanager-class)

---

## Primary Score: Distance

The foundation of every score is distance traveled in meters. Every meter of
flight counts.

```
distanceScore = Math.floor(craft.x)
```

- 1 point per meter flown.
- Measured from the end of the ramp to the splash point.
- Displayed prominently during flight as a running counter.

---

## Style Multiplier

The craft's "style" stat (set during the build phase) applies a multiplier to the
total score. Style represents how creative/entertaining the craft design is.

| Style Stat | Multiplier |
|-----------|------------|
| 0 (minimum) | 1.0x |
| 25 | 1.25x |
| 50 | 1.5x |
| 75 | 1.75x |
| 100 (maximum) | 2.0x |

Formula:

```javascript
function getStyleMultiplier(styleStat) {
  // styleStat is 0-100
  return 1.0 + (styleStat / 100) * 1.0;
}
```

The style multiplier rewards players who invest in creative craft designs, even
if those designs sacrifice some aerodynamic performance. A beautiful craft that
flies 150 m scores the same as a boring craft that flies 300 m.

---

## Performance Bonuses

Bonuses are awarded for specific achievements during a flight. Multiple bonuses
can stack.

| Bonus | Condition | Points | Notes |
|-------|-----------|--------|-------|
| **Perfect Launch** | Power meter > 90% accuracy | +500 | Timing-based, on the launch ramp |
| **Thermal Chain** | Catch 3+ thermals consecutively | +200 per chain thermal | Resets if a thermal is missed between catches |
| **Low Rider** | Fly below 3 m altitude for 5+ continuous seconds | +300 | Must not splash; risky ground-effect play |
| **Sky High** | Reach altitude > 30 m at any point | +400 | Requires climbing into thin air zone |
| **Bird Dodge** | Pass within 2 m of a bird flock without collision | +100 | Per flock dodged; rewards precise flying |
| **Distance Milestone** | Every 50 m reached | +100 | Buoy milestones |
| **Distance Banner** | Every 100 m reached | +200 | Banner milestones (replaces buoy bonus at 100 m increments) |
| **Smooth Landing** | Splash angle within 10 degrees of horizontal | +150 | Nose-first or flat landings only |
| **Long Glide** | No altitude gained for 10+ seconds straight | +250 | Pure glide skill, no thermals used |

### Thermal Chain Details

A "thermal chain" tracks consecutive thermals caught without missing one. The
chain counter increments each time the craft enters a thermal radius. If the
craft passes a thermal's X position without entering it, the chain resets.

```
Chain of 3: +200 (for the 3rd thermal)
Chain of 4: +200 (for the 4th)
Chain of 5: +200 (for the 5th)
...
```

The bonus is awarded per thermal after the chain reaches 3. A chain of 6 thermals
awards +200 * 4 = +800 bonus points.

### Bird Dodge Details

A "near miss" is when the craft's bounding box passes within 2 m of the nearest
bird in a flock without colliding. The dodge bonus is awarded once per flock.

---

## Total Score Formula

```
totalScore = (distanceScore + sumOfBonuses) * styleMultiplier
```

Broken down:

```javascript
function calculateTotalScore(distance, bonuses, styleStat) {
  const distanceScore = Math.floor(distance);
  const bonusTotal = bonuses.reduce((sum, b) => sum + b.points, 0);
  const multiplier = getStyleMultiplier(styleStat);

  return Math.floor((distanceScore + bonusTotal) * multiplier);
}
```

### Example Flight

- Distance: 247 m = 247 pts
- Perfect Launch: +500
- Thermal Chain (5): +200 * 3 = +600
- Bird Dodge (2 flocks): +100 * 2 = +200
- Milestones: 50m + 100m + 150m + 200m = buoys at 50, 150 give +100 each; banners at 100, 200 give +200 each = +600
- Style stat: 60 -> multiplier = 1.6x
- **Total**: (247 + 500 + 600 + 200 + 600) * 1.6 = 2147 * 1.6 = **3,435 points**

---

## Crowd Reaction System

The crowd reacts to the flight in real-time based on distance reached. Reactions
are both audio (crowd sounds) and visual (text popups).

| Distance | Reaction Text | Audio | Visual Effect |
|----------|--------------|-------|---------------|
| < 50 m | "OHHH..." | Sympathetic groan | Crowd shakes heads |
| 50-100 m | "Not bad!" | Polite clapping | Light applause animation |
| 100-200 m | "WOOO!" | Cheering | Crowd stands up, waves |
| 200-300 m | "INCREDIBLE!" | Wild cheering, whistles | Crowd jumping, confetti |
| 300-400 m | "UNBELIEVABLE!" | Roaring crowd | Air horns, streamers |
| 400-500 m | "LEGENDARY!" | Deafening roar | Full crowd mania, fireworks |
| 500 m+ | "WORLD RECORD!" | Stadium-level roar | Everything at once |

### Crowd Reaction Triggers

Reactions trigger at two moments:

1. **During flight**: When the craft crosses a distance threshold for the first
   time, a brief cheer erupts. This is a quick audio cue + small text popup.
2. **At splash**: The final reaction is the big one. Full crowd animation plays.
   The reaction text stays on screen for 3 seconds before transitioning to the
   results screen.

### Implementation

```javascript
const CROWD_REACTIONS = [
  { minDistance: 0,   text: 'OHHH...',       audio: 'crowd_groan',    intensity: 0.2 },
  { minDistance: 50,  text: 'Not bad!',      audio: 'crowd_clap',     intensity: 0.4 },
  { minDistance: 100, text: 'WOOO!',         audio: 'crowd_cheer',    intensity: 0.6 },
  { minDistance: 200, text: 'INCREDIBLE!',   audio: 'crowd_wild',     intensity: 0.8 },
  { minDistance: 300, text: 'UNBELIEVABLE!', audio: 'crowd_roar',     intensity: 0.9 },
  { minDistance: 400, text: 'LEGENDARY!',    audio: 'crowd_legendary', intensity: 1.0 },
  { minDistance: 500, text: 'WORLD RECORD!', audio: 'crowd_record',   intensity: 1.0 }
];

function getCrowdReaction(distance) {
  let reaction = CROWD_REACTIONS[0];
  for (const r of CROWD_REACTIONS) {
    if (distance >= r.minDistance) {
      reaction = r;
    }
  }
  return reaction;
}
```

---

## Medal System

### Thresholds

| Course | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Pier | 75 m | 150 m | 300 m |
| Cliff | 100 m | 200 m | 400 m |
| Rooftop | 100 m | 200 m | 400 m |
| Mountain | 150 m | 300 m | 600 m |

### Medal Award Animation

When a player earns a new medal (or upgrades an existing one):

1. Results screen shows current flight stats.
2. Medal drops in from above with a metallic "clink" sound.
3. Medal shines with a brief lens-flare effect.
4. If it's an upgrade (e.g., Bronze -> Silver), the old medal fades out as the
   new one drops in.
5. "NEW!" badge appears next to the medal for first-time awards.

### All-Gold Achievement

If the player earns Gold on all four courses, unlock a special cosmetic item
in the craft builder (e.g., a golden paint option or trophy wing ornament).

---

## High Score Persistence

### Data Structure

```javascript
// Stored in localStorage
const playerData = {
  courses: {
    pier: {
      bestDistance: 0,
      bestScore: 0,
      bestMedal: null,     // null | 'bronze' | 'silver' | 'gold'
      totalFlights: 0,
      totalDistance: 0,     // Cumulative across all flights
      bestSeed: null        // Seed of the best flight (for replay)
    },
    cliff: { /* same structure */ },
    rooftop: { /* same structure */ },
    mountain: { /* same structure */ }
  },
  globalBestDistance: 0,    // Best distance across all courses
  totalFlights: 0,
  unlockedCourses: ['pier']
};
```

### Save / Load

```javascript
function savePlayerData(data) {
  localStorage.setItem('flugtag_player_data', JSON.stringify(data));
}

function loadPlayerData() {
  const raw = localStorage.getItem('flugtag_player_data');
  if (!raw) return createDefaultPlayerData();
  return JSON.parse(raw);
}

function createDefaultPlayerData() {
  return {
    courses: {
      pier:     { bestDistance: 0, bestScore: 0, bestMedal: null, totalFlights: 0, totalDistance: 0, bestSeed: null },
      cliff:    { bestDistance: 0, bestScore: 0, bestMedal: null, totalFlights: 0, totalDistance: 0, bestSeed: null },
      rooftop:  { bestDistance: 0, bestScore: 0, bestMedal: null, totalFlights: 0, totalDistance: 0, bestSeed: null },
      mountain: { bestDistance: 0, bestScore: 0, bestMedal: null, totalFlights: 0, totalDistance: 0, bestSeed: null }
    },
    globalBestDistance: 0,
    totalFlights: 0,
    unlockedCourses: ['pier']
  };
}
```

### Post-Flight Update

```javascript
function updatePlayerData(data, courseId, flightResult) {
  const course = data.courses[courseId];

  // Update course stats
  course.totalFlights++;
  course.totalDistance += flightResult.distance;

  // Update best distance
  if (flightResult.distance > course.bestDistance) {
    course.bestDistance = flightResult.distance;
    course.bestSeed = flightResult.seed;
  }

  // Update best score
  if (flightResult.totalScore > course.bestScore) {
    course.bestScore = flightResult.totalScore;
  }

  // Update medal
  const medal = getMedal(courseId, flightResult.distance);
  if (medal) {
    const medalRank = { bronze: 1, silver: 2, gold: 3 };
    const currentRank = course.bestMedal ? medalRank[course.bestMedal] : 0;
    if (medalRank[medal] > currentRank) {
      course.bestMedal = medal;
    }
  }

  // Update global stats
  data.totalFlights++;
  data.globalBestDistance = Math.max(data.globalBestDistance, flightResult.distance);

  // Check course unlocks
  if (flightResult.distance >= 200 && !data.unlockedCourses.includes('cliff')) {
    data.unlockedCourses.push('cliff');
  }
  if (flightResult.distance >= 350 && !data.unlockedCourses.includes('rooftop')) {
    data.unlockedCourses.push('rooftop');
  }
  if (flightResult.distance >= 500 && !data.unlockedCourses.includes('mountain')) {
    data.unlockedCourses.push('mountain');
  }

  savePlayerData(data);
  return data;
}
```

---

## ScoringManager Class

Complete implementation that tracks all scoring during a flight.

```javascript
class ScoringManager {
  constructor(styleStat) {
    this.styleStat = styleStat;
    this.styleMultiplier = 1.0 + (styleStat / 100) * 1.0;

    // Distance tracking
    this.distance = 0;

    // Bonus tracking
    this.bonuses = [];
    this.perfectLaunch = false;

    // Thermal chain state
    this.thermalChain = 0;
    this.lastThermalX = -1;
    this.missedThermal = false;

    // Low rider tracking
    this.lowRiderTimer = 0;
    this.lowRiderAwarded = false;

    // Sky high tracking
    this.skyHighAwarded = false;

    // Long glide tracking
    this.noAltGainTimer = 0;
    this.longGlideAwarded = false;

    // Bird dodge tracking
    this.dodgedFlocks = new Set();

    // Milestone tracking
    this.lastMilestone = 0;
  }

  /**
   * Called once after launch with the power meter accuracy.
   */
  recordLaunch(powerAccuracy) {
    if (powerAccuracy > 0.9) {
      this.perfectLaunch = true;
      this._addBonus('Perfect Launch', 500);
    }
  }

  /**
   * Called when the craft enters a thermal.
   */
  recordThermalCatch(thermalX) {
    this.thermalChain++;
    this.lastThermalX = thermalX;
    this.missedThermal = false;

    if (this.thermalChain >= 3) {
      this._addBonus('Thermal Chain', 200);
    }
  }

  /**
   * Called when a thermal's X position is passed without entering it.
   */
  recordThermalMiss() {
    this.thermalChain = 0;
    this.missedThermal = true;
  }

  /**
   * Called when the craft narrowly avoids a bird flock.
   */
  recordBirdDodge(flockId) {
    if (!this.dodgedFlocks.has(flockId)) {
      this.dodgedFlocks.add(flockId);
      this._addBonus('Bird Dodge', 100);
    }
  }

  /**
   * Called every frame during flight.
   */
  update(dt, craft) {
    this.distance = craft.x;

    // Low Rider: below 3m for 5+ seconds
    if (craft.y < 3 && craft.y > 0) {
      this.lowRiderTimer += dt;
      if (this.lowRiderTimer >= 5 && !this.lowRiderAwarded) {
        this.lowRiderAwarded = true;
        this._addBonus('Low Rider', 300);
      }
    } else {
      this.lowRiderTimer = 0;
    }

    // Sky High: reach above 30m
    if (craft.y > 30 && !this.skyHighAwarded) {
      this.skyHighAwarded = true;
      this._addBonus('Sky High', 400);
    }

    // Long Glide: no altitude gained for 10+ seconds
    if (craft.vy <= 0) {
      this.noAltGainTimer += dt;
      if (this.noAltGainTimer >= 10 && !this.longGlideAwarded) {
        this.longGlideAwarded = true;
        this._addBonus('Long Glide', 250);
      }
    } else {
      this.noAltGainTimer = 0;
    }

    // Distance milestones
    const currentMilestone = Math.floor(craft.x / 50) * 50;
    if (currentMilestone > this.lastMilestone && currentMilestone > 0) {
      this.lastMilestone = currentMilestone;

      if (currentMilestone % 100 === 0) {
        this._addBonus(`${currentMilestone}m Banner`, 200);
      } else {
        this._addBonus(`${currentMilestone}m Buoy`, 100);
      }
    }
  }

  /**
   * Called at splash. Check for smooth landing bonus.
   */
  recordSplash(splashAngle) {
    if (Math.abs(splashAngle) < 10 * (Math.PI / 180)) {
      this._addBonus('Smooth Landing', 150);
    }
  }

  /**
   * Calculate the final score.
   */
  getFinalScore() {
    const distanceScore = Math.floor(this.distance);
    const bonusTotal = this.bonuses.reduce((sum, b) => sum + b.points, 0);
    const rawTotal = distanceScore + bonusTotal;
    const finalScore = Math.floor(rawTotal * this.styleMultiplier);

    return {
      distance: this.distance,
      distanceScore: distanceScore,
      bonuses: [...this.bonuses],
      bonusTotal: bonusTotal,
      styleMultiplier: this.styleMultiplier,
      rawTotal: rawTotal,
      finalScore: finalScore
    };
  }

  /**
   * Get current running score (for HUD display during flight).
   */
  getRunningScore() {
    const distanceScore = Math.floor(this.distance);
    const bonusTotal = this.bonuses.reduce((sum, b) => sum + b.points, 0);
    return Math.floor((distanceScore + bonusTotal) * this.styleMultiplier);
  }

  /**
   * Get the list of bonuses earned so far (for HUD popups).
   */
  getRecentBonuses(withinLastMs = 2000) {
    const now = performance.now();
    return this.bonuses.filter(b => now - b.timestamp < withinLastMs);
  }

  _addBonus(name, points) {
    this.bonuses.push({
      name: name,
      points: points,
      timestamp: performance.now()
    });
  }
}
```

### HUD Integration

During flight, the scoring manager feeds data to the HUD:

- **Distance counter**: Always visible, top-center. Shows meters flown.
- **Running score**: Below distance counter. Updates in real-time.
- **Bonus popups**: When a bonus is earned, a brief "+500 Perfect Launch!" popup
  animates from center-right and fades out over 1.5 seconds.
- **Thermal chain indicator**: When chain >= 2, show "Chain x3" with fire icon.
- **Low rider warning**: When below 3 m, show altitude warning with red tint.

### Results Screen Integration

After splash, the `getFinalScore()` output drives the results screen:

1. Distance rolls up from 0 to final value (1 second animation).
2. Each bonus line appears one at a time (0.3 second stagger).
3. Style multiplier is revealed with a "x1.6" animation.
4. Final score drops in with impact effect.
5. Medal (if earned) drops in.
6. Crowd reaction text and audio play.
7. "Try Again" and "Course Select" buttons appear.
