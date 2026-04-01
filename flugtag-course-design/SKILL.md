---
name: flugtag-course-design
description: >
  Triggers for: wind patterns, thermal placement, altitude zones, environmental
  hazards, course progression, difficulty tuning, bird flocks, distance milestones,
  course unlocks, flights that feel too short / too long / too easy / too hard /
  too repetitive.
---

# Flugtag Course Design Skill

## Design Philosophy

1. **Every splash must feel earned.** The player always understands what they could
   have done differently -- pitched up sooner, caught that thermal, avoided the
   flock. No "unfair" deaths; the information was always visible in advance.

2. **Difficulty is distance.** The further you glide, the sparser the thermals,
   the stronger the headwinds, and the more turbulent the air. Distance *is* the
   difficulty slider -- no separate setting needed.

3. **Teach through flight.** New elements are introduced one at a time over
   distance. The first thermal is impossible to miss. The first bird flock has a
   huge gap. The player learns by doing, never by reading.

4. **Reward exploration of the altitude envelope.** High altitude is risky (thin
   air, less lift) but offers rare bonus stars. Low altitude is dangerous (one
   wrong pitch = splash) but grants ground-effect speed. The sweet spot is in the
   middle -- but the rewards are at the edges.

5. **No two flights feel the same.** Course layouts use seeded semi-random
   generation. The same seed produces the same course (for replays and
   leaderboards), but each new attempt shuffles thermal positions, bird flock
   altitudes, and gust timing just enough to keep things fresh.

---

## Course Elements

### Thermals (Rising Air Columns)
Invisible columns of warm, rising air. When the craft enters a thermal's radius
the player feels a sustained upward push for roughly 2 seconds. Thermals are
telegraphed by a subtle visual shimmer (heat-haze particles) visible ~20 m before
the craft reaches them. They are the primary tool for extending flight distance.

- Appear every 30-60 m (density decreases with distance).
- Radius: 4-8 m. Strength: 3-6 m/s upward force.
- Always placed in the MID or LOW-MID altitude band so the player must maintain
  reasonable altitude to benefit.

### Wind Gusts (Horizontal / Vertical Bursts)
Short bursts of moving air lasting 1-3 seconds. Types:

| Type | Effect |
|------|--------|
| Tailwind | Pushes craft forward -- helpful, free distance |
| Headwind | Pushes craft backward -- harmful, costs distance |
| Updraft | Pushes craft up -- helpful but may overshoot into thin air |
| Downdraft | Pushes craft down -- dangerous near water |

- Telegraphed by animated wind-line particles 2 seconds before the gust arrives.
- Headwinds never appear before 200 m.
- Max 2 gusts active at once.

### Bird Flocks
Groups of 3-5 birds flying in formation at a specific altitude band (5-8 m tall).
Hitting a bird causes speed loss and a pitch wobble. Flocks are always dodgeable
by pitching above or below the band.

- Minimum 50 m between flocks.
- Flock size increases with distance (3 early, 5+ after 300 m).
- Birds are visually distinct and audible (squawking) from ~25 m away.

### Cloud Layers
Appear at HIGH altitude (>25 m). Some clouds contain updrafts (free lift), others
contain turbulence (random pitch wobble). Entering a cloud reduces visibility
briefly. Clouds reward risky high-altitude play -- but the payoff is uncertain.

### Distance Bonuses
Collectible stars and boost pickups placed at milestone distances. Stars add to
score; boosts provide a brief speed/lift impulse. Placed to guide the player
toward optimal flight paths.

---

## Altitude Zones

| Zone | Altitude | Physics Effect | Reward / Risk |
|------|----------|---------------|---------------|
| **HIGH** | > 25 m | Lift coefficient -30%, thin air | Rare 500 pt bonus stars, cloud layer access |
| **MID** | 8-25 m | Normal physics | Sweet spot, most thermals spawn here |
| **LOW** | 2-8 m | Ground effect: speed bonus +15% | Danger zone -- one bad pitch = splash |
| **SPLASH** | 0-2 m | Ground effect slight lift, last chance | Critical -- pull up or it's over |

### Zone Transitions
- HIGH to MID (25 m): Lift gradually restores over 20-25 m transition band.
- MID to LOW (8 m): Ground effect kicks in, speed increases, risk increases.
- LOW to SPLASH (2 m): Warning indicator flashes, crowd gasps audibly.

---

## Element Introduction Schedule

| Distance | New Element | How It's Introduced |
|----------|------------|---------------------|
| 0-50 m | Nothing | Pure glide. Player learns pitch up/down controls. |
| 50-100 m | Thermals | First thermal is large (8 m radius) and obvious (strong shimmer). |
| 100-150 m | Stars | Collectible stars appear along the natural flight path. |
| 150-200 m | Birds | First flock is small (3 birds), at a single altitude, wide gap above/below. |
| 200-300 m | Wind gusts | Gentle tailwinds first (helpful), then first headwind at ~250 m. |
| 300-400 m | Cloud layer | Clouds visible at high altitude; first cloud contains an updraft (reward). |
| 400-500 m | Turbulence | Random pitch perturbation zones appear. Mild at first. |
| 500 m+ | Everything | Full complexity. All elements combined, maximum density/intensity. |

---

## Pacing Structure

```
Distance (m)
0         100        200        300        400        500+
|---------|----------|----------|----------|----------|---------->
|  CALM   | THERMALS | BIRDS +  | CLOUDS + | TURB +   | FULL
|  Learn  | + Stars  | WIND     | Density  | Intensity| CHAOS
|  pitch  |          | intro    | ramps up | ramps up |
|         |          |          |          |          |
Thermal   |||  ||  ||    |   |    |   |     |    |      |     |
density:  DENSE -------> MEDIUM ----------> SPARSE ----------->
Wind:     none   none    gentle   moderate  strong    extreme
Birds:    none   none    sparse   medium    dense     dense
Crowd:    silent  ohhh   clap     WOOO!     INCREDIBLE LEGENDARY
```

---

## Course Unlocks

| Course | Unlock Condition | Ramp Height | Special Feature |
|--------|-----------------|-------------|-----------------|
| **Pier** | Available at start | 10 m | Calm winds, standard conditions. The training course. |
| **Cliff** | Best distance >= 200 m | 20 m | Higher launch, stronger winds, ocean gusts. |
| **Rooftop** | Best distance >= 350 m | 15 m | Urban setting, heavy bird traffic, updrafts off buildings. |
| **Mountain** | Best distance >= 500 m | 30 m | Extreme launch height, thin air everywhere, severe turbulence. |

---

## Thermal Placement Rules

1. **Minimum spacing**: 30 m between thermal centers.
2. **Density curve**: Starts at ~1 thermal per 30 m, drops to ~1 per 60 m at 500 m+.
   Formula: `spacing = 30 + (distance / 500) * 30` (clamped to 30-60).
3. **Altitude placement**: Thermals alternate between mid-altitude (12-20 m) and
   low-mid altitude (5-12 m). Never spawn in HIGH zone (would be too generous) or
   SPLASH zone (would be invisible).
4. **Visual preview**: Shimmer particles render when thermal is within 20 m ahead
   of the camera. Players can plan ahead.
5. **Radius variation**: 4-8 m radius, trending smaller at greater distances.
6. **Strength variation**: 3-6 m/s upward, trending weaker at greater distances.
7. **Active duration**: Each thermal applies force for ~2 seconds of craft contact.

---

## Bird Flock Rules

1. **Minimum spacing**: 50 m between flock centers.
2. **Flock size**: 3 birds at 150-300 m, 4 birds at 300-400 m, 5+ birds at 400 m+.
3. **Altitude band**: Each flock occupies a 5-8 m vertical band. The band is
   chosen semi-randomly but always leaves a dodgeable gap above or below.
4. **Always dodgeable**: If a flock is at 10-16 m, the player can pitch above 16 m
   or dive below 10 m. Flocks never span the full altitude range.
5. **Speed**: Birds move in the same direction as the craft but slower, so relative
   approach speed is moderate. Player has ~1.5 s of reaction time after visual
   acquisition.
6. **Collision effect**: Speed reduced by 20%, random pitch wobble for 1 s, no
   instant-death. Recoverable but costly.
7. **Audio cue**: Squawking sound begins at 25 m distance.

---

## Wind Pattern Rules

1. **Warning system**: Animated wind-line particles appear 2 seconds before a gust
   affects the craft. Direction of lines indicates gust type.
2. **Headwind restriction**: No headwinds in the first 200 m. Only tailwinds and
   mild updrafts appear early.
3. **Strength curve**: Gust force starts at 1 m/s and increases to 4 m/s at 500 m+.
   Formula: `force = 1 + (distance / 500) * 3` (clamped to 1-4).
4. **Simultaneous limit**: Maximum 2 gusts active at the same time.
5. **Duration**: 1-3 seconds per gust. Short gusts are stronger, long gusts are
   gentler.
6. **Type distribution by distance**:
   - 0-200 m: 100% tailwind/updraft (helpful only).
   - 200-300 m: 60% helpful, 40% harmful.
   - 300 m+: 40% helpful, 60% harmful.

---

## Implementation Checklist

- [ ] `ThermalManager` class: spawn, update, apply force, despawn behind camera
- [ ] Thermal visual: heat-shimmer particle system (Three.js PointsMaterial)
- [ ] `WindGustManager` class: spawn, warning phase, active phase, despawn
- [ ] Wind visual: animated directional lines (Three.js Line segments)
- [ ] `BirdFlockManager` class: spawn, move, collision detect, despawn
- [ ] Bird models: simple toon-shaded bird meshes with flap animation
- [ ] `CloudManager` class: spawn cloud meshes at HIGH altitude with updraft/turbulence zones
- [ ] `CourseManager` class: orchestrate all element managers, handle seeded generation
- [ ] Altitude zone system: apply lift modifiers, ground effect, thin-air penalty
- [ ] Distance milestone markers: buoy models every 50 m, banners every 100 m
- [ ] Element introduction schedule: gate new element types by current distance
- [ ] Adaptive difficulty: if player best < 100 m, boost early thermal density by 25%
- [ ] Course unlock system: track per-course best distance, unlock next course at threshold
- [ ] Scoring integration: feed distance, bonuses, and multiplier into ScoringManager

---

## Test Prompts

Use these prompts to verify course design quality:

1. **"My first flight went 30 m and I don't know what happened."**
   The first 50 m should be completely clear. If the player splashed at 30 m, the
   pitch tutorial was insufficient. Add a more forgiving pitch-control grace period
   or auto-stabilize briefly at launch.

2. **"I can fly 200 m easily but 300 m feels impossible."**
   Check the thermal density curve at 200-300 m. The transition from "birds + wind
   intro" to "clouds + density ramp" may be too steep. Smooth the difficulty curve
   or add one extra thermal in the 250-300 m range.

3. **"Every flight feels the same."**
   Seeded randomness may have too little variance. Increase the random offset range
   for thermal X positions, bird altitudes, and gust timing. Ensure each seed
   produces meaningfully different layouts.

4. **"I never want to fly high because there's nothing up there."**
   HIGH zone rewards are missing or too rare. Ensure 500 pt bonus stars spawn above
   25 m at least once per 150 m. Add visible cloud-layer updrafts as a reward for
   climbing.

5. **"Birds are unfair -- I can't dodge them."**
   Check that flocks always leave a 5+ m gap above or below. Verify the audio cue
   plays at 25 m distance. Ensure the bird altitude band is clearly visible (dark
   silhouettes against sky).

6. **"The game is too easy. I flew 800 m on my first try."**
   Thermals are too dense or too strong at distance. Verify the density curve drops
   past 500 m. Check that headwind strength reaches 4 m/s. Consider reducing
   ground-effect bonus above 500 m distance.

7. **"Wind gusts feel random and unfair."**
   Verify the 2-second warning is working and visually clear. Check that gust
   direction is readable from the wind-line animation. No gust should be undodgeable
   -- the player should always be able to pitch through it.
