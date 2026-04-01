# Part Stats Reference

Complete stats system for all craft parts in the Red Bull Flugtag game.

---

## Table of Contents

1. [Stat Formula](#stat-formula)
2. [Fuselage Base Stats](#fuselage-base-stats)
3. [Wing Modifiers](#wing-modifiers)
4. [Propulsion Modifiers](#propulsion-modifiers)
5. [Decoration Stat Nudges](#decoration-stat-nudges)
6. [How Stats Affect Gameplay](#how-stats-affect-gameplay)
7. [Example Builds](#example-builds)
8. [Stat Lookup Tables (Code)](#stat-lookup-tables-code)

---

## Stat Formula

```
finalStat = clamp(fuselageBase + wingModifier + propulsionModifier + decorationBonuses, 1, 5)
```

- All three stats (Lift, Weight, Style) use the same formula
- `clamp(value, 1, 5)` -- stats never go below 1 or above 5
- Values are rounded to the nearest integer for display
- Decoration bonuses are small nudges (+1 or -1 per decoration)
- A fully loaded craft (all 4 decoration slots filled) can shift a stat by up to +4 or -4, but clamping keeps things in range

---

## Fuselage Base Stats

The fuselage is the largest contributor to overall stats. Each fuselage has a distinct personality.

| Fuselage | Lift | Weight | Style | Design Notes |
|----------|------|--------|-------|-------------|
| `bathtub` | 2 | 3 | 3 | Classic all-rounder. Medium everything. The default starting craft. |
| `shopping_cart` | 2 | 1 | 2 | Ultralight wire frame. Fast off the ramp but boring. Budget pick. |
| `rubber_duck` | 1 | 2 | 5 | Crowd favorite -- terrible aerodynamics but maximum showmanship. |
| `piano` | 1 | 5 | 4 | Heavy and dramatic. Plummets fast but judges love it. |
| `boat` | 3 | 3 | 2 | Designed to float (not fly). Decent lift, meh style. |
| `rocket` | 4 | 2 | 3 | Best base lift. Looks like it should fly. Cardboard is light. |

### Design philosophy

- **Light fuselages** (shopping_cart, rocket): Low weight enables faster ramp speed
- **Heavy fuselages** (piano): Dramatic but need max-lift wings to compensate
- **Stylish fuselages** (rubber_duck, piano): High base style means less decoration needed for style points
- **Lifty fuselages** (rocket, boat): Naturally fly farther, good for distance runs

---

## Wing Modifiers

Wings are the primary way to adjust lift. They modify all three stats relative to the fuselage base.

| Wings | Lift | Weight | Style | Design Notes |
|-------|------|--------|-------|-------------|
| `biplane` | +2 | +1 | +0 | Best raw lift but adds structural weight. Classic choice. |
| `delta` | +1 | +0 | +1 | Balanced. Clean lines look sharp. No weight penalty. |
| `hang_glider` | +2 | +0 | -1 | Maximum lift, no weight penalty, but looks utilitarian. |
| `cardboard_box` | +0 | -1 | +2 | Terrible wings but hilarious. Crowd loves the effort. |
| `umbrella` | +1 | +0 | +1 | Surprisingly decent. Mary Poppins energy. |

### Wing selection strategy

- **Going for distance?** Pick `hang_glider` or `biplane` for max lift
- **Going for style?** `cardboard_box` is comedy gold (+2 style, -1 weight bonus)
- **Balanced?** `delta` or `umbrella` give +1/+1 with no downsides

---

## Propulsion Modifiers

Propulsion provides a small in-flight boost and tweaks stats. The boost activates
during the first 3 seconds of flight.

| Propulsion | Lift | Weight | Style | Boost Duration | Boost Force | Notes |
|-----------|------|--------|-------|---------------|-------------|-------|
| `rubber_band` | +0 | +0 | +1 | 2s | 0.3 | Light and quirky. Unwinds quickly. |
| `fan` | +0 | +1 | +0 | 3s | 0.5 | Heavier but stronger sustained push. |
| `pedal` | +1 | +1 | +0 | 5s | 0.4 | Sustained power. Adds weight from mechanism. |
| `none` | +0 | -1 | +0 | 0s | 0.0 | No propulsion = lighter craft. Pure glider. |

### Propulsion notes

- `rubber_band`: Best for style builds, minimal weight impact
- `fan`: Strongest raw boost but adds weight. Good for rocket builds
- `pedal`: Longest boost duration. Pilot animation of pedaling adds visual flair
- `none`: Choosing no propulsion gives a -1 weight bonus (lighter craft)

---

## Decoration Stat Nudges

Each decoration provides a small +1 or -1 nudge to one stat. These are flavor
adjustments, not build-defining. A craft has up to 4 decoration slots.

### Fuselage Side (slot: `fuselage_side`)

| Decoration | Lift | Weight | Style |
|-----------|------|--------|-------|
| `team_banner` | +0 | +0 | +1 |
| `sponsor_logo` | +0 | +0 | +1 |
| `flame_paint` | +0 | +0 | +1 |
| `number` | +0 | +0 | +0 |
| `stars` | +0 | +0 | +1 |

> Side decorations are primarily style boosters. `number` is the exception (neutral).

### Wing Tip (slot: `wing_tip`)

| Decoration | Lift | Weight | Style |
|-----------|------|--------|-------|
| `streamers` | -1 | +0 | +1 |
| `ribbons` | +0 | +0 | +1 |
| `lights` | +0 | +1 | +1 |
| `flags` | -1 | +0 | +1 |

> Wing tip decorations all add style but streamers and flags create drag (-1 lift).
> Lights add weight from the battery pack.

### Nose (slot: `nose`)

| Decoration | Lift | Weight | Style |
|-----------|------|--------|-------|
| `mascot` | +0 | +1 | +1 |
| `eyes` | +0 | +0 | +1 |
| `horn` | +0 | +0 | +1 |
| `figurehead` | -1 | +1 | +1 |

> Nose decorations add flair. Heavier options (mascot, figurehead) add weight.
> Figurehead's carved shape catches wind (-1 lift).

### Tail (slot: `tail`)

| Decoration | Lift | Weight | Style |
|-----------|------|--------|-------|
| `tail_flag` | +0 | +0 | +1 |
| `smoke_can` | +0 | +1 | +1 |
| `parachute` | +1 | +1 | +0 |
| `cape` | +1 | +0 | +1 |

> Tail decorations are the most varied. Parachute adds lift (drag stabilization)
> and weight but no style. Cape adds both lift (catches air) and style. Smoke
> canister is heavy but looks amazing.

---

## How Stats Affect Gameplay

### Lift -- Flight Lift Coefficient

```javascript
// Higher lift = more upward force during flight
const liftCoefficient = 0.3 + (craft.stats.lift - 1) * 0.1;
// Lift 1 = 0.3 (barely glides)
// Lift 2 = 0.4
// Lift 3 = 0.5 (decent glide)
// Lift 4 = 0.6
// Lift 5 = 0.7 (soars like a bird)
```

Lift directly scales the upward force applied to the craft during flight.
Higher lift means a flatter glide angle, longer hang time, and more distance.

### Weight -- Launch Speed & Drag

```javascript
// Higher weight = slower launch speed off the ramp
const speedMultiplier = 1.0 - (craft.stats.weight - 1) * 0.08;
// Weight 1 = 1.00 (full speed)
// Weight 2 = 0.92
// Weight 3 = 0.84
// Weight 4 = 0.76
// Weight 5 = 0.68 (sluggish launch)

const baseRampSpeed = 12.0; // m/s
const launchSpeed = baseRampSpeed * speedMultiplier;

// Weight also adds drag during flight
const dragCoefficient = 0.02 + (craft.stats.weight - 1) * 0.005;
// Weight 1 = 0.020
// Weight 5 = 0.040
```

Heavy crafts launch slower and decelerate faster in the air. This is the primary
distance killer -- a Weight 5 craft needs exceptional lift to compensate.

### Style -- Showmanship Score Multiplier

```javascript
// Higher style = bigger multiplier on total score
const styleMultiplier = 1.0 + (craft.stats.style - 1) * 0.25;
// Style 1 = 1.00x (no bonus)
// Style 2 = 1.25x
// Style 3 = 1.50x
// Style 4 = 1.75x
// Style 5 = 2.00x (double score!)

const distanceScore = distanceFlown * 10; // 10 points per meter
const trickScore = tricksPerformed * 50;  // 50 points per trick
const collectibleScore = collectiblesGathered;
const totalScore = (distanceScore + trickScore + collectibleScore) * styleMultiplier;
```

Style is a pure score multiplier. It does not affect flight physics at all.
A Style 5 craft doubles all points, making it the optimal strategy for
high scores (if you can stay in the air long enough).

---

## Example Builds

### The Balanced Flyer

> Goal: Decent at everything, no weaknesses.

| Part | Choice | Reasoning |
|------|--------|-----------|
| Fuselage | `bathtub` | 2/3/3 base, no extremes |
| Wings | `delta` | +1/+0/+1, clean and balanced |
| Propulsion | `rubber_band` | +0/+0/+1, light and stylish |
| Side deco | `team_banner` | +0/+0/+1 |
| Wing tip | `ribbons` | +0/+0/+1 |
| Nose | `eyes` | +0/+0/+1 |
| Tail | `cape` | +1/+0/+1 |

**Final stats: Lift 4, Weight 3, Style 5** (clamped from raw 4/3/8)

A great all-around build. Style gets clamped to 5 easily so extra style
decorations are wasted -- but it maximizes the score multiplier while maintaining
decent flight characteristics.

---

### The Distance Machine

> Goal: Maximum distance. Fly as far as possible.

| Part | Choice | Reasoning |
|------|--------|-----------|
| Fuselage | `rocket` | 4/2/3 base, best lift |
| Wings | `hang_glider` | +2/+0/-1, max lift no weight |
| Propulsion | `none` | +0/-1/+0, lightest option |
| Side deco | `number` | +0/+0/+0, no drag |
| Wing tip | `ribbons` | +0/+0/+1, no lift penalty |
| Nose | `eyes` | +0/+0/+1 |
| Tail | `cape` | +1/+0/+1 |

**Final stats: Lift 5 (clamped from 7), Weight 1, Style 5 (clamped from 5)**

This build maxes lift and minimizes weight for the longest possible flight.
Raw lift of 7 gets clamped to 5, so there is room to swap `hang_glider` for
`biplane` without losing effective lift (biplane: raw 6 still clamps to 5, and
only +1 weight). Use the extra build room for style or fun.

---

### The Style King

> Goal: Maximum score via style multiplier. Distance secondary.

| Part | Choice | Reasoning |
|------|--------|-----------|
| Fuselage | `rubber_duck` | 1/2/5 base, max style |
| Wings | `cardboard_box` | +0/-1/+2, comedy gold |
| Propulsion | `rubber_band` | +0/+0/+1, quirky |
| Side deco | `flame_paint` | +0/+0/+1 |
| Wing tip | `streamers` | -1/+0/+1 |
| Nose | `horn` | +0/+0/+1 |
| Tail | `smoke_can` | +0/+1/+1 |

**Final stats: Lift 1 (clamped from 0), Weight 2, Style 5 (clamped from 12)**

This craft can barely fly but the crowd goes absolutely wild. With Style 5
giving a 2.0x multiplier, even a short flight scores well. The rubber duck with
cardboard wings, streamers, flames, a party horn, and trailing smoke is peak
Flugtag energy.

---

### The Heavy Hitter

> Goal: Maximum spectacle. Go big or go home.

| Part | Choice | Reasoning |
|------|--------|-----------|
| Fuselage | `piano` | 1/5/4 base, dramatic |
| Wings | `biplane` | +2/+1/+0, need all the lift |
| Propulsion | `pedal` | +1/+1/+0, sustained boost |
| Side deco | `stars` | +0/+0/+1 |
| Wing tip | `lights` | +0/+1/+1 |
| Nose | `figurehead` | -1/+1/+1 |
| Tail | `parachute` | +1/+1/+0 |

**Final stats: Lift 4, Weight 5 (clamped from 10), Style 5 (clamped from 7)**

A grand piano with biplane wings, fairy lights, a figurehead, and a parachute.
Absurdly heavy (Weight 5 = 0.68x launch speed) but Lift 4 gives decent flight.
The parachute adds lift for distance, and the style multiplier is maxed. Pure
Flugtag insanity.

---

## Stat Lookup Tables (Code)

Copy-paste these tables into the game code for stat calculation.

```javascript
const FUSELAGE_STATS = {
  bathtub:       { lift: 2, weight: 3, style: 3 },
  shopping_cart: { lift: 2, weight: 1, style: 2 },
  rubber_duck:   { lift: 1, weight: 2, style: 5 },
  piano:         { lift: 1, weight: 5, style: 4 },
  boat:          { lift: 3, weight: 3, style: 2 },
  rocket:        { lift: 4, weight: 2, style: 3 }
};

const WING_MODIFIERS = {
  biplane:       { lift: 2,  weight: 1,  style: 0 },
  delta:         { lift: 1,  weight: 0,  style: 1 },
  hang_glider:   { lift: 2,  weight: 0,  style: -1 },
  cardboard_box: { lift: 0,  weight: -1, style: 2 },
  umbrella:      { lift: 1,  weight: 0,  style: 1 }
};

const PROPULSION_MODIFIERS = {
  rubber_band: { lift: 0, weight: 0,  style: 1 },
  fan:         { lift: 0, weight: 1,  style: 0 },
  pedal:       { lift: 1, weight: 1,  style: 0 },
  none:        { lift: 0, weight: -1, style: 0 }
};

const DECORATION_NUDGES = {
  fuselage_side: {
    team_banner: { lift: 0, weight: 0, style: 1 },
    sponsor_logo: { lift: 0, weight: 0, style: 1 },
    flame_paint: { lift: 0, weight: 0, style: 1 },
    number:      { lift: 0, weight: 0, style: 0 },
    stars:       { lift: 0, weight: 0, style: 1 }
  },
  wing_tip: {
    streamers: { lift: -1, weight: 0, style: 1 },
    ribbons:   { lift: 0,  weight: 0, style: 1 },
    lights:    { lift: 0,  weight: 1, style: 1 },
    flags:     { lift: -1, weight: 0, style: 1 }
  },
  nose: {
    mascot:     { lift: 0,  weight: 1, style: 1 },
    eyes:       { lift: 0,  weight: 0, style: 1 },
    horn:       { lift: 0,  weight: 0, style: 1 },
    figurehead: { lift: -1, weight: 1, style: 1 }
  },
  tail: {
    tail_flag:  { lift: 0, weight: 0, style: 1 },
    smoke_can:  { lift: 0, weight: 1, style: 1 },
    parachute:  { lift: 1, weight: 1, style: 0 },
    cape:       { lift: 1, weight: 0, style: 1 }
  }
};

// Propulsion boost values (applied during flight, not stat calculation)
const PROPULSION_BOOST = {
  rubber_band: { duration: 2.0, force: 0.3 },
  fan:         { duration: 3.0, force: 0.5 },
  pedal:       { duration: 5.0, force: 0.4 },
  none:        { duration: 0.0, force: 0.0 }
};

/**
 * Calculate final clamped stats for a craft configuration.
 * @param {Object} craft - Craft data model (see SKILL.md)
 * @returns {{ lift: number, weight: number, style: number }}
 */
function calculateStats(craft) {
  const f = FUSELAGE_STATS[craft.fuselage.type];
  const w = WING_MODIFIERS[craft.wings.type];
  const p = PROPULSION_MODIFIERS[craft.propulsion.type];

  let lift   = f.lift   + w.lift   + p.lift;
  let weight = f.weight + w.weight + p.weight;
  let style  = f.style  + w.style  + p.style;

  // Add decoration nudges
  for (const deco of craft.decorations) {
    if (!deco.enabled) continue;
    const nudge = DECORATION_NUDGES[deco.slot]?.[deco.type];
    if (nudge) {
      lift   += nudge.lift   || 0;
      weight += nudge.weight || 0;
      style  += nudge.style  || 0;
    }
  }

  return {
    lift:   Math.max(1, Math.min(5, Math.round(lift))),
    weight: Math.max(1, Math.min(5, Math.round(weight))),
    style:  Math.max(1, Math.min(5, Math.round(style)))
  };
}
```
