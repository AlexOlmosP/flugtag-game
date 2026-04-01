---
name: flugtag-game-ui
description: >
  Game UI system for the Red Bull Flugtag HTML game. Invoke this skill when working on:
  game menus, title screen, HUD, heads-up display, results screen, pause menu, screen
  transitions, screen navigation, persistent storage, high scores, best distances, unlocks,
  settings, workshop flow, flight flow, ramp power meter UI, altitude display, distance
  counter, crowd reactions, medal display, score counters, floating text, splash screen,
  course select, responsive layout, touch targets, overall UX, game flow, screen manager,
  CSS overlays, button styles, font system, color palette, data persistence.
---

# Flugtag Game UI Skill

## Context

Red Bull Flugtag is a single-file HTML flight game built with **Three.js r128**.
Players build absurd flying machines in a workshop, launch them off a pier ramp over
water, and compete for distance and style points. The UI is entirely HTML/CSS overlays
floating on top of the Three.js canvas.

- **Art style**: Subway Surfers / Fall Guys chunky colorful toon-shaded cartoon
- **Tech stack**: Three.js r128 loaded from CDN, single `index.html`, HTML/CSS overlays
- **Persistence**: `localStorage` keyed under `flugtag:*` namespace
- **Layout**: `position: absolute` screens over a full-viewport `<canvas>`

## Design Philosophy

**Bold, chunky, LOUD.** The UI should feel like a festive Flugtag event -- colorful
banners, crowd energy, celebration. Big blocky buttons, thick text shadows, saturated
colors. Everything is an HTML/CSS overlay with `position: absolute` and `pointer-events`
control so the 3D scene always shows through.

The UI never feels "minimal" -- it's a party. But it's also crystal clear and never
blocks gameplay information. During flight, the HUD is sparse and readable. Between
phases, the screens are bold and fun.

## References

| Document | Purpose |
|----------|---------|
| [screen-flows.md](references/screen-flows.md) | Complete screen-by-screen specs, HTML templates, CSS, transitions, ScreenManager wiring |
| [hud-and-effects.md](references/hud-and-effects.md) | Flight HUD, ramp HUD, floating text, splash effects, crowd reactions, score animations |
| [persistence.md](references/persistence.md) | GameDataManager class, localStorage keys, defaults, migration, error handling |

---

## Screen Map

```
Title --> Workshop (builder) --> Ramp Run --> Flight --> Splash --> Results
               ^                                                     |
               |                                                     |
               +-----------------------------------------------------+
              "Workshop" button                "Fly Again" loops to Ramp Run
```

**Alternate paths:**
- Title --> Course Select --> Workshop (if multiple courses unlocked)
- Flight --> Pause --> Resume (back to Flight)
- Flight --> Pause --> Workshop (abandon flight)
- Results --> Workshop (build a new craft)
- Results --> Ramp Run (fly again with same craft, same course)

---

## Screen List

| Screen | ID | Purpose | 3D Behind? | Pointer Events |
|--------|----|---------|------------|----------------|
| Title | `screen-title` | Brand splash, "TAP TO FLY" | Yes (craft on pier, sunset) | Overlay captures taps |
| Workshop | `screen-workshop` | Craft customizer (builder skill) | Yes (workshop scene) | Full interaction |
| Ramp Run | `screen-ramp` | Power meter, "TAP!" | Yes (side view of ramp) | Tap-only |
| Flight HUD | `screen-hud` | Active gameplay overlay | Yes (flight scene) | Pass-through (except pause btn) |
| Splash | `screen-splash` | Brief splash moment (1.5s) | Yes (splash in water) | No interaction |
| Results | `screen-results` | Score summary, medals, retry | Yes (craft bobbing in water) | Overlay captures taps |
| Pause | `screen-pause` | Pause menu during flight | Yes (frozen, dimmed) | Overlay captures taps |
| Course Select | `screen-courses` | Choose course before flight | Optional overlay | Full interaction |

---

## UI Architecture

### HTML Structure

```html
<div id="game-container" style="position:relative; width:100vw; height:100vh; overflow:hidden;">
  <!-- Three.js renders here -->
  <canvas id="game-canvas"></canvas>

  <!-- UI screens (only one visible at a time, except HUD during flight) -->
  <div id="screen-title" class="screen"></div>
  <div id="screen-workshop" class="screen"></div>
  <div id="screen-courses" class="screen"></div>
  <div id="screen-ramp" class="screen"></div>
  <div id="screen-hud" class="screen"></div>
  <div id="screen-splash" class="screen"></div>
  <div id="screen-results" class="screen"></div>
  <div id="screen-pause" class="screen"></div>

  <!-- Always-on effects layer for floating text, particles -->
  <div id="effects-layer" class="screen" style="pointer-events:none;"></div>
</div>
```

### Base CSS

```css
* { margin: 0; padding: 0; box-sizing: border-box; }

html, body {
  width: 100%; height: 100%;
  overflow: hidden;
  touch-action: none;
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  user-select: none;
}

#game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

#game-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 0;
}

.screen {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 10;
  display: none;
  pointer-events: none;
}

.screen.active {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.screen.interactive {
  pointer-events: auto;
}

#effects-layer {
  z-index: 20;
  display: block !important;
  pointer-events: none;
}
```

---

## ScreenManager Class

```javascript
class ScreenManager {
  constructor() {
    this.screens = {};
    this.currentScreen = null;
    this.previousScreen = null;
    this.transitionActive = false;
    this.listeners = {};
  }

  /**
   * Register a screen element with enter/exit callbacks.
   * @param {string} id - Screen element ID (e.g., 'screen-title')
   * @param {object} opts
   * @param {Function} opts.onEnter - Called when screen becomes active
   * @param {Function} opts.onExit  - Called when screen is hidden
   * @param {boolean} opts.interactive - Whether screen captures pointer events
   * @param {string} opts.transition - 'fade' | 'slide-up' | 'pop' | 'none'
   */
  register(id, opts = {}) {
    const el = document.getElementById(id);
    if (!el) { console.warn(`ScreenManager: #${id} not found`); return; }
    this.screens[id] = {
      el,
      onEnter: opts.onEnter || (() => {}),
      onExit: opts.onExit || (() => {}),
      interactive: opts.interactive !== false,
      transition: opts.transition || 'fade'
    };
  }

  /**
   * Navigate to a screen with animated transition.
   * @param {string} id - Target screen ID
   * @param {object} data - Optional data passed to onEnter
   * @param {string} overrideTransition - Override the default transition type
   */
  async show(id, data = {}, overrideTransition) {
    if (this.transitionActive) return;
    const target = this.screens[id];
    if (!target) { console.warn(`ScreenManager: unknown screen "${id}"`); return; }

    this.transitionActive = true;
    const transition = overrideTransition || target.transition;

    // Exit current screen
    if (this.currentScreen && this.screens[this.currentScreen]) {
      const current = this.screens[this.currentScreen];
      current.onExit();
      await this._animateOut(current.el, transition);
      current.el.classList.remove('active', 'interactive');
    }

    this.previousScreen = this.currentScreen;
    this.currentScreen = id;

    // Enter new screen
    target.el.classList.add('active');
    if (target.interactive) target.el.classList.add('interactive');
    await this._animateIn(target.el, transition);
    target.onEnter(data);

    this.transitionActive = false;
    this._emit('screenChanged', { from: this.previousScreen, to: id, data });
  }

  /**
   * Show the HUD overlay (does not hide current screen -- used during flight).
   */
  showHUD() {
    const hud = this.screens['screen-hud'];
    if (hud) {
      hud.el.classList.add('active');
      hud.onEnter();
    }
  }

  hideHUD() {
    const hud = this.screens['screen-hud'];
    if (hud) {
      hud.el.classList.remove('active', 'interactive');
      hud.onExit();
    }
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
  }

  _emit(event, data) {
    (this.listeners[event] || []).forEach(cb => cb(data));
  }

  async _animateIn(el, transition) {
    return new Promise(resolve => {
      switch (transition) {
        case 'fade':
          el.style.opacity = '0';
          el.style.transition = 'opacity 0.3s ease-out';
          requestAnimationFrame(() => {
            el.style.opacity = '1';
            setTimeout(resolve, 300);
          });
          break;

        case 'slide-up':
          el.style.transform = 'translateY(100%)';
          el.style.opacity = '1';
          el.style.transition = 'transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1)';
          requestAnimationFrame(() => {
            el.style.transform = 'translateY(0)';
            setTimeout(resolve, 400);
          });
          break;

        case 'pop':
          el.style.transform = 'scale(0.5)';
          el.style.opacity = '0';
          el.style.transition = 'transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.2s ease-out';
          requestAnimationFrame(() => {
            el.style.transform = 'scale(1)';
            el.style.opacity = '1';
            setTimeout(resolve, 350);
          });
          break;

        case 'none':
        default:
          el.style.opacity = '1';
          el.style.transform = '';
          resolve();
          break;
      }
    });
  }

  async _animateOut(el, transition) {
    return new Promise(resolve => {
      switch (transition) {
        case 'fade':
          el.style.transition = 'opacity 0.25s ease-in';
          el.style.opacity = '0';
          setTimeout(resolve, 250);
          break;

        case 'slide-up':
          el.style.transition = 'transform 0.3s ease-in';
          el.style.transform = 'translateY(-30%)';
          setTimeout(resolve, 300);
          break;

        case 'pop':
          el.style.transition = 'transform 0.2s ease-in, opacity 0.2s ease-in';
          el.style.transform = 'scale(0.8)';
          el.style.opacity = '0';
          setTimeout(resolve, 200);
          break;

        case 'none':
        default:
          el.style.opacity = '0';
          resolve();
          break;
      }
    });
  }
}
```

### Screen Registration Wiring

```javascript
const screenManager = new ScreenManager();

// Title Screen
screenManager.register('screen-title', {
  interactive: true,
  transition: 'fade',
  onEnter: () => { startTitleAnimation(); },
  onExit: () => { stopTitleAnimation(); }
});

// Workshop
screenManager.register('screen-workshop', {
  interactive: true,
  transition: 'slide-up',
  onEnter: () => { initWorkshop(); },
  onExit: () => { saveCurrentCraft(); }
});

// Course Select
screenManager.register('screen-courses', {
  interactive: true,
  transition: 'pop',
  onEnter: () => { populateCourseGrid(); },
  onExit: () => {}
});

// Ramp Run
screenManager.register('screen-ramp', {
  interactive: true,
  transition: 'fade',
  onEnter: () => { initRampSequence(); },
  onExit: () => { cleanupRampUI(); }
});

// Flight HUD (special -- overlays during flight, not a full-screen transition)
screenManager.register('screen-hud', {
  interactive: false, // pointer-events: none except pause button
  transition: 'fade',
  onEnter: () => { startHUDUpdates(); },
  onExit: () => { stopHUDUpdates(); }
});

// Splash
screenManager.register('screen-splash', {
  interactive: false,
  transition: 'pop',
  onEnter: () => { playSplashAnimation(); },
  onExit: () => {}
});

// Results
screenManager.register('screen-results', {
  interactive: true,
  transition: 'slide-up',
  onEnter: (data) => { populateResults(data); },
  onExit: () => { cleanupResults(); }
});

// Pause
screenManager.register('screen-pause', {
  interactive: true,
  transition: 'pop',
  onEnter: () => { pauseGame(); },
  onExit: () => { resumeGame(); }
});

// Start on title
screenManager.show('screen-title');
```

---

## Typography & Color System

### Font Stack

```css
@import url('https://fonts.googleapis.com/css2?family=Boogaloo&family=Nunito:wght@600;700;800;900&display=swap');

:root {
  --font-display: 'Boogaloo', cursive;    /* Titles, big numbers, branding */
  --font-body: 'Nunito', sans-serif;       /* Body text, labels, buttons */
  --font-counter: 'Courier New', monospace; /* Score counters, distances */
}
```

### Color Palette

```css
:root {
  /* Sky & Environment */
  --sky-blue: #4A90D9;
  --sky-light: #87CEEB;
  --ocean: #0088AA;
  --ocean-deep: #005577;
  --water-foam: #B8E6F0;

  /* Brand & Accents */
  --red: #FF3344;
  --red-dark: #CC1122;
  --yellow: #FFD700;
  --yellow-glow: #FFEE55;
  --orange: #FF8800;
  --green: #44CC44;
  --green-dark: #229922;

  /* UI Chrome */
  --panel-bg: rgba(0, 0, 0, 0.75);
  --panel-border: rgba(255, 255, 255, 0.15);
  --text-primary: #FFFFFF;
  --text-secondary: #CCDDEE;
  --text-accent: var(--yellow);
  --text-shadow: 2px 2px 0 rgba(0, 0, 0, 0.5);

  /* Medal Colors */
  --gold: #FFD700;
  --silver: #C0C0C0;
  --bronze: #CD7F32;

  /* Danger / Warning */
  --danger: #FF4444;
  --warning: #FFAA00;

  /* Shadows & Depths */
  --shadow-light: 0 2px 8px rgba(0,0,0,0.3);
  --shadow-heavy: 0 4px 16px rgba(0,0,0,0.5);
  --shadow-btn: 0 4px 0 rgba(0,0,0,0.3);
}
```

### Button Styles

```css
.game-btn {
  font-family: var(--font-body);
  font-weight: 800;
  font-size: 1.2rem;
  padding: 14px 32px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  text-transform: uppercase;
  letter-spacing: 1px;
  color: white;
  text-shadow: var(--text-shadow);
  box-shadow: var(--shadow-btn);
  transition: transform 0.1s ease, box-shadow 0.1s ease;
  -webkit-tap-highlight-color: transparent;
  min-height: 44px;
  min-width: 44px;
}

.game-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 0 rgba(0,0,0,0.3);
}

.game-btn:active {
  transform: translateY(2px);
  box-shadow: 0 1px 0 rgba(0,0,0,0.3);
}

.game-btn-primary {
  background: linear-gradient(180deg, var(--red) 0%, var(--red-dark) 100%);
}

.game-btn-secondary {
  background: linear-gradient(180deg, var(--sky-blue) 0%, #3570B0 100%);
}

.game-btn-success {
  background: linear-gradient(180deg, var(--green) 0%, var(--green-dark) 100%);
}

.game-btn-gold {
  background: linear-gradient(180deg, var(--yellow) 0%, var(--orange) 100%);
  color: #333;
  text-shadow: 1px 1px 0 rgba(255,255,255,0.3);
}

/* Pulsing CTA button */
.game-btn-pulse {
  animation: btnPulse 1.5s ease-in-out infinite;
}

@keyframes btnPulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}
```

### Text Styles

```css
.title-text {
  font-family: var(--font-display);
  font-size: 4rem;
  color: var(--text-primary);
  text-shadow: 3px 3px 0 rgba(0,0,0,0.5), 0 0 20px rgba(255,215,0,0.3);
  text-align: center;
  line-height: 1.1;
}

.subtitle-text {
  font-family: var(--font-body);
  font-weight: 700;
  font-size: 1.4rem;
  color: var(--text-secondary);
  text-shadow: var(--text-shadow);
  text-align: center;
}

.stat-label {
  font-family: var(--font-body);
  font-weight: 600;
  font-size: 0.85rem;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 2px;
}

.stat-value {
  font-family: var(--font-counter);
  font-weight: 700;
  font-size: 2rem;
  color: var(--text-primary);
  text-shadow: var(--text-shadow);
}

.score-counter {
  font-family: var(--font-counter);
  font-weight: 700;
  font-size: 3rem;
  color: var(--yellow);
  text-shadow: 3px 3px 0 rgba(0,0,0,0.5);
}
```

---

## Flight HUD Layout

```
+---------------------------------------------+
| ALT |||||||...  18m    DIST: 247m        [||] |
|                                               |
|            (3D flight viewport)               |
|                                               |
|  SPD ||||||||...             [pitch arrow]    |
+---------------------------------------------+
```

- **Altitude**: Vertical bar (left edge), color-coded (red LOW, green MID, blue HIGH)
- **Distance**: Large counter (top center) -- the primary metric players watch
- **Speed**: Horizontal bar (bottom left)
- **Pitch indicator**: Subtle directional arrow (bottom right)
- **Pause button**: Top right, 44x44px touch target
- **All HUD elements**: `pointer-events: none` except the pause button

---

## Ramp Run HUD Layout

```
+---------------------------------------------+
|       (Side view of ramp + craft)            |
|                                              |
|  +---------------------------------------+   |
|  | POWER ||||||||||||....................  |   |
|  +---------------------------------------+   |
|                                              |
|            TAP TAP TAP!                      |
+---------------------------------------------+
```

- **Power meter**: Horizontal bar with colored segments (red/yellow/green zones)
- **Perfect zone**: Golden highlight segment on the power meter
- **"TAP!" text**: Bouncing animation, large font
- **Speed readout**: Small counter showing current ramp speed

---

## Results Screen Layout

```
+=============================+
|         SPLASH!             |
|                             |
|   Distance:      247m      |
|   Max Altitude:   32m      |
|   Style Bonus:  x1.75      |
|   ________________________  |
|   TOTAL SCORE:   4,320     |
|                             |
|   ** ** ** .. ..  SILVER    |
|   BEST: 312m               |
|                             |
|   [ FLY AGAIN ]            |
|   [ WORKSHOP  ]            |
+=============================+
```

- Score values animate (count up from 0)
- Medal animates in with scale-bounce
- "NEW RECORD!" banner appears if best distance beaten
- Buttons appear after animation completes

---

## Responsive Design

```css
/* Mobile-first base */
.title-text { font-size: 2.5rem; }
.stat-value { font-size: 1.5rem; }
.game-btn { font-size: 1rem; padding: 12px 24px; }

/* Tablet and up */
@media (min-width: 768px) {
  .title-text { font-size: 4rem; }
  .stat-value { font-size: 2rem; }
  .game-btn { font-size: 1.2rem; padding: 14px 32px; }
}

/* Desktop */
@media (min-width: 1200px) {
  .title-text { font-size: 5rem; }
  .stat-value { font-size: 2.5rem; }
}

/* Landscape phones -- constrain HUD height */
@media (max-height: 480px) and (orientation: landscape) {
  .title-text { font-size: 2rem; }
  .game-btn { padding: 8px 20px; font-size: 0.9rem; }
  #screen-results .results-panel { max-height: 90vh; overflow-y: auto; }
}

/* Safe area insets for notched phones */
@supports (padding: env(safe-area-inset-top)) {
  #screen-hud { padding-top: env(safe-area-inset-top); }
  .screen { padding-bottom: env(safe-area-inset-bottom); }
}
```

### Touch Target Sizes

All interactive elements must be **at least 44x44px** for reliable touch input:

```css
.touch-target {
  min-width: 44px;
  min-height: 44px;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

### Prevent Scroll / Bounce

```css
html, body {
  overflow: hidden;
  position: fixed;
  width: 100%;
  height: 100%;
}
```

```javascript
// Prevent pull-to-refresh and scroll bounce
document.addEventListener('touchmove', (e) => { e.preventDefault(); }, { passive: false });
document.addEventListener('gesturestart', (e) => { e.preventDefault(); });
```

---

## Implementation Checklist

1. [ ] Set up `#game-container` HTML structure with all screen divs
2. [ ] Implement base CSS (screen positioning, display toggle, z-index layers)
3. [ ] Load Google Fonts (Boogaloo + Nunito) via `@import` or `<link>`
4. [ ] Define CSS custom properties (color palette, shadows, fonts)
5. [ ] Implement `ScreenManager` class with register/show/transition methods
6. [ ] Build Title Screen (logo, "TAP TO FLY" pulse animation, background)
7. [ ] Build Workshop wrapper (delegates to craft-builder skill)
8. [ ] Build Course Select grid (lock/unlock states, best distances)
9. [ ] Build Ramp Run screen (power meter bar, tap instruction, countdown)
10. [ ] Build Flight HUD (altitude bar, distance counter, speed bar, pause btn)
11. [ ] Build Splash screen (1.5s "SPLASH!" animation)
12. [ ] Build Results screen (animated counters, medal reveal, buttons)
13. [ ] Build Pause overlay (dimmed background, resume/workshop/settings)
14. [ ] Implement floating text system ("+500 PERFECT!" style popups)
15. [ ] Implement crowd reaction text overlays
16. [ ] Implement score counter animation (count-up from 0)
17. [ ] Implement medal reveal animation (scale-bounce + sparkle)
18. [ ] Add responsive breakpoints and safe-area handling
19. [ ] Implement `GameDataManager` for localStorage persistence
20. [ ] Wire all screen transitions (title->workshop->ramp->flight->splash->results)
21. [ ] Add touch prevention (scroll, bounce, pull-to-refresh)
22. [ ] Test on mobile viewport sizes (375px, 414px, 768px)

---

## Test Prompts

Use these prompts to verify the UI skill is working correctly:

1. **"Show me the title screen"** -- Should render the FLUGTAG! logo with pulsing "TAP TO FLY" button over a 3D pier scene.

2. **"Build the flight HUD"** -- Should create the altitude bar, distance counter, speed indicator, and pause button as a non-interactive overlay.

3. **"Create the results screen with a score of 4320, distance 247m, altitude 32m, style 1.75x"** -- Should animate score counters, show medal, display stats.

4. **"Wire up the screen transitions from title through results"** -- Should implement the full ScreenManager flow with proper animations between each phase.

5. **"Add the power meter for the ramp run"** -- Should create a horizontal bar with colored segments, perfect zone highlight, and "TAP!" instruction.

6. **"Make it responsive for mobile"** -- Should apply proper breakpoints, touch targets, safe areas, and prevent scroll/bounce.
