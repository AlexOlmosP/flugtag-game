# Screen Flows Reference

Complete screen-by-screen specifications for every UI screen in the Flugtag game.

---

## Table of Contents

1. [Navigation Map](#navigation-map)
2. [Transition Matrix](#transition-matrix)
3. [Title Screen](#title-screen)
4. [Workshop Screen](#workshop-screen)
5. [Course Select Screen](#course-select-screen)
6. [Ramp Run Screen](#ramp-run-screen)
7. [Flight HUD Screen](#flight-hud-screen)
8. [Splash Screen](#splash-screen)
9. [Results Screen](#results-screen)
10. [Pause Screen](#pause-screen)
11. [ScreenManager Wiring](#screenmanager-wiring)

---

## Navigation Map

```
                    +===========+
                    |   TITLE   |
                    | "TAP TO   |
                    |   FLY"    |
                    +===========+
                         |
                    tap / click
                         |
                         v
                  +--------------+
              +-->|   WORKSHOP   |<---------+
              |   | (craft build)|          |
              |   +--------------+          |
              |        |                    |
              |   "LAUNCH!" btn             |
              |        |                    |
              |        v                    |
              |   +-----------+             |
              |   |  COURSE   | (optional)  |
              |   |  SELECT   |             |
              |   +-----------+             |
              |        |                    |
              |   select course             |
              |        |                    |
              |        v                    |
              |   +-----------+             |
              |   |  RAMP RUN |             |
              |   | (power    |             |
              |   |  meter)   |             |
              |   +-----------+             |
              |        |                    |
              |   launch                    |
              |        |                    |
              |        v                    |
              |   +-----------+     +-------+---+
              |   |  FLIGHT   |---->|   PAUSE   |
              |   |  (HUD)    |<----|  resume / |
              |   +-----------+     |  workshop |
              |        |            +-----------+
              |   land in water          |
              |        |                 |
              |        v            "Workshop"
              |   +-----------+     btn goes to
              |   |  SPLASH   |     Workshop
              |   |  (1.5s)   |
              |   +-----------+
              |        |
              |   auto-advance
              |        |
              |        v
              |   +-----------+
              |   |  RESULTS  |
              |   | score,    |
              |   | medals    |
              +---|  buttons  |
         "Workshop"+-----------+
          button        |
                   "Fly Again"
                   loops to Ramp Run
```

---

## Transition Matrix

| From | To | Transition | Duration | Trigger |
|------|----|-----------|----------|---------|
| Title | Workshop | fade | 300ms | Tap anywhere / "TAP TO FLY" |
| Workshop | Course Select | pop | 350ms | "LAUNCH!" button |
| Workshop | Ramp Run | fade | 300ms | "LAUNCH!" (if single course) |
| Course Select | Ramp Run | fade | 300ms | Course card tap |
| Ramp Run | Flight HUD | fade | 300ms | Launch event from ramp |
| Flight HUD | Pause | pop | 350ms | Pause button tap |
| Pause | Flight HUD | pop | 350ms | "Resume" button |
| Pause | Workshop | fade | 300ms | "Workshop" button |
| Flight HUD | Splash | none | 0ms | Water collision |
| Splash | Results | slide-up | 400ms | Auto after 1.5s |
| Results | Ramp Run | fade | 300ms | "Fly Again" button |
| Results | Workshop | slide-up | 400ms | "Workshop" button |

---

## Title Screen

### HTML Template

```html
<div id="screen-title" class="screen">
  <div class="title-content">
    <!-- Logo -->
    <div class="title-logo">
      <h1 class="title-text title-flugtag">FLUGTAG!</h1>
      <p class="subtitle-text title-tagline">Build. Launch. Fly!</p>
    </div>

    <!-- CTA -->
    <button class="game-btn game-btn-gold game-btn-pulse title-cta">
      TAP TO FLY
    </button>

    <!-- Best distance badge (if exists) -->
    <div class="title-best" style="display:none;">
      <span class="stat-label">BEST FLIGHT</span>
      <span class="stat-value title-best-value">0m</span>
    </div>
  </div>

  <!-- Version / credits -->
  <div class="title-footer">
    <span class="title-version">v0.1</span>
  </div>
</div>
```

### CSS

```css
#screen-title .title-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 24px;
  padding: 20px;
}

.title-flugtag {
  font-family: var(--font-display);
  font-size: 5rem;
  color: var(--yellow);
  text-shadow:
    4px 4px 0 var(--red),
    6px 6px 0 rgba(0,0,0,0.4),
    0 0 40px rgba(255,215,0,0.4);
  letter-spacing: 4px;
  animation: titleBob 3s ease-in-out infinite;
}

@keyframes titleBob {
  0%, 100% { transform: translateY(0) rotate(-1deg); }
  50% { transform: translateY(-8px) rotate(1deg); }
}

.title-tagline {
  font-size: 1.3rem;
  color: var(--text-secondary);
  margin-top: -8px;
}

.title-cta {
  font-size: 1.5rem;
  padding: 18px 48px;
  margin-top: 20px;
}

.title-best {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  margin-top: 16px;
  padding: 12px 24px;
  background: var(--panel-bg);
  border-radius: 12px;
  border: 1px solid var(--panel-border);
}

.title-footer {
  position: absolute;
  bottom: 16px;
  left: 0; right: 0;
  text-align: center;
  font-family: var(--font-body);
  font-size: 0.7rem;
  color: rgba(255,255,255,0.3);
}
```

### Enter / Exit

```javascript
function startTitleAnimation() {
  // Show best distance if available
  const best = gameData.getBestDistance('default');
  const bestEl = document.querySelector('.title-best');
  const bestVal = document.querySelector('.title-best-value');
  if (best > 0 && bestEl && bestVal) {
    bestEl.style.display = 'flex';
    bestVal.textContent = Math.round(best) + 'm';
  }
}

function stopTitleAnimation() {
  // Cleanup if needed
}
```

### 3D Scene Behind

- Craft sitting on the pier/ramp, facing the ocean
- Sunset sky with warm golden lighting
- Gentle waves, birds in distance
- Camera slowly orbits around the craft
- Crowd milling about on the pier

### Interactive Elements

- **"TAP TO FLY" button**: Navigates to Workshop screen
- **Anywhere tap** (optional): Also navigates to Workshop

---

## Workshop Screen

The Workshop screen is primarily managed by the **craft-builder skill**. This document
covers the wrapper and navigation elements that the UI skill provides.

### HTML Template

```html
<div id="screen-workshop" class="screen">
  <!-- Top bar with navigation -->
  <div class="workshop-topbar">
    <button class="game-btn game-btn-secondary workshop-back-btn touch-target">
      BACK
    </button>
    <h2 class="subtitle-text">WORKSHOP</h2>
    <div style="width:80px;"></div> <!-- spacer -->
  </div>

  <!-- Craft builder content (populated by builder skill) -->
  <div id="workshop-builder-area">
    <!-- Builder skill injects part picker, 3D preview, stat bars here -->
  </div>

  <!-- Bottom bar with launch button -->
  <div class="workshop-bottombar">
    <div class="workshop-stats-summary">
      <span class="stat-label">LIFT</span>
      <span class="workshop-stat-lift stat-value">3</span>
      <span class="stat-label">WEIGHT</span>
      <span class="workshop-stat-weight stat-value">3</span>
      <span class="stat-label">STYLE</span>
      <span class="workshop-stat-style stat-value">4</span>
    </div>
    <button class="game-btn game-btn-primary game-btn-pulse workshop-launch-btn">
      LAUNCH!
    </button>
  </div>
</div>
```

### CSS

```css
.workshop-topbar {
  position: absolute;
  top: 0; left: 0; right: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: linear-gradient(180deg, rgba(0,0,0,0.6) 0%, transparent 100%);
  z-index: 2;
}

.workshop-bottombar {
  position: absolute;
  bottom: 0; left: 0; right: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px;
  background: linear-gradient(0deg, rgba(0,0,0,0.7) 0%, transparent 100%);
  z-index: 2;
}

.workshop-stats-summary {
  display: flex;
  gap: 12px;
  align-items: center;
}

.workshop-stats-summary .stat-label {
  font-size: 0.7rem;
}

.workshop-stats-summary .stat-value {
  font-size: 1.2rem;
  margin-right: 8px;
}

.workshop-launch-btn {
  font-size: 1.3rem;
  padding: 14px 40px;
}
```

### Enter / Exit

```javascript
function initWorkshop() {
  // Load saved craft data
  const craft = gameData.getCraft();
  // Initialize builder skill with craft data
  if (window.CraftBuilder) {
    window.CraftBuilder.init(craft);
  }
  updateWorkshopStats(craft.stats);
}

function saveCurrentCraft() {
  if (window.CraftBuilder) {
    const craft = window.CraftBuilder.getCraft();
    gameData.saveCraft(craft);
  }
}

function updateWorkshopStats(stats) {
  document.querySelector('.workshop-stat-lift').textContent = stats.lift;
  document.querySelector('.workshop-stat-weight').textContent = stats.weight;
  document.querySelector('.workshop-stat-style').textContent = stats.style;
}
```

### 3D Scene Behind

- Workshop / hangar environment
- Craft on a platform, rotatable by drag
- Good lighting to see craft details

### Interactive Elements

- **"BACK" button**: Returns to Title screen
- **"LAUNCH!" button**: Navigates to Course Select (or Ramp Run if single course)
- **Builder area**: Full interaction handled by craft-builder skill

---

## Course Select Screen

### HTML Template

```html
<div id="screen-courses" class="screen">
  <div class="courses-panel">
    <h2 class="title-text" style="font-size:2.5rem;">SELECT COURSE</h2>

    <div class="courses-grid">
      <!-- Generated dynamically -->
    </div>

    <button class="game-btn game-btn-secondary courses-back-btn">
      BACK
    </button>
  </div>
</div>
```

### Course Card Template (generated per course)

```html
<div class="course-card" data-course="pier">
  <div class="course-card-thumb">
    <div class="course-card-icon">PIER</div>
  </div>
  <div class="course-card-info">
    <h3 class="course-card-name">Seaside Pier</h3>
    <p class="course-card-desc">Calm waters, gentle winds</p>
    <div class="course-card-best">
      <span class="stat-label">BEST</span>
      <span class="stat-value">247m</span>
    </div>
    <div class="course-card-medals">
      <span class="medal medal-gold">&#9733;</span>
      <span class="medal medal-silver">&#9733;</span>
      <span class="medal medal-bronze inactive">&#9733;</span>
    </div>
  </div>
</div>

<!-- Locked course variant -->
<div class="course-card course-locked" data-course="cliff">
  <div class="course-card-thumb">
    <div class="course-lock-icon">&#128274;</div>
  </div>
  <div class="course-card-info">
    <h3 class="course-card-name">Windy Cliff</h3>
    <p class="course-card-desc">Reach 200m in Pier to unlock</p>
  </div>
</div>
```

### CSS

```css
.courses-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  padding: 24px;
  max-height: 90vh;
  overflow-y: auto;
}

.courses-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 16px;
  width: 100%;
  max-width: 640px;
}

.course-card {
  background: var(--panel-bg);
  border: 2px solid var(--panel-border);
  border-radius: 16px;
  padding: 16px;
  cursor: pointer;
  transition: transform 0.15s ease, border-color 0.15s ease;
  display: flex;
  gap: 16px;
  align-items: center;
}

.course-card:hover {
  transform: scale(1.02);
  border-color: var(--yellow);
}

.course-card:active {
  transform: scale(0.98);
}

.course-locked {
  opacity: 0.5;
  pointer-events: none;
}

.course-card-thumb {
  width: 80px;
  height: 80px;
  background: rgba(255,255,255,0.1);
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-family: var(--font-display);
  font-size: 0.9rem;
  color: var(--text-secondary);
  flex-shrink: 0;
}

.course-card-name {
  font-family: var(--font-display);
  font-size: 1.3rem;
  color: var(--text-primary);
}

.course-card-desc {
  font-family: var(--font-body);
  font-size: 0.8rem;
  color: var(--text-secondary);
  margin-top: 2px;
}

.course-card-best {
  display: flex;
  gap: 8px;
  align-items: baseline;
  margin-top: 4px;
}

.course-card-best .stat-value {
  font-size: 1.1rem;
}

.course-card-medals {
  display: flex;
  gap: 4px;
  margin-top: 4px;
}

.medal {
  font-size: 1.2rem;
}

.medal-gold { color: var(--gold); }
.medal-silver { color: var(--silver); }
.medal-bronze { color: var(--bronze); }
.medal.inactive { opacity: 0.2; }

.course-lock-icon {
  font-size: 2rem;
}
```

### Enter / Exit

```javascript
function populateCourseGrid() {
  const grid = document.querySelector('.courses-grid');
  grid.innerHTML = '';

  const courses = [
    { id: 'pier', name: 'Seaside Pier', desc: 'Calm waters, gentle winds', unlock: 0 },
    { id: 'cliff', name: 'Windy Cliff', desc: 'Strong gusts, tricky thermals', unlock: 200 },
    { id: 'rooftop', name: 'Urban Rooftop', desc: 'Warm updrafts, longer flight', unlock: 350 },
    { id: 'mountain', name: 'Alpine Mountain', desc: 'Chaotic winds, big rewards', unlock: 500 }
  ];

  const unlocks = gameData.getCourseUnlocks();

  courses.forEach(course => {
    const isUnlocked = unlocks.includes(course.id);
    const best = gameData.getBestDistance(course.id);
    const medals = gameData.getMedals(course.id);

    const card = document.createElement('div');
    card.className = 'course-card' + (isUnlocked ? '' : ' course-locked');
    card.dataset.course = course.id;

    card.innerHTML = `
      <div class="course-card-thumb">
        <div class="${isUnlocked ? 'course-card-icon' : 'course-lock-icon'}">
          ${isUnlocked ? course.id.toUpperCase().slice(0, 3) : '&#128274;'}
        </div>
      </div>
      <div class="course-card-info">
        <h3 class="course-card-name">${course.name}</h3>
        <p class="course-card-desc">${isUnlocked ? course.desc : 'Reach ' + course.unlock + 'm to unlock'}</p>
        ${isUnlocked && best > 0 ? `
          <div class="course-card-best">
            <span class="stat-label">BEST</span>
            <span class="stat-value">${Math.round(best)}m</span>
          </div>
        ` : ''}
      </div>
    `;

    if (isUnlocked) {
      card.addEventListener('click', () => {
        window.selectedCourse = course.id;
        screenManager.show('screen-ramp', { course: course.id });
      });
    }

    grid.appendChild(card);
  });
}
```

### Interactive Elements

- **Course cards**: Tap unlocked course to proceed to Ramp Run
- **"BACK" button**: Returns to Workshop

---

## Ramp Run Screen

### HTML Template

```html
<div id="screen-ramp" class="screen">
  <!-- Countdown overlay (shown first, then hidden) -->
  <div class="ramp-countdown" style="display:none;">
    <span class="ramp-countdown-number">3</span>
  </div>

  <!-- Power meter area -->
  <div class="ramp-hud">
    <!-- Speed readout -->
    <div class="ramp-speed">
      <span class="stat-label">SPEED</span>
      <span class="stat-value ramp-speed-value">0</span>
    </div>

    <!-- Power meter bar -->
    <div class="ramp-meter-container">
      <div class="ramp-meter-label stat-label">POWER</div>
      <div class="ramp-meter-track">
        <div class="ramp-meter-fill"></div>
        <div class="ramp-meter-perfect-zone"></div>
        <div class="ramp-meter-marker"></div>
      </div>
    </div>

    <!-- Tap instruction -->
    <div class="ramp-tap-instruction">
      <span class="ramp-tap-text">TAP TAP TAP!</span>
    </div>
  </div>
</div>
```

### CSS

```css
/* Countdown */
.ramp-countdown {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 5;
}

.ramp-countdown-number {
  font-family: var(--font-display);
  font-size: 8rem;
  color: var(--yellow);
  text-shadow: 4px 4px 0 rgba(0,0,0,0.5);
  animation: countdownPop 0.8s ease-out;
}

@keyframes countdownPop {
  0% { transform: scale(2); opacity: 0; }
  30% { transform: scale(1); opacity: 1; }
  80% { transform: scale(1); opacity: 1; }
  100% { transform: scale(0.8); opacity: 0; }
}

/* Power meter HUD */
.ramp-hud {
  position: absolute;
  bottom: 0; left: 0; right: 0;
  padding: 20px 24px 32px;
  background: linear-gradient(0deg, rgba(0,0,0,0.7) 0%, transparent 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.ramp-speed {
  display: flex;
  gap: 8px;
  align-items: baseline;
}

.ramp-speed .stat-value {
  font-size: 1.8rem;
}

/* Power meter track */
.ramp-meter-container {
  width: 100%;
  max-width: 500px;
}

.ramp-meter-label {
  margin-bottom: 4px;
}

.ramp-meter-track {
  position: relative;
  width: 100%;
  height: 32px;
  background: rgba(255,255,255,0.15);
  border-radius: 16px;
  overflow: hidden;
  border: 2px solid rgba(255,255,255,0.3);
}

.ramp-meter-fill {
  position: absolute;
  top: 0; left: 0; bottom: 0;
  width: 0%;
  border-radius: 14px;
  background: linear-gradient(90deg,
    var(--red) 0%,
    var(--orange) 30%,
    var(--yellow) 50%,
    var(--green) 70%,
    var(--sky-blue) 100%
  );
  transition: width 0.05s linear;
}

.ramp-meter-perfect-zone {
  position: absolute;
  top: 0; bottom: 0;
  left: 65%; width: 15%;
  background: rgba(255, 215, 0, 0.3);
  border-left: 2px solid var(--yellow);
  border-right: 2px solid var(--yellow);
}

.ramp-meter-marker {
  position: absolute;
  top: -4px; bottom: -4px;
  width: 4px;
  left: 0%;
  background: white;
  border-radius: 2px;
  box-shadow: 0 0 8px rgba(255,255,255,0.8);
  transition: left 0.05s linear;
}

/* Tap instruction */
.ramp-tap-instruction {
  margin-top: 8px;
}

.ramp-tap-text {
  font-family: var(--font-display);
  font-size: 2rem;
  color: var(--yellow);
  text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
  animation: tapBounce 0.4s ease-in-out infinite alternate;
}

@keyframes tapBounce {
  0% { transform: translateY(0) scale(1); }
  100% { transform: translateY(-6px) scale(1.05); }
}
```

### Enter / Exit

```javascript
let rampTapCount = 0;
let rampPower = 0;
let rampInterval = null;
let rampCountdownTimer = null;

function initRampSequence() {
  rampTapCount = 0;
  rampPower = 0;
  updatePowerMeter(0);

  const countdown = document.querySelector('.ramp-countdown');
  const hud = document.querySelector('.ramp-hud');
  const tapText = document.querySelector('.ramp-tap-text');

  // Hide HUD during countdown
  hud.style.opacity = '0';
  countdown.style.display = 'flex';

  // 3-2-1-GO countdown
  let count = 3;
  const numEl = document.querySelector('.ramp-countdown-number');
  numEl.textContent = count;

  rampCountdownTimer = setInterval(() => {
    count--;
    if (count > 0) {
      numEl.textContent = count;
      // Re-trigger animation
      numEl.style.animation = 'none';
      numEl.offsetHeight; // force reflow
      numEl.style.animation = 'countdownPop 0.8s ease-out';
    } else if (count === 0) {
      numEl.textContent = 'GO!';
      numEl.style.color = 'var(--green)';
      numEl.style.animation = 'none';
      numEl.offsetHeight;
      numEl.style.animation = 'countdownPop 0.8s ease-out';
    } else {
      clearInterval(rampCountdownTimer);
      countdown.style.display = 'none';
      numEl.style.color = ''; // reset
      hud.style.opacity = '1';
      startRampInput();
    }
  }, 1000);
}

function startRampInput() {
  const screen = document.getElementById('screen-ramp');

  // Listen for taps
  const tapHandler = (e) => {
    e.preventDefault();
    rampTapCount++;
    rampPower = Math.min(rampPower + 4, 100);
    updatePowerMeter(rampPower);
    // Haptic feedback if available
    if (navigator.vibrate) navigator.vibrate(15);
  };

  screen.addEventListener('pointerdown', tapHandler);
  screen._tapHandler = tapHandler;

  // Power decays over time
  rampInterval = setInterval(() => {
    rampPower = Math.max(rampPower - 1.5, 0);
    updatePowerMeter(rampPower);
  }, 100);

  // Auto-launch after 5 seconds of tapping
  setTimeout(() => {
    launchFromRamp(rampPower);
  }, 5000);
}

function updatePowerMeter(power) {
  const fill = document.querySelector('.ramp-meter-fill');
  const marker = document.querySelector('.ramp-meter-marker');
  const speedVal = document.querySelector('.ramp-speed-value');

  if (fill) fill.style.width = power + '%';
  if (marker) marker.style.left = power + '%';
  if (speedVal) speedVal.textContent = Math.round(power * 0.8);
}

function launchFromRamp(power) {
  cleanupRampUI();

  // Determine if launch was in the perfect zone (65-80%)
  const isPerfect = power >= 65 && power <= 80;

  // Notify flight engine
  if (window.FlightEngine) {
    window.FlightEngine.launch({
      power: power / 100,
      perfect: isPerfect,
      tapCount: rampTapCount
    });
  }

  // Show floating text for perfect launch
  if (isPerfect) {
    showFloatingText('+500 PERFECT LAUNCH!', 'var(--yellow)');
  }

  // Transition to flight HUD
  screenManager.show('screen-hud');
}

function cleanupRampUI() {
  if (rampInterval) { clearInterval(rampInterval); rampInterval = null; }
  if (rampCountdownTimer) { clearInterval(rampCountdownTimer); rampCountdownTimer = null; }
  const screen = document.getElementById('screen-ramp');
  if (screen._tapHandler) {
    screen.removeEventListener('pointerdown', screen._tapHandler);
    delete screen._tapHandler;
  }
}
```

### 3D Scene Behind

- Side view of the pier ramp
- Craft at the top of the ramp with the pilot
- Crowd cheering on either side
- Ocean visible at the end of the ramp

### Interactive Elements

- **Tap anywhere on screen**: Increases power meter
- Power meter auto-decays, requiring rapid tapping to build up
- After 5 seconds, auto-launches with current power level
- Perfect zone (65-80%) gives bonus score

---

## Flight HUD Screen

### HTML Template

```html
<div id="screen-hud" class="screen">
  <!-- Top HUD bar -->
  <div class="hud-top">
    <!-- Altitude (left) -->
    <div class="hud-altitude">
      <div class="hud-alt-bar-container">
        <div class="hud-alt-bar-track">
          <div class="hud-alt-bar-fill"></div>
        </div>
        <span class="hud-alt-value stat-value">0m</span>
      </div>
      <span class="hud-alt-label stat-label">ALT</span>
    </div>

    <!-- Distance (center) -->
    <div class="hud-distance">
      <span class="hud-dist-value score-counter">0</span>
      <span class="hud-dist-unit stat-label">meters</span>
    </div>

    <!-- Pause button (right) -->
    <button class="hud-pause-btn touch-target" style="pointer-events:auto;">
      <span class="hud-pause-icon">| |</span>
    </button>
  </div>

  <!-- Wind indicator (top area) -->
  <div class="hud-wind">
    <span class="hud-wind-arrow">&#8594;</span>
    <span class="hud-wind-label stat-label">WIND</span>
  </div>

  <!-- Bottom HUD bar -->
  <div class="hud-bottom">
    <!-- Speed (left) -->
    <div class="hud-speed">
      <span class="hud-speed-label stat-label">SPD</span>
      <div class="hud-speed-bar-track">
        <div class="hud-speed-bar-fill"></div>
      </div>
      <span class="hud-speed-value stat-value">0</span>
    </div>

    <!-- Pitch indicator (right) -->
    <div class="hud-pitch">
      <span class="hud-pitch-arrow">&#8595;</span>
    </div>
  </div>

  <!-- Thermal proximity glow (hidden by default) -->
  <div class="hud-thermal-glow" style="display:none;"></div>
</div>
```

### CSS

```css
#screen-hud {
  pointer-events: none;
}

.hud-top {
  position: absolute;
  top: 0; left: 0; right: 0;
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  padding: 12px 16px;
  padding-top: calc(12px + env(safe-area-inset-top, 0px));
  background: linear-gradient(180deg, rgba(0,0,0,0.4) 0%, transparent 100%);
}

/* Altitude bar */
.hud-altitude {
  display: flex;
  align-items: center;
  gap: 8px;
}

.hud-alt-bar-container {
  display: flex;
  align-items: center;
  gap: 6px;
}

.hud-alt-bar-track {
  width: 80px;
  height: 12px;
  background: rgba(255,255,255,0.15);
  border-radius: 6px;
  overflow: hidden;
}

.hud-alt-bar-fill {
  height: 100%;
  width: 0%;
  border-radius: 6px;
  background: var(--green);
  transition: width 0.15s linear, background-color 0.3s ease;
}

/* Color coding for altitude */
.hud-alt-bar-fill.alt-low { background: var(--red); }
.hud-alt-bar-fill.alt-mid { background: var(--green); }
.hud-alt-bar-fill.alt-high { background: var(--sky-blue); }

.hud-alt-value {
  font-size: 1.2rem;
  min-width: 40px;
}

/* Distance counter (center) */
.hud-distance {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.hud-dist-value {
  font-size: 2.5rem;
  line-height: 1;
}

.hud-dist-unit {
  font-size: 0.7rem;
  margin-top: 2px;
}

/* Pause button */
.hud-pause-btn {
  width: 44px;
  height: 44px;
  background: rgba(0,0,0,0.4);
  border: 2px solid rgba(255,255,255,0.3);
  border-radius: 10px;
  color: white;
  font-size: 1.2rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.hud-pause-btn:active {
  background: rgba(255,255,255,0.2);
}

/* Wind indicator */
.hud-wind {
  position: absolute;
  top: 60px;
  right: 16px;
  display: flex;
  align-items: center;
  gap: 4px;
  color: var(--text-secondary);
  font-size: 0.8rem;
}

.hud-wind-arrow {
  font-size: 1.2rem;
  transition: transform 0.5s ease;
}

/* Bottom HUD bar */
.hud-bottom {
  position: absolute;
  bottom: 0; left: 0; right: 0;
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  padding: 12px 16px;
  padding-bottom: calc(12px + env(safe-area-inset-bottom, 0px));
  background: linear-gradient(0deg, rgba(0,0,0,0.3) 0%, transparent 100%);
}

/* Speed bar */
.hud-speed {
  display: flex;
  align-items: center;
  gap: 8px;
}

.hud-speed-bar-track {
  width: 100px;
  height: 10px;
  background: rgba(255,255,255,0.15);
  border-radius: 5px;
  overflow: hidden;
}

.hud-speed-bar-fill {
  height: 100%;
  width: 0%;
  border-radius: 5px;
  background: var(--green);
  transition: width 0.15s linear;
}

.hud-speed-value {
  font-size: 1rem;
  min-width: 30px;
}

/* Pitch indicator */
.hud-pitch {
  font-size: 1.5rem;
  color: rgba(255,255,255,0.5);
  transition: transform 0.2s ease;
}

/* Thermal proximity glow */
.hud-thermal-glow {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  border: 4px solid transparent;
  border-radius: 0;
  pointer-events: none;
  animation: thermalGlow 1s ease-in-out infinite;
}

@keyframes thermalGlow {
  0%, 100% { border-color: rgba(255, 165, 0, 0); box-shadow: inset 0 0 20px rgba(255,165,0,0); }
  50% { border-color: rgba(255, 165, 0, 0.3); box-shadow: inset 0 0 40px rgba(255,165,0,0.15); }
}
```

### HUD Update Loop

```javascript
let hudAnimFrame = null;

function startHUDUpdates() {
  function updateHUD() {
    if (!window.FlightEngine) { hudAnimFrame = requestAnimationFrame(updateHUD); return; }

    const state = window.FlightEngine.getState();
    // state = { altitude, distance, speed, maxAltitude, pitchAngle, windDirection, nearThermal }

    // Altitude
    const altFill = document.querySelector('.hud-alt-bar-fill');
    const altVal = document.querySelector('.hud-alt-value');
    const altPct = Math.min((state.altitude / 100) * 100, 100);
    altFill.style.width = altPct + '%';
    altFill.className = 'hud-alt-bar-fill ' +
      (state.altitude < 10 ? 'alt-low' : state.altitude < 50 ? 'alt-mid' : 'alt-high');
    altVal.textContent = Math.round(state.altitude) + 'm';

    // Distance
    document.querySelector('.hud-dist-value').textContent = Math.round(state.distance);

    // Speed
    const spdFill = document.querySelector('.hud-speed-bar-fill');
    const spdVal = document.querySelector('.hud-speed-value');
    const spdPct = Math.min((state.speed / 80) * 100, 100);
    spdFill.style.width = spdPct + '%';
    spdVal.textContent = Math.round(state.speed);

    // Pitch
    const pitchArrow = document.querySelector('.hud-pitch-arrow');
    if (state.pitchAngle > 5) {
      pitchArrow.innerHTML = '&#8593;'; // up arrow
      pitchArrow.style.transform = 'rotate(0deg)';
    } else if (state.pitchAngle < -5) {
      pitchArrow.innerHTML = '&#8595;'; // down arrow
      pitchArrow.style.transform = 'rotate(0deg)';
    } else {
      pitchArrow.innerHTML = '&#8594;'; // level
    }

    // Wind direction
    const windArrow = document.querySelector('.hud-wind-arrow');
    if (windArrow && state.windDirection !== undefined) {
      windArrow.style.transform = `rotate(${state.windDirection}deg)`;
    }

    // Thermal proximity
    const thermalGlow = document.querySelector('.hud-thermal-glow');
    if (thermalGlow) {
      thermalGlow.style.display = state.nearThermal ? 'block' : 'none';
    }

    hudAnimFrame = requestAnimationFrame(updateHUD);
  }

  hudAnimFrame = requestAnimationFrame(updateHUD);

  // Pause button
  document.querySelector('.hud-pause-btn').addEventListener('click', () => {
    screenManager.show('screen-pause');
  });
}

function stopHUDUpdates() {
  if (hudAnimFrame) {
    cancelAnimationFrame(hudAnimFrame);
    hudAnimFrame = null;
  }
}
```

### 3D Scene Behind

- Active flight scene (flight-engine skill controls this)
- Camera follows the craft
- Ocean below, sky above, course environment visible

### Interactive Elements

- **Pause button** (top right): Only interactive element, opens Pause screen
- All other HUD elements are read-only overlays with `pointer-events: none`
- Flight controls are handled directly by the flight engine (touch on canvas)

---

## Splash Screen

### HTML Template

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
  color: var(--water-foam);
  text-shadow:
    3px 3px 0 var(--ocean),
    6px 6px 0 var(--ocean-deep),
    0 0 30px rgba(0, 136, 170, 0.5);
  animation: splashBounce 0.6s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
  z-index: 2;
}

@keyframes splashBounce {
  0% { transform: scale(0) rotate(-10deg); opacity: 0; }
  50% { transform: scale(1.2) rotate(3deg); opacity: 1; }
  100% { transform: scale(1) rotate(0deg); opacity: 1; }
}

.splash-water-burst {
  position: absolute;
  width: 300px;
  height: 300px;
  border-radius: 50%;
  background: radial-gradient(
    circle,
    rgba(184, 230, 240, 0.4) 0%,
    rgba(0, 136, 170, 0.2) 40%,
    transparent 70%
  );
  animation: waterBurst 1s ease-out forwards;
  z-index: 1;
}

@keyframes waterBurst {
  0% { transform: scale(0); opacity: 1; }
  100% { transform: scale(3); opacity: 0; }
}

/* Screen shake on splash */
#screen-splash.shake {
  animation: screenShake 0.4s ease-out;
}

@keyframes screenShake {
  0% { transform: translate(0, 0); }
  15% { transform: translate(-8px, 4px); }
  30% { transform: translate(6px, -6px); }
  45% { transform: translate(-4px, 2px); }
  60% { transform: translate(3px, -3px); }
  75% { transform: translate(-2px, 1px); }
  100% { transform: translate(0, 0); }
}
```

### Enter / Exit

```javascript
function playSplashAnimation() {
  const screen = document.getElementById('screen-splash');
  screen.classList.add('shake');

  // Haptic feedback
  if (navigator.vibrate) navigator.vibrate([50, 30, 100]);

  // Auto-advance to results after 1.5s
  setTimeout(() => {
    screen.classList.remove('shake');

    // Gather flight data for results
    const flightData = window.FlightEngine ? window.FlightEngine.getResults() : {
      distance: 0, maxAltitude: 0, styleMultiplier: 1, score: 0
    };

    screenManager.show('screen-results', flightData);
  }, 1500);
}
```

### 3D Scene Behind

- Craft splashing into the water
- Large splash particle effect from the 3D engine
- Water ripples expanding outward
- Crowd visible on the pier in the background

### Interactive Elements

- None -- this is a brief non-interactive transition screen (1.5s)

---

## Results Screen

### HTML Template

```html
<div id="screen-results" class="screen">
  <div class="results-panel">
    <!-- Header -->
    <h1 class="results-title title-text">SPLASH!</h1>

    <!-- Crowd reaction (changes based on distance) -->
    <div class="results-crowd-reaction">
      <span class="crowd-reaction-text"></span>
    </div>

    <!-- Stats -->
    <div class="results-stats">
      <div class="results-stat-row">
        <span class="stat-label">DISTANCE</span>
        <span class="stat-value results-distance" data-target="0">0m</span>
      </div>
      <div class="results-stat-row">
        <span class="stat-label">MAX ALTITUDE</span>
        <span class="stat-value results-altitude" data-target="0">0m</span>
      </div>
      <div class="results-stat-row">
        <span class="stat-label">STYLE BONUS</span>
        <span class="stat-value results-style" data-target="0">x1.00</span>
      </div>
      <div class="results-divider"></div>
      <div class="results-stat-row results-total-row">
        <span class="stat-label">TOTAL SCORE</span>
        <span class="score-counter results-total" data-target="0">0</span>
      </div>
    </div>

    <!-- Medal -->
    <div class="results-medal" style="display:none;">
      <span class="results-medal-icon"></span>
      <span class="results-medal-label"></span>
    </div>

    <!-- New record banner -->
    <div class="results-new-record" style="display:none;">
      NEW RECORD!
    </div>

    <!-- Best distance -->
    <div class="results-best">
      <span class="stat-label">PERSONAL BEST</span>
      <span class="stat-value results-best-value">0m</span>
    </div>

    <!-- Action buttons (appear after animation) -->
    <div class="results-buttons" style="opacity:0;">
      <button class="game-btn game-btn-primary game-btn-pulse results-fly-again-btn">
        FLY AGAIN
      </button>
      <button class="game-btn game-btn-secondary results-workshop-btn">
        WORKSHOP
      </button>
    </div>
  </div>
</div>
```

### CSS

```css
.results-panel {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 32px 24px;
  max-width: 420px;
  width: 90%;
  background: var(--panel-bg);
  border-radius: 24px;
  border: 2px solid var(--panel-border);
  box-shadow: var(--shadow-heavy);
  max-height: 90vh;
  overflow-y: auto;
}

.results-title {
  font-size: 2.8rem;
  color: var(--water-foam);
  text-shadow: 3px 3px 0 var(--ocean), 0 0 20px rgba(0,136,170,0.3);
}

/* Crowd reaction */
.results-crowd-reaction {
  min-height: 32px;
}

.crowd-reaction-text {
  font-family: var(--font-display);
  font-size: 1.5rem;
  color: var(--yellow);
  text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
  animation: reactionPop 0.5s cubic-bezier(0.34, 1.56, 0.64, 1);
}

@keyframes reactionPop {
  0% { transform: scale(0); }
  100% { transform: scale(1); }
}

/* Stats */
.results-stats {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.results-stat-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  padding: 4px 0;
}

.results-stat-row .stat-value {
  font-size: 1.5rem;
}

.results-divider {
  width: 100%;
  height: 2px;
  background: rgba(255,255,255,0.15);
  margin: 8px 0;
}

.results-total-row .score-counter {
  font-size: 2.2rem;
}

/* Medal */
.results-medal {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  margin: 8px 0;
}

.results-medal-icon {
  font-size: 3rem;
  animation: medalReveal 0.6s cubic-bezier(0.34, 1.56, 0.64, 1);
}

@keyframes medalReveal {
  0% { transform: scale(0) rotate(-180deg); opacity: 0; }
  100% { transform: scale(1) rotate(0deg); opacity: 1; }
}

.results-medal-label {
  font-family: var(--font-display);
  font-size: 1.2rem;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.results-medal.gold .results-medal-icon { color: var(--gold); }
.results-medal.gold .results-medal-label { color: var(--gold); }
.results-medal.silver .results-medal-icon { color: var(--silver); }
.results-medal.silver .results-medal-label { color: var(--silver); }
.results-medal.bronze .results-medal-icon { color: var(--bronze); }
.results-medal.bronze .results-medal-label { color: var(--bronze); }

/* New record banner */
.results-new-record {
  font-family: var(--font-display);
  font-size: 1.5rem;
  color: var(--yellow);
  text-shadow: 2px 2px 0 rgba(0,0,0,0.5);
  padding: 8px 24px;
  background: linear-gradient(90deg, transparent, rgba(255,215,0,0.2), transparent);
  animation: recordFlash 1s ease-in-out infinite alternate;
}

@keyframes recordFlash {
  0% { text-shadow: 2px 2px 0 rgba(0,0,0,0.5); }
  100% { text-shadow: 2px 2px 0 rgba(0,0,0,0.5), 0 0 20px rgba(255,215,0,0.5); }
}

/* Best distance */
.results-best {
  display: flex;
  gap: 8px;
  align-items: baseline;
}

/* Buttons */
.results-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 100%;
  margin-top: 8px;
  transition: opacity 0.3s ease;
}

.results-buttons .game-btn {
  width: 100%;
}
```

### Enter / Exit

```javascript
function populateResults(data) {
  // data = { distance, maxAltitude, styleMultiplier, score }
  const distance = data.distance || 0;
  const maxAlt = data.maxAltitude || 0;
  const style = data.styleMultiplier || 1;
  const score = data.score || 0;

  const course = window.selectedCourse || 'pier';

  // Determine medal
  let medal = null;
  if (distance >= 500) medal = 'gold';
  else if (distance >= 300) medal = 'silver';
  else if (distance >= 150) medal = 'bronze';

  // Check for new record
  const prevBest = gameData.getBestDistance(course);
  const isNewRecord = distance > prevBest;

  // Save data
  if (isNewRecord) gameData.saveBestDistance(course, distance);
  if (score > gameData.getHighScore(course)) gameData.saveHighScore(course, score);
  if (medal) gameData.saveMedal(course, medal);
  gameData.incrementFlights();

  // Check course unlocks
  checkCourseUnlocks();

  // Set crowd reaction
  const reactionEl = document.querySelector('.crowd-reaction-text');
  if (distance < 50) reactionEl.textContent = 'OHHH...';
  else if (distance < 150) reactionEl.textContent = 'NOT BAD!';
  else if (distance < 300) reactionEl.textContent = 'WOOO!';
  else if (distance < 500) reactionEl.textContent = 'INCREDIBLE!';
  else reactionEl.textContent = 'LEGENDARY!';

  // Set best distance
  const bestVal = document.querySelector('.results-best-value');
  bestVal.textContent = Math.round(Math.max(distance, prevBest)) + 'm';

  // Animate stats sequentially
  const distEl = document.querySelector('.results-distance');
  const altEl = document.querySelector('.results-altitude');
  const styleEl = document.querySelector('.results-style');
  const totalEl = document.querySelector('.results-total');

  // Start animations with delays
  setTimeout(() => animateCounter(distEl, 0, distance, 800, (v) => Math.round(v) + 'm'), 200);
  setTimeout(() => animateCounter(altEl, 0, maxAlt, 600, (v) => Math.round(v) + 'm'), 700);
  setTimeout(() => animateCounter(styleEl, 1, style, 400, (v) => 'x' + v.toFixed(2)), 1100);
  setTimeout(() => animateCounter(totalEl, 0, score, 1000, (v) => Math.round(v).toLocaleString()), 1500);

  // Show medal after score animation
  if (medal) {
    setTimeout(() => {
      const medalEl = document.querySelector('.results-medal');
      medalEl.style.display = 'flex';
      medalEl.className = 'results-medal ' + medal;
      document.querySelector('.results-medal-icon').textContent = '★';
      document.querySelector('.results-medal-label').textContent =
        medal === 'gold' ? 'GOLD' : medal === 'silver' ? 'SILVER' : 'BRONZE';
    }, 2600);
  }

  // Show new record banner
  if (isNewRecord && prevBest > 0) {
    setTimeout(() => {
      document.querySelector('.results-new-record').style.display = 'block';
    }, 2800);
  }

  // Show buttons after all animations
  setTimeout(() => {
    document.querySelector('.results-buttons').style.opacity = '1';
  }, 3000);

  // Wire buttons
  document.querySelector('.results-fly-again-btn').onclick = () => {
    screenManager.show('screen-ramp', { course });
  };
  document.querySelector('.results-workshop-btn').onclick = () => {
    screenManager.show('screen-workshop');
  };
}

function animateCounter(el, from, to, duration, formatter) {
  const start = performance.now();
  function tick(now) {
    const elapsed = now - start;
    const progress = Math.min(elapsed / duration, 1);
    // Ease-out cubic
    const eased = 1 - Math.pow(1 - progress, 3);
    const value = from + (to - from) * eased;
    el.textContent = formatter(value);
    if (progress < 1) requestAnimationFrame(tick);
  }
  requestAnimationFrame(tick);
}

function checkCourseUnlocks() {
  const thresholds = { cliff: 200, rooftop: 350, mountain: 500 };
  const bestPier = gameData.getBestDistance('pier');
  Object.entries(thresholds).forEach(([course, threshold]) => {
    if (bestPier >= threshold) {
      gameData.unlockCourse(course);
    }
  });
}

function cleanupResults() {
  // Reset display states for next time
  document.querySelector('.results-medal').style.display = 'none';
  document.querySelector('.results-new-record').style.display = 'none';
  document.querySelector('.results-buttons').style.opacity = '0';
  document.querySelector('.results-distance').textContent = '0m';
  document.querySelector('.results-altitude').textContent = '0m';
  document.querySelector('.results-style').textContent = 'x1.00';
  document.querySelector('.results-total').textContent = '0';
}
```

### 3D Scene Behind

- Craft bobbing in the water after splashdown
- Camera positioned to see the pier in the distance
- Sunset/golden hour lighting for celebration mood
- Gentle waves

### Interactive Elements

- **"FLY AGAIN" button**: Returns to Ramp Run with the same craft and course
- **"WORKSHOP" button**: Returns to Workshop to modify the craft

---

## Pause Screen

### HTML Template

```html
<div id="screen-pause" class="screen">
  <!-- Dim overlay -->
  <div class="pause-overlay"></div>

  <!-- Pause panel -->
  <div class="pause-panel">
    <h2 class="title-text" style="font-size:2.5rem;">PAUSED</h2>

    <!-- Current flight stats snapshot -->
    <div class="pause-stats">
      <div class="pause-stat">
        <span class="stat-label">DISTANCE</span>
        <span class="stat-value pause-distance">0m</span>
      </div>
      <div class="pause-stat">
        <span class="stat-label">ALTITUDE</span>
        <span class="stat-value pause-altitude">0m</span>
      </div>
    </div>

    <!-- Buttons -->
    <div class="pause-buttons">
      <button class="game-btn game-btn-primary pause-resume-btn">
        RESUME
      </button>
      <button class="game-btn game-btn-secondary pause-workshop-btn">
        WORKSHOP
      </button>
      <button class="game-btn game-btn-secondary pause-settings-btn">
        SETTINGS
      </button>
    </div>
  </div>
</div>
```

### CSS

```css
.pause-overlay {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
}

.pause-panel {
  position: relative;
  z-index: 2;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  padding: 32px;
  background: var(--panel-bg);
  border-radius: 24px;
  border: 2px solid var(--panel-border);
  box-shadow: var(--shadow-heavy);
  min-width: 280px;
}

.pause-stats {
  display: flex;
  gap: 24px;
}

.pause-stat {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.pause-stat .stat-value {
  font-size: 1.5rem;
}

.pause-buttons {
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 100%;
}

.pause-buttons .game-btn {
  width: 100%;
}
```

### Enter / Exit

```javascript
function pauseGame() {
  // Freeze the flight engine
  if (window.FlightEngine) {
    window.FlightEngine.pause();
    const state = window.FlightEngine.getState();
    document.querySelector('.pause-distance').textContent = Math.round(state.distance) + 'm';
    document.querySelector('.pause-altitude').textContent = Math.round(state.altitude) + 'm';
  }

  // Wire buttons
  document.querySelector('.pause-resume-btn').onclick = () => {
    screenManager.show('screen-hud');
  };
  document.querySelector('.pause-workshop-btn').onclick = () => {
    if (window.FlightEngine) window.FlightEngine.abort();
    screenManager.show('screen-workshop');
  };
  document.querySelector('.pause-settings-btn').onclick = () => {
    // Open settings overlay (in-pause settings)
    openSettingsOverlay();
  };
}

function resumeGame() {
  if (window.FlightEngine) {
    window.FlightEngine.resume();
  }
}

function openSettingsOverlay() {
  // Minimal settings: sound toggle, controls sensitivity
  // Implementation depends on game engine capabilities
  console.log('Settings overlay -- to be implemented');
}
```

### 3D Scene Behind

- Frozen (paused) flight scene
- Dimmed via the `.pause-overlay` CSS with `backdrop-filter: blur`

### Interactive Elements

- **"RESUME" button**: Returns to Flight HUD, unpauses engine
- **"WORKSHOP" button**: Abandons flight, returns to Workshop
- **"SETTINGS" button**: Opens settings overlay (sound, controls)

---

## ScreenManager Wiring

Complete wiring code that connects all screens, buttons, and transitions.

```javascript
// ====================================
// Full ScreenManager initialization
// ====================================

document.addEventListener('DOMContentLoaded', () => {
  // Initialize persistence
  const gameData = new GameDataManager();
  gameData.init();
  window.gameData = gameData;

  // Initialize screen manager
  const screenManager = new ScreenManager();
  window.screenManager = screenManager;

  // Register all screens
  screenManager.register('screen-title', {
    interactive: true,
    transition: 'fade',
    onEnter: () => {
      startTitleAnimation();
    },
    onExit: () => {
      stopTitleAnimation();
    }
  });

  screenManager.register('screen-workshop', {
    interactive: true,
    transition: 'slide-up',
    onEnter: () => {
      initWorkshop();
    },
    onExit: () => {
      saveCurrentCraft();
    }
  });

  screenManager.register('screen-courses', {
    interactive: true,
    transition: 'pop',
    onEnter: () => {
      populateCourseGrid();
    },
    onExit: () => {}
  });

  screenManager.register('screen-ramp', {
    interactive: true,
    transition: 'fade',
    onEnter: (data) => {
      window.selectedCourse = data.course || window.selectedCourse || 'pier';
      initRampSequence();
    },
    onExit: () => {
      cleanupRampUI();
    }
  });

  screenManager.register('screen-hud', {
    interactive: false,
    transition: 'fade',
    onEnter: () => {
      startHUDUpdates();
    },
    onExit: () => {
      stopHUDUpdates();
    }
  });

  screenManager.register('screen-splash', {
    interactive: false,
    transition: 'none',
    onEnter: () => {
      playSplashAnimation();
    },
    onExit: () => {}
  });

  screenManager.register('screen-results', {
    interactive: true,
    transition: 'slide-up',
    onEnter: (data) => {
      populateResults(data);
    },
    onExit: () => {
      cleanupResults();
    }
  });

  screenManager.register('screen-pause', {
    interactive: true,
    transition: 'pop',
    onEnter: () => {
      pauseGame();
    },
    onExit: () => {
      resumeGame();
    }
  });

  // ---- Button Wiring ----

  // Title -> Workshop
  document.querySelector('.title-cta').addEventListener('click', () => {
    screenManager.show('screen-workshop');
  });

  // Workshop -> Back to Title
  document.querySelector('.workshop-back-btn').addEventListener('click', () => {
    screenManager.show('screen-title');
  });

  // Workshop -> Launch (Course Select or Ramp Run)
  document.querySelector('.workshop-launch-btn').addEventListener('click', () => {
    const unlocks = gameData.getCourseUnlocks();
    if (unlocks.length > 1) {
      screenManager.show('screen-courses');
    } else {
      window.selectedCourse = 'pier';
      screenManager.show('screen-ramp', { course: 'pier' });
    }
  });

  // Course Select -> Back
  document.querySelector('.courses-back-btn').addEventListener('click', () => {
    screenManager.show('screen-workshop');
  });

  // ---- Flight Engine Events ----

  // Listen for water collision from flight engine
  window.addEventListener('flugtag:splash', () => {
    screenManager.hideHUD();
    screenManager.show('screen-splash');
  });

  // ---- Start Game ----
  screenManager.show('screen-title');
});
```
