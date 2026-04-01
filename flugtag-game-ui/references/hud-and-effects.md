# HUD & Effects Reference

Complete reference for all HUD components, floating text, visual effects, and
screen overlays in the Flugtag game.

---

## Table of Contents

1. [Flight HUD Components](#flight-hud-components)
2. [Ramp HUD Components](#ramp-hud-components)
3. [Floating Text System](#floating-text-system)
4. [Splash Effect Overlay](#splash-effect-overlay)
5. [Crowd Reaction System](#crowd-reaction-system)
6. [Results Screen Effects](#results-screen-effects)
7. [Confetti System](#confetti-system)
8. [Distance Milestone System](#distance-milestone-system)
9. [Effect Utilities](#effect-utilities)

---

## Flight HUD Components

The flight HUD is a non-interactive overlay (`pointer-events: none`) displayed during
active flight. Only the pause button captures input.

### Layout

```
+-----------------------------------------------------+
| [ALT |||...  18m]    DIST: 247m              [| |]  |
|                         meters                       |
|                                         WIND ->      |
|                                                      |
|              (3D flight viewport)                    |
|                                                      |
|                                                      |
|  SPD ||||||||...  42      [thermal glow border]  ^   |
+-----------------------------------------------------+
```

### Altitude Meter

Vertical-styled horizontal bar on the top left. Color-coded by altitude zone.

| Altitude Range | Color | CSS Class | Meaning |
|---------------|-------|-----------|---------|
| 0 - 10m | Red (#FF4444) | `alt-low` | Dangerously low, about to splash |
| 10 - 50m | Green (#44CC44) | `alt-mid` | Safe flying altitude |
| 50m+ | Blue (#4A90D9) | `alt-high` | High altitude, catching thermals |

```javascript
function updateAltitude(altitude, maxPossible) {
  const fill = document.querySelector('.hud-alt-bar-fill');
  const label = document.querySelector('.hud-alt-value');

  const pct = Math.min((altitude / maxPossible) * 100, 100);
  fill.style.width = pct + '%';

  // Color coding
  fill.classList.remove('alt-low', 'alt-mid', 'alt-high');
  if (altitude < 10) fill.classList.add('alt-low');
  else if (altitude < 50) fill.classList.add('alt-mid');
  else fill.classList.add('alt-high');

  label.textContent = Math.round(altitude) + 'm';

  // Flash red when critically low
  if (altitude < 5) {
    fill.style.animation = 'altFlash 0.5s ease-in-out infinite';
  } else {
    fill.style.animation = '';
  }
}
```

```css
@keyframes altFlash {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}
```

### Distance Counter

The primary metric. Large centered counter at the top of the HUD.

```javascript
function updateDistance(distance) {
  const el = document.querySelector('.hud-dist-value');
  el.textContent = Math.round(distance);

  // Milestone check
  checkDistanceMilestone(distance);
}
```

```css
.hud-dist-value {
  font-family: var(--font-counter);
  font-weight: 700;
  font-size: 2.5rem;
  color: var(--yellow);
  text-shadow: 3px 3px 0 rgba(0,0,0,0.5);
  line-height: 1;
  transition: transform 0.1s ease;
}

/* Bump animation when distance ticks over milestones */
.hud-dist-value.bump {
  animation: distBump 0.3s ease-out;
}

@keyframes distBump {
  0% { transform: scale(1); }
  50% { transform: scale(1.15); }
  100% { transform: scale(1); }
}
```

### Speed Indicator

Horizontal bar at the bottom left showing current speed.

```javascript
function updateSpeed(speed, maxSpeed) {
  const fill = document.querySelector('.hud-speed-bar-fill');
  const label = document.querySelector('.hud-speed-value');

  const pct = Math.min((speed / maxSpeed) * 100, 100);
  fill.style.width = pct + '%';
  label.textContent = Math.round(speed);

  // Change color at high speed
  if (speed > maxSpeed * 0.8) {
    fill.style.background = 'var(--yellow)';
  } else {
    fill.style.background = 'var(--green)';
  }
}
```

### Pitch Direction Indicator

A subtle arrow in the bottom right showing the craft's pitch angle.

```javascript
function updatePitch(pitchDegrees) {
  const arrow = document.querySelector('.hud-pitch-arrow');

  if (pitchDegrees > 5) {
    arrow.innerHTML = '&#8593;'; // Up
    arrow.style.color = 'rgba(68, 204, 68, 0.6)'; // green tint
  } else if (pitchDegrees < -5) {
    arrow.innerHTML = '&#8595;'; // Down
    arrow.style.color = 'rgba(255, 68, 68, 0.6)'; // red tint
  } else {
    arrow.innerHTML = '&#8594;'; // Level
    arrow.style.color = 'rgba(255, 255, 255, 0.4)';
  }

  // Rotate proportionally to pitch angle (clamped)
  const rotation = Math.max(-45, Math.min(45, -pitchDegrees));
  arrow.style.transform = `rotate(${rotation}deg)`;
}
```

```css
.hud-pitch {
  width: 44px;
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.hud-pitch-arrow {
  font-size: 1.8rem;
  color: rgba(255, 255, 255, 0.4);
  transition: transform 0.2s ease, color 0.3s ease;
}
```

### Pause Button

Top right, 44x44px minimum touch target. Only HUD element with `pointer-events: auto`.

```html
<button class="hud-pause-btn touch-target" style="pointer-events:auto;">
  <span class="hud-pause-icon">| |</span>
</button>
```

```css
.hud-pause-btn {
  width: 44px;
  height: 44px;
  background: rgba(0, 0, 0, 0.4);
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 10px;
  color: white;
  font-size: 1.2rem;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: auto;
  transition: background 0.15s ease;
}

.hud-pause-btn:active {
  background: rgba(255, 255, 255, 0.2);
}
```

### Thermal Proximity Indicator

A glowing border effect when the craft is near a thermal column.

```javascript
function updateThermalProximity(distance, maxRange) {
  const glow = document.querySelector('.hud-thermal-glow');
  if (!glow) return;

  if (distance < maxRange) {
    glow.style.display = 'block';
    // Intensity scales with proximity
    const intensity = 1 - (distance / maxRange);
    glow.style.opacity = intensity;
  } else {
    glow.style.display = 'none';
  }
}
```

```css
.hud-thermal-glow {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  pointer-events: none;
  animation: thermalPulse 1.2s ease-in-out infinite;
}

@keyframes thermalPulse {
  0%, 100% {
    box-shadow: inset 0 0 30px rgba(255, 165, 0, 0);
    border: 3px solid rgba(255, 165, 0, 0);
  }
  50% {
    box-shadow: inset 0 0 60px rgba(255, 165, 0, 0.2);
    border: 3px solid rgba(255, 165, 0, 0.4);
  }
}
```

### Wind Direction Indicator

Small arrow near the top right showing wind direction.

```javascript
function updateWindIndicator(windAngleDegrees, windStrength) {
  const arrow = document.querySelector('.hud-wind-arrow');
  const label = document.querySelector('.hud-wind-label');

  if (arrow) {
    arrow.style.transform = `rotate(${windAngleDegrees}deg)`;
    // Stronger wind = more visible
    const opacity = 0.3 + (windStrength / 20) * 0.7;
    arrow.parentElement.style.opacity = Math.min(opacity, 1);
  }
}
```

### Complete HUD Update Function

Called every frame from `requestAnimationFrame`:

```javascript
function updateFlightHUD(state) {
  // state shape from flight engine:
  // {
  //   altitude: number,     // current altitude in meters
  //   distance: number,     // horizontal distance from ramp
  //   speed: number,        // current speed
  //   maxAltitude: number,  // peak altitude this flight
  //   pitchAngle: number,   // degrees, positive = nose up
  //   windDirection: number,// degrees, 0 = from left
  //   windStrength: number, // 0-20 scale
  //   nearThermal: boolean, // within thermal detection range
  //   thermalDist: number   // distance to nearest thermal
  // }

  updateAltitude(state.altitude, 100);
  updateDistance(state.distance);
  updateSpeed(state.speed, 80);
  updatePitch(state.pitchAngle);
  updateWindIndicator(state.windDirection, state.windStrength);
  updateThermalProximity(state.thermalDist || 999, 50);
}
```

---

## Ramp HUD Components

Displayed during the Ramp Run phase when the player taps to build launch power.

### Layout

```
+-----------------------------------------------------+
|                                                      |
|          (Side view of ramp with craft)              |
|                                                      |
|     SPEED: 42                                        |
|  +-------------------------------------------------+ |
|  | POWER ||||||||||||||||||||||||||................. | |
|  |       ^^^^^^^^^^^^^^^^^^^^^^^^                   | |
|  |       fill                   [PERFECT ZONE]      | |
|  +-------------------------------------------------+ |
|                                                      |
|               TAP TAP TAP!                           |
+-----------------------------------------------------+
```

### Power Meter

The core ramp interaction. A horizontal bar that fills with rapid tapping.

```javascript
class PowerMeter {
  constructor() {
    this.power = 0;
    this.maxPower = 100;
    this.fillRate = 4;       // per tap
    this.decayRate = 1.5;    // per 100ms
    this.perfectMin = 65;
    this.perfectMax = 80;

    this.trackEl = document.querySelector('.ramp-meter-track');
    this.fillEl = document.querySelector('.ramp-meter-fill');
    this.markerEl = document.querySelector('.ramp-meter-marker');
    this.perfectEl = document.querySelector('.ramp-meter-perfect-zone');
  }

  tap() {
    this.power = Math.min(this.power + this.fillRate, this.maxPower);
    this.render();

    // Visual feedback on tap
    this.fillEl.style.filter = 'brightness(1.3)';
    setTimeout(() => { this.fillEl.style.filter = ''; }, 80);
  }

  decay() {
    this.power = Math.max(this.power - this.decayRate, 0);
    this.render();
  }

  render() {
    const pct = (this.power / this.maxPower) * 100;
    this.fillEl.style.width = pct + '%';
    this.markerEl.style.left = pct + '%';
  }

  isPerfect() {
    return this.power >= this.perfectMin && this.power <= this.perfectMax;
  }

  getPower() {
    return this.power / this.maxPower; // 0-1 normalized
  }
}
```

### Power Meter CSS

```css
.ramp-meter-track {
  position: relative;
  width: 100%;
  height: 32px;
  background: rgba(255, 255, 255, 0.15);
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.3);
}

.ramp-meter-fill {
  position: absolute;
  top: 0; left: 0; bottom: 0;
  width: 0%;
  border-radius: 14px;
  background: linear-gradient(90deg,
    #FF3344 0%,       /* low power = red */
    #FF8800 30%,      /* building = orange */
    #FFD700 50%,      /* mid = yellow */
    #44CC44 70%,      /* good = green */
    #4A90D9 100%      /* max = blue */
  );
  transition: width 0.05s linear, filter 0.08s ease;
}

.ramp-meter-perfect-zone {
  position: absolute;
  top: 0; bottom: 0;
  left: 65%;
  width: 15%;
  background: rgba(255, 215, 0, 0.25);
  border-left: 2px dashed var(--yellow);
  border-right: 2px dashed var(--yellow);
  animation: perfectPulse 1s ease-in-out infinite;
}

@keyframes perfectPulse {
  0%, 100% { background: rgba(255, 215, 0, 0.15); }
  50% { background: rgba(255, 215, 0, 0.35); }
}

.ramp-meter-marker {
  position: absolute;
  top: -4px; bottom: -4px;
  width: 4px;
  left: 0%;
  background: white;
  border-radius: 2px;
  box-shadow: 0 0 8px rgba(255, 255, 255, 0.8);
  transition: left 0.05s linear;
}
```

### "TAP!" Instruction Text

Bouncing text that encourages rapid tapping.

```css
.ramp-tap-text {
  font-family: var(--font-display);
  font-size: 2.2rem;
  color: var(--yellow);
  text-shadow:
    2px 2px 0 rgba(0,0,0,0.5),
    0 0 15px rgba(255,215,0,0.3);
  animation: tapBounce 0.4s ease-in-out infinite alternate;
  display: inline-block;
}

@keyframes tapBounce {
  0% { transform: translateY(0) scale(1); }
  100% { transform: translateY(-8px) scale(1.08); }
}

/* Individual letter stagger for extra energy */
.ramp-tap-text .letter {
  display: inline-block;
  animation: letterWave 0.6s ease-in-out infinite;
}

.ramp-tap-text .letter:nth-child(2) { animation-delay: 0.05s; }
.ramp-tap-text .letter:nth-child(3) { animation-delay: 0.1s; }
.ramp-tap-text .letter:nth-child(4) { animation-delay: 0.15s; }
.ramp-tap-text .letter:nth-child(5) { animation-delay: 0.2s; }

@keyframes letterWave {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-4px); }
}
```

### Speed Readout

Small counter showing current ramp speed based on power level.

```javascript
function updateRampSpeed(power) {
  const speedEl = document.querySelector('.ramp-speed-value');
  const speed = Math.round(power * 0.8); // power 0-100 maps to speed 0-80
  speedEl.textContent = speed;
}
```

---

## Floating Text System

Pop-up text overlays that appear during gameplay for feedback and rewards.

### Architecture

Floating text elements are appended to `#effects-layer` and auto-remove after animation.

```javascript
/**
 * Show a floating text popup.
 * @param {string} text - The text to display
 * @param {string} color - CSS color value
 * @param {object} options
 * @param {number} options.x - Horizontal position (0-1, default 0.5 = center)
 * @param {number} options.y - Vertical position (0-1, default 0.4)
 * @param {number} options.size - Font size in rem (default 1.5)
 * @param {number} options.duration - Animation duration in ms (default 1500)
 */
function showFloatingText(text, color = 'var(--yellow)', options = {}) {
  const layer = document.getElementById('effects-layer');
  if (!layer) return;

  const el = document.createElement('div');
  el.className = 'floating-text';
  el.textContent = text;
  el.style.color = color;
  el.style.left = ((options.x || 0.5) * 100) + '%';
  el.style.top = ((options.y || 0.4) * 100) + '%';
  el.style.fontSize = (options.size || 1.5) + 'rem';
  el.style.animationDuration = (options.duration || 1500) + 'ms';

  layer.appendChild(el);

  // Remove after animation
  setTimeout(() => {
    if (el.parentNode) el.parentNode.removeChild(el);
  }, options.duration || 1500);
}
```

### CSS

```css
.floating-text {
  position: absolute;
  font-family: var(--font-display);
  font-weight: 700;
  text-shadow: 2px 2px 0 rgba(0,0,0,0.6);
  pointer-events: none;
  transform: translate(-50%, -50%);
  white-space: nowrap;
  animation: floatUp 1.5s ease-out forwards;
  z-index: 25;
}

@keyframes floatUp {
  0% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.5) translateY(0);
  }
  15% {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1.2) translateY(0);
  }
  30% {
    transform: translate(-50%, -50%) scale(1) translateY(0);
  }
  100% {
    opacity: 0;
    transform: translate(-50%, -50%) scale(1) translateY(-80px);
  }
}
```

### Floating Text Triggers

| Trigger | Text | Color | Size | When |
|---------|------|-------|------|------|
| Perfect launch | "+500 PERFECT LAUNCH!" | `var(--yellow)` | 2rem | Launch in perfect zone |
| Thermal chain | "+200 THERMAL CHAIN!" | `var(--orange)` | 1.8rem | Catching sequential thermals |
| Bird warning | "WATCH OUT!" | `var(--danger)` | 2rem | Birds approaching |
| Collectible | "+100" | `var(--green)` | 1.5rem | Pick up floating item |
| Distance milestone | "100m!" / "200m!" etc | `var(--text-primary)` | 2.5rem | Every 100m |
| Trick bonus | "+150 BARREL ROLL!" | `var(--yellow)` | 1.8rem | Style trick performed |
| Low altitude | "PULL UP!" | `var(--danger)` | 2.2rem | Below 3m altitude |

```javascript
// Example trigger integrations

// Perfect launch (called from ramp system)
function onPerfectLaunch() {
  showFloatingText('+500 PERFECT LAUNCH!', 'var(--yellow)', { size: 2, y: 0.3 });
}

// Thermal catch (called from flight engine)
function onThermalCatch(chainCount) {
  const bonus = 100 + (chainCount * 100);
  const text = chainCount > 1 ? `+${bonus} THERMAL CHAIN x${chainCount}!` : `+${bonus} THERMAL!`;
  showFloatingText(text, 'var(--orange)', { size: 1.8, y: 0.35 });
}

// Bird warning (called from hazard system)
function onBirdApproach() {
  showFloatingText('WATCH OUT!', 'var(--danger)', { size: 2, y: 0.3, duration: 2000 });
}

// Collectible pickup
function onCollectible(value) {
  showFloatingText('+' + value, 'var(--green)', { size: 1.5, x: 0.6, y: 0.45 });
}

// Low altitude warning
function onLowAltitude() {
  showFloatingText('PULL UP!', 'var(--danger)', { size: 2.2, y: 0.5, duration: 2000 });
}
```

---

## Splash Effect Overlay

Brief visual feedback when the craft hits the water.

### Components

1. **"SPLASH!" text** -- Bouncy entrance animation
2. **Water burst** -- Radial blue burst expanding outward
3. **Screen shake** -- CSS transform jitter on the screen container
4. **Haptic feedback** -- Device vibration if available

### HTML

```html
<div id="screen-splash" class="screen">
  <div class="splash-content">
    <h1 class="splash-text">SPLASH!</h1>
    <div class="splash-water-burst"></div>
  </div>
</div>
```

### CSS

```css
.splash-content {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.splash-text {
  font-family: var(--font-display);
  font-size: 6rem;
  color: #B8E6F0;
  text-shadow:
    4px 4px 0 #0088AA,
    8px 8px 0 #005577,
    0 0 40px rgba(0, 136, 170, 0.6);
  animation: splashTextIn 0.6s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
  z-index: 2;
}

@keyframes splashTextIn {
  0% {
    transform: scale(0) rotate(-15deg);
    opacity: 0;
  }
  40% {
    transform: scale(1.3) rotate(5deg);
    opacity: 1;
  }
  70% {
    transform: scale(0.95) rotate(-2deg);
  }
  100% {
    transform: scale(1) rotate(0deg);
    opacity: 1;
  }
}

/* Radial water burst */
.splash-water-burst {
  position: absolute;
  width: 200px;
  height: 200px;
  border-radius: 50%;
  background: radial-gradient(
    circle,
    rgba(184, 230, 240, 0.5) 0%,
    rgba(0, 136, 170, 0.3) 30%,
    rgba(0, 85, 119, 0.1) 60%,
    transparent 80%
  );
  animation: waterBurstExpand 1.2s ease-out forwards;
  z-index: 1;
}

@keyframes waterBurstExpand {
  0% {
    transform: scale(0);
    opacity: 1;
  }
  60% {
    opacity: 0.6;
  }
  100% {
    transform: scale(4);
    opacity: 0;
  }
}

/* Water droplet particles */
.splash-droplet {
  position: absolute;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--water-foam);
  animation: dropletFly 1s ease-out forwards;
}

@keyframes dropletFly {
  0% {
    transform: translate(0, 0) scale(1);
    opacity: 1;
  }
  100% {
    transform: translate(var(--dx), var(--dy)) scale(0.3);
    opacity: 0;
  }
}

/* Screen shake */
@keyframes screenShake {
  0% { transform: translate(0, 0); }
  10% { transform: translate(-10px, 5px); }
  20% { transform: translate(8px, -8px); }
  30% { transform: translate(-6px, 3px); }
  40% { transform: translate(5px, -4px); }
  50% { transform: translate(-3px, 2px); }
  60% { transform: translate(2px, -2px); }
  70% { transform: translate(-1px, 1px); }
  100% { transform: translate(0, 0); }
}

#screen-splash.shake {
  animation: screenShake 0.5s ease-out;
}
```

### Splash Implementation

```javascript
function playSplashAnimation() {
  const screen = document.getElementById('screen-splash');

  // Screen shake
  screen.classList.add('shake');

  // Spawn water droplet particles
  spawnSplashDroplets(12);

  // Haptic feedback
  if (navigator.vibrate) {
    navigator.vibrate([50, 20, 100, 30, 50]);
  }

  // Auto-advance to results after 1.5s
  setTimeout(() => {
    screen.classList.remove('shake');

    // Gather flight results
    const flightData = window.FlightEngine
      ? window.FlightEngine.getResults()
      : { distance: 0, maxAltitude: 0, styleMultiplier: 1, score: 0 };

    screenManager.show('screen-results', flightData);
  }, 1500);
}

function spawnSplashDroplets(count) {
  const layer = document.getElementById('effects-layer');
  if (!layer) return;

  for (let i = 0; i < count; i++) {
    const droplet = document.createElement('div');
    droplet.className = 'splash-droplet';

    // Random direction
    const angle = (Math.PI * 2 * i) / count + (Math.random() - 0.5) * 0.5;
    const distance = 60 + Math.random() * 120;
    const dx = Math.cos(angle) * distance;
    const dy = Math.sin(angle) * distance - 40; // bias upward

    droplet.style.left = '50%';
    droplet.style.top = '50%';
    droplet.style.setProperty('--dx', dx + 'px');
    droplet.style.setProperty('--dy', dy + 'px');
    droplet.style.animationDelay = (Math.random() * 0.15) + 's';
    droplet.style.width = (4 + Math.random() * 8) + 'px';
    droplet.style.height = droplet.style.width;

    layer.appendChild(droplet);

    setTimeout(() => {
      if (droplet.parentNode) droplet.parentNode.removeChild(droplet);
    }, 1200);
  }
}
```

---

## Crowd Reaction System

Text overlays that reflect how the crowd reacts based on flight distance.

### Reaction Tiers

| Distance | Reaction Text | Color | Crowd Sound Mood |
|----------|--------------|-------|-----------------|
| 0 - 25m | "OH NO..." | `var(--danger)` | Sympathetic groan |
| 25 - 50m | "OHHH..." | `var(--text-secondary)` | Mild disappointment |
| 50 - 100m | "NOT BAD!" | `var(--text-primary)` | Polite applause |
| 100 - 200m | "NICE!" | `var(--green)` | Enthusiastic cheering |
| 200 - 350m | "WOOO!" | `var(--yellow)` | Loud cheering |
| 350 - 500m | "INCREDIBLE!" | `var(--yellow-glow)` | Roaring crowd |
| 500m+ | "LEGENDARY!" | `var(--gold)` | Absolute pandemonium |

### Implementation

```javascript
function getCrowdReaction(distance) {
  if (distance < 25)  return { text: 'OH NO...',     color: 'var(--danger)',         confetti: false };
  if (distance < 50)  return { text: 'OHHH...',      color: 'var(--text-secondary)', confetti: false };
  if (distance < 100) return { text: 'NOT BAD!',     color: 'var(--text-primary)',   confetti: false };
  if (distance < 200) return { text: 'NICE!',        color: 'var(--green)',          confetti: false };
  if (distance < 350) return { text: 'WOOO!',        color: 'var(--yellow)',         confetti: true  };
  if (distance < 500) return { text: 'INCREDIBLE!',  color: 'var(--yellow-glow)',    confetti: true  };
  return                      { text: 'LEGENDARY!',  color: 'var(--gold)',           confetti: true  };
}

function showCrowdReaction(distance) {
  const reaction = getCrowdReaction(distance);

  const el = document.querySelector('.crowd-reaction-text');
  if (el) {
    el.textContent = reaction.text;
    el.style.color = reaction.color;
    // Retrigger animation
    el.style.animation = 'none';
    el.offsetHeight;
    el.style.animation = 'reactionPop 0.5s cubic-bezier(0.34, 1.56, 0.64, 1)';
  }

  // Confetti for good flights
  if (reaction.confetti) {
    launchConfetti();
  }
}
```

### Crowd Reaction CSS

```css
.crowd-reaction-text {
  font-family: var(--font-display);
  font-size: 1.6rem;
  text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
  min-height: 32px;
  text-align: center;
}

@keyframes reactionPop {
  0% { transform: scale(0) rotate(-5deg); opacity: 0; }
  50% { transform: scale(1.2) rotate(2deg); opacity: 1; }
  100% { transform: scale(1) rotate(0deg); opacity: 1; }
}
```

---

## Results Screen Effects

### Score Counter Animation

Animated count-up from 0 to final value with ease-out curve.

```javascript
/**
 * Animate a counter from one value to another.
 * @param {HTMLElement} el - Element to update textContent
 * @param {number} from - Starting value
 * @param {number} to - Target value
 * @param {number} duration - Animation duration in ms
 * @param {Function} formatter - Function to format the displayed value
 * @returns {Promise} Resolves when animation completes
 */
function animateCounter(el, from, to, duration, formatter) {
  return new Promise(resolve => {
    const start = performance.now();

    function tick(now) {
      const elapsed = now - start;
      const progress = Math.min(elapsed / duration, 1);

      // Ease-out cubic for satisfying deceleration
      const eased = 1 - Math.pow(1 - progress, 3);
      const value = from + (to - from) * eased;

      el.textContent = formatter(value);

      if (progress < 1) {
        requestAnimationFrame(tick);
      } else {
        el.textContent = formatter(to); // ensure exact final value
        resolve();
      }
    }

    requestAnimationFrame(tick);
  });
}
```

### Medal Reveal

Medals scale in with rotation and a bounce effect.

```css
.results-medal {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
  margin: 8px 0;
}

.results-medal-icon {
  font-size: 3.5rem;
  display: inline-block;
  animation: medalReveal 0.6s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
}

@keyframes medalReveal {
  0% {
    transform: scale(0) rotate(-180deg);
    opacity: 0;
  }
  60% {
    transform: scale(1.2) rotate(10deg);
    opacity: 1;
  }
  100% {
    transform: scale(1) rotate(0deg);
    opacity: 1;
  }
}

/* Gold medal gets extra sparkle */
.results-medal.gold .results-medal-icon {
  color: var(--gold);
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.6);
  animation: medalReveal 0.6s cubic-bezier(0.34, 1.56, 0.64, 1) forwards,
             goldSparkle 2s ease-in-out infinite 0.6s;
}

@keyframes goldSparkle {
  0%, 100% { text-shadow: 0 0 20px rgba(255, 215, 0, 0.3); }
  50% { text-shadow: 0 0 40px rgba(255, 215, 0, 0.8), 0 0 60px rgba(255, 215, 0, 0.4); }
}

/* Silver */
.results-medal.silver .results-medal-icon {
  color: var(--silver);
  text-shadow: 0 0 15px rgba(192, 192, 192, 0.4);
}

/* Bronze */
.results-medal.bronze .results-medal-icon {
  color: var(--bronze);
  text-shadow: 0 0 15px rgba(205, 127, 50, 0.4);
}

.results-medal-label {
  font-family: var(--font-display);
  font-size: 1.2rem;
  text-transform: uppercase;
  letter-spacing: 3px;
}
```

### "NEW RECORD!" Banner

```css
.results-new-record {
  font-family: var(--font-display);
  font-size: 1.6rem;
  color: var(--yellow);
  text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
  padding: 10px 28px;
  background: linear-gradient(90deg,
    transparent 0%,
    rgba(255, 215, 0, 0.15) 20%,
    rgba(255, 215, 0, 0.25) 50%,
    rgba(255, 215, 0, 0.15) 80%,
    transparent 100%
  );
  border-top: 1px solid rgba(255, 215, 0, 0.3);
  border-bottom: 1px solid rgba(255, 215, 0, 0.3);
  animation: recordSlideIn 0.5s cubic-bezier(0.34, 1.56, 0.64, 1) forwards,
             recordGlow 1.5s ease-in-out infinite 0.5s alternate;
}

@keyframes recordSlideIn {
  0% { transform: scaleX(0); opacity: 0; }
  100% { transform: scaleX(1); opacity: 1; }
}

@keyframes recordGlow {
  0% {
    text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
    background-position: -100% 0;
  }
  100% {
    text-shadow: 2px 2px 0 rgba(0,0,0,0.5), 0 0 25px rgba(255,215,0,0.5);
    background-position: 100% 0;
  }
}
```

### Stat Bar Animation Sequence

Stats animate in one at a time with staggered delays:

```javascript
function animateResultsSequence(data) {
  const { distance, maxAltitude, styleMultiplier, score } = data;

  // Staggered animation timeline
  const timeline = [
    { el: '.results-distance',  from: 0, to: distance,         dur: 800,  fmt: v => Math.round(v) + 'm',        delay: 200  },
    { el: '.results-altitude',  from: 0, to: maxAltitude,      dur: 600,  fmt: v => Math.round(v) + 'm',        delay: 700  },
    { el: '.results-style',     from: 1, to: styleMultiplier,  dur: 400,  fmt: v => 'x' + v.toFixed(2),         delay: 1100 },
    { el: '.results-total',     from: 0, to: score,            dur: 1000, fmt: v => Math.round(v).toLocaleString(), delay: 1500 }
  ];

  timeline.forEach(({ el, from, to, dur, fmt, delay }) => {
    const element = document.querySelector(el);
    if (!element) return;

    setTimeout(() => {
      // Highlight row as it animates
      const row = element.closest('.results-stat-row');
      if (row) {
        row.style.background = 'rgba(255,255,255,0.05)';
        setTimeout(() => { row.style.background = ''; }, dur + 200);
      }
      animateCounter(element, from, to, dur, fmt);
    }, delay);
  });
}
```

---

## Confetti System

CSS-based confetti particles for celebrating good flights.

```javascript
function launchConfetti(count = 40) {
  const layer = document.getElementById('effects-layer');
  if (!layer) return;

  const colors = ['#FF3344', '#FFD700', '#4A90D9', '#44CC44', '#FF8800', '#FFFFFF'];

  for (let i = 0; i < count; i++) {
    const particle = document.createElement('div');
    particle.className = 'confetti-particle';

    // Random properties
    const color = colors[Math.floor(Math.random() * colors.length)];
    const x = 30 + Math.random() * 40; // spawn in center 40%
    const size = 6 + Math.random() * 8;
    const rotation = Math.random() * 360;
    const drift = (Math.random() - 0.5) * 200; // horizontal drift
    const duration = 2 + Math.random() * 2;
    const delay = Math.random() * 0.5;

    particle.style.cssText = `
      left: ${x}%;
      top: -10px;
      width: ${size}px;
      height: ${size * 0.6}px;
      background: ${color};
      transform: rotate(${rotation}deg);
      animation: confettiFall ${duration}s ease-in ${delay}s forwards;
      --drift: ${drift}px;
      --rotation: ${rotation + 720}deg;
    `;

    layer.appendChild(particle);

    setTimeout(() => {
      if (particle.parentNode) particle.parentNode.removeChild(particle);
    }, (duration + delay) * 1000 + 100);
  }
}
```

```css
.confetti-particle {
  position: absolute;
  border-radius: 2px;
  pointer-events: none;
  z-index: 30;
}

@keyframes confettiFall {
  0% {
    transform: translateX(0) translateY(0) rotate(0deg);
    opacity: 1;
  }
  25% {
    opacity: 1;
  }
  100% {
    transform: translateX(var(--drift)) translateY(110vh) rotate(var(--rotation));
    opacity: 0;
  }
}
```

---

## Distance Milestone System

Shows distance milestones during flight at every 100m interval.

```javascript
let lastMilestone = 0;

function checkDistanceMilestone(distance) {
  const milestone = Math.floor(distance / 100) * 100;

  if (milestone > lastMilestone && milestone > 0) {
    lastMilestone = milestone;

    // Big floating text
    showFloatingText(milestone + 'm!', 'var(--text-primary)', {
      size: 2.5,
      y: 0.35,
      duration: 2000
    });

    // Bump the distance counter
    const distEl = document.querySelector('.hud-dist-value');
    if (distEl) {
      distEl.classList.add('bump');
      setTimeout(() => distEl.classList.remove('bump'), 300);
    }

    // Extra celebration at big milestones
    if (milestone >= 300) {
      showFloatingText('AMAZING!', 'var(--yellow)', {
        size: 1.5,
        y: 0.42,
        duration: 1500
      });
    }
  }
}

function resetMilestones() {
  lastMilestone = 0;
}
```

---

## Effect Utilities

### Clear All Effects

```javascript
function clearAllEffects() {
  const layer = document.getElementById('effects-layer');
  if (layer) layer.innerHTML = '';
  resetMilestones();
}
```

### Disable Effects (for low-end devices)

```javascript
let effectsEnabled = true;

function setEffectsEnabled(enabled) {
  effectsEnabled = enabled;
  const layer = document.getElementById('effects-layer');
  if (layer) {
    layer.style.display = enabled ? 'block' : 'none';
  }
}

// Wrap showFloatingText to respect setting
const _showFloatingText = showFloatingText;
showFloatingText = function(...args) {
  if (!effectsEnabled) return;
  _showFloatingText(...args);
};
```

### Screen Flash Effect

Brief color flash for impacts or transitions.

```javascript
function screenFlash(color = 'white', duration = 200) {
  const layer = document.getElementById('effects-layer');
  if (!layer) return;

  const flash = document.createElement('div');
  flash.className = 'screen-flash';
  flash.style.background = color;
  flash.style.animationDuration = duration + 'ms';

  layer.appendChild(flash);
  setTimeout(() => {
    if (flash.parentNode) flash.parentNode.removeChild(flash);
  }, duration);
}
```

```css
.screen-flash {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  pointer-events: none;
  animation: flashFade linear forwards;
  z-index: 30;
}

@keyframes flashFade {
  0% { opacity: 0.6; }
  100% { opacity: 0; }
}
```
