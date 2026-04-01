# Game Loop Reference

Complete game loop architecture for the Flugtag Flight Engine. Covers the main
loop, state machine, per-frame updates for each phase, visual effects, and
object pooling.

---

## Table of Contents

1. [Main Loop and Delta Time](#main-loop-and-delta-time)
2. [State Machine Overview](#state-machine-overview)
3. [State: LOADING](#state-loading)
4. [State: RAMP_RUN](#state-ramp_run)
5. [State: LAUNCH](#state-launch)
6. [State: FLIGHT](#state-flight)
7. [State: SPLASH](#state-splash)
8. [State: RESULTS](#state-results)
9. [State Transition Functions](#state-transition-functions)
10. [Per-Frame FLIGHT Update (15 Steps)](#per-frame-flight-update)
11. [Per-Frame RAMP_RUN Update](#per-frame-ramp_run-update)
12. [Visual Effects](#visual-effects)
13. [Object Pooling](#object-pooling)

---

## Main Loop and Delta Time

The game runs on `requestAnimationFrame` with a capped delta time to prevent
physics explosions from frame drops.

```javascript
class GameLoop {
  constructor() {
    this.lastTime = 0;
    this.gameTime = 0;
    this.MAX_DT = 1 / 30; // Cap at 33ms (30fps minimum simulation)
    this.running = false;
    this.state = null;
    this.renderer = null;
    this.scene = null;
    this.camera = null;
  }

  start() {
    this.running = true;
    this.lastTime = performance.now();
    this._loop = this._loop.bind(this);
    requestAnimationFrame(this._loop);
  }

  stop() {
    this.running = false;
  }

  _loop(timestamp) {
    if (!this.running) return;

    // Calculate delta time in seconds
    let dt = (timestamp - this.lastTime) / 1000;
    this.lastTime = timestamp;

    // Cap delta time to prevent physics explosion
    if (dt > this.MAX_DT) dt = this.MAX_DT;

    // Skip very small deltas (tab regain focus artifact)
    if (dt < 0.001) {
      requestAnimationFrame(this._loop);
      return;
    }

    this.gameTime += dt;

    // Update current state
    this.update(dt);

    // Render
    this.renderer.render(this.scene, this.camera);

    // Next frame
    requestAnimationFrame(this._loop);
  }

  update(dt) {
    if (this.state) {
      this.state.update(dt, this.gameTime);
    }
  }
}
```

### Delta Time Safety

```javascript
// Additional safety in physics update
function safeUpdate(dt) {
  // Sub-step if dt is large (shouldn't happen with cap, but defensive)
  const MAX_SUBSTEP = 1 / 60;
  let remaining = dt;

  while (remaining > 0) {
    const step = Math.min(remaining, MAX_SUBSTEP);
    physicsStep(step);
    remaining -= step;
  }
}
```

---

## State Machine Overview

```
+------------------------------------------------------------------+
|                        STATE MACHINE                             |
+------------------------------------------------------------------+
|                                                                  |
|  LOADING ──────> RAMP_RUN ──────> LAUNCH ──────> FLIGHT          |
|     |                                              |             |
|     |                                              v             |
|     |                                           SPLASH           |
|     |                                              |             |
|     |                                              v             |
|     |                           RESULTS <──────────+             |
|     |                              |                             |
|     |                              v                             |
|     +<─────────────── (retry) ─────+                             |
|                                                                  |
+------------------------------------------------------------------+
```

Each state is an object with:
- `enter()` -- called once when transitioning into this state
- `update(dt, gameTime)` -- called every frame while in this state
- `exit()` -- called once when transitioning out of this state

```javascript
class StateMachine {
  constructor() {
    this.states = {};
    this.currentState = null;
    this.currentStateName = null;
  }

  addState(name, stateObj) {
    this.states[name] = stateObj;
  }

  transition(newStateName) {
    // Exit current state
    if (this.currentState && this.currentState.exit) {
      this.currentState.exit();
    }

    // Enter new state
    this.currentStateName = newStateName;
    this.currentState = this.states[newStateName];

    if (this.currentState && this.currentState.enter) {
      this.currentState.enter();
    }

    console.log(`State: ${newStateName}`);
  }

  update(dt, gameTime) {
    if (this.currentState && this.currentState.update) {
      this.currentState.update(dt, gameTime);
    }
  }
}
```

---

## State: LOADING

Initialization state. Loads craft data, builds the 3D scene, and prepares
all assets before gameplay begins.

```javascript
const LoadingState = {
  enter() {
    showLoadingScreen();

    // 1. Load craft data
    const craftData = JSON.parse(
      window.storage?.getItem('currentCraft') || '{}'
    );
    game.craftStats = craftData.stats || { lift: 3, weight: 3, style: 3 };
    game.craftColors = craftData.colors || { primary: '#FF4444', secondary: '#FFFFFF' };

    // 2. Derive physics
    game.physics = new FlightPhysics(game.craftStats);

    // 3. Initialize Three.js scene
    initRenderer();
    initScene();
    initLights();

    // 4. Build environment
    buildSkyGradient();
    buildWaterPlane();
    buildRampAndPier();
    buildCrowdSpectators();
    buildCloudLayer();
    buildDistanceMarkers();

    // 5. Build or load craft model
    if (craftData.model) {
      loadGLBModel(craftData.model, () => {
        this._ready();
      });
    } else {
      buildProceduralCraft(game.craftStats, game.craftColors);
      this._ready();
    }

    // 6. Initialize systems
    game.thermals = generateThermals(1000); // 1km course
    game.windGusts = [];
    game.birdFlocks = generateBirdFlocks(1000);
    game.powerMeter = new PowerMeter(game.physics.rampSpeedPenalty);
    game.pitchController = new PitchController();
    game.flightCamera = new FlightCamera(camera);
    game.hud = createHUD();

    // 7. Initialize object pools
    game.splashPool = new ObjectPool(() => createSplashDroplet(), 60);
    game.trailPool = new ObjectPool(() => createTrailParticle(), 40);
    game.featherPool = new ObjectPool(() => createFeather(), 20);
  },

  _ready() {
    hideLoadingScreen();
    game.stateMachine.transition('RAMP_RUN');
  },

  update(dt) {
    // Loading spinner animation
    updateLoadingSpinner(dt);
  },

  exit() {
    hideLoadingScreen();
  }
};
```

---

## State: RAMP_RUN

Player rapidly taps/clicks to build speed on the power meter. The craft
rolls down the ramp, pushed by the team.

```javascript
const RampRunState = {
  enter() {
    // Position craft at top of ramp
    game.craftMesh.position.set(-15, game.physics.RAMP_HEIGHT + 2, 0);
    game.craftMesh.rotation.z = -game.physics.RAMP_ANGLE * (Math.PI / 180);

    // Position camera for close ramp view
    game.flightCamera.setPhase('RAMP_RUN');

    // Show power meter UI
    showPowerMeterUI();

    // Reset power meter
    game.powerMeter.power = 0;
    game.powerMeter.tapTimes = [];

    // Track ramp position
    this.rampProgress = 0; // 0 (top) to 1 (edge)
    this.rampLength = 15; // meters
    this.currentSpeed = 0;

    // Show "TAP!" instruction
    showInstruction('TAP to build speed!');

    // Bind tap handlers
    this._onTap = (e) => {
      e.preventDefault();
      game.powerMeter.tap(performance.now() / 1000);
      triggerTapFeedback();
    };
    window.addEventListener('mousedown', this._onTap);
    window.addEventListener('touchstart', this._onTap);
    window.addEventListener('keydown', (e) => {
      if (e.code === 'Space' || e.code === 'ArrowUp') {
        this._onTap(e);
      }
    });
  },

  update(dt, gameTime) {
    // Update power meter decay
    game.powerMeter.update(dt);

    // Calculate current speed from power
    this.currentSpeed = game.powerMeter.getLaunchSpeed();

    // Move craft down ramp
    if (this.currentSpeed > 0) {
      this.rampProgress += (this.currentSpeed / this.rampLength) * dt;
    }

    // Update craft position on ramp
    const rampAngleRad = game.physics.RAMP_ANGLE * (Math.PI / 180);
    const dx = this.rampProgress * this.rampLength;
    game.craftMesh.position.x = -15 + dx * Math.cos(rampAngleRad);
    game.craftMesh.position.y = (game.physics.RAMP_HEIGHT + 2)
                               - dx * Math.sin(rampAngleRad);

    // Update team characters (running alongside)
    updateTeamCharacters(this.rampProgress, this.currentSpeed);

    // Update crowd cheering (louder with higher power)
    updateCrowdVolume(game.powerMeter.power);

    // Visual feedback: speed lines intensity
    updateSpeedLines(game.powerMeter.power);

    // Camera slight shake based on power
    if (game.powerMeter.power > 0.5) {
      triggerCameraShake(0.02 * game.powerMeter.power, 0.05);
    }

    // Update power meter UI
    updatePowerMeterUI(game.powerMeter.power, game.powerMeter.isPerfect());

    // Update camera
    game.flightCamera.update(game.craftMesh.position, dt);

    // Check if reached ramp edge
    if (this.rampProgress >= 1.0) {
      game.stateMachine.transition('LAUNCH');
    }
  },

  exit() {
    hidePowerMeterUI();
    hideInstruction();
    window.removeEventListener('mousedown', this._onTap);
    window.removeEventListener('touchstart', this._onTap);
  }
};
```

---

## State: LAUNCH

Brief cinematic transition as the craft leaves the ramp edge. Lasts ~0.5 seconds.

```javascript
const LaunchState = {
  enter() {
    this.timer = 0;
    this.duration = 0.5; // seconds

    // Calculate launch velocity
    const launch = calculateLaunch(game.powerMeter, game.physics.RAMP_ANGLE);
    game.physics.vx = launch.vx;
    game.physics.vy = launch.vy;
    game.physics.px = 0;
    game.physics.py = game.physics.RAMP_HEIGHT;
    game.physics.pitch = game.physics.RAMP_ANGLE; // Match ramp angle initially

    // Track bonuses
    game.bonuses = {
      perfectLaunch: launch.perfect,
      thermalChain: 0,
      lowRiderTime: 0,
      impactSpeed: 0,
    };

    // Camera transition
    game.flightCamera.setPhase('LAUNCH');

    // Visual: brief slow-motion feel
    this.timeScale = 0.5; // Half speed

    // Crowd reaction
    if (launch.perfect) {
      triggerCrowdCheer('wild');
      showFloatingText('PERFECT LAUNCH!', '#FFD700');
    } else if (game.powerMeter.power > 0.5) {
      triggerCrowdCheer('normal');
    } else {
      triggerCrowdGasp();
    }

    // Team characters wave goodbye
    triggerTeamWave();

    // Show launch speed
    const speed = Math.sqrt(launch.vx * launch.vx + launch.vy * launch.vy);
    showFloatingText(`${speed.toFixed(1)} m/s`, '#FFFFFF');
  },

  update(dt, gameTime) {
    this.timer += dt;

    // Slow-motion effect ramps back to normal
    const effectiveDt = dt * this.timeScale;
    this.timeScale = Math.min(1.0, this.timeScale + dt * 2); // Ramp to 1.0

    // Minimal physics during launch (just move forward)
    game.physics.px += game.physics.vx * effectiveDt;
    game.physics.py += game.physics.vy * effectiveDt;

    // Update craft mesh position
    game.craftMesh.position.x = game.physics.px;
    game.craftMesh.position.y = game.physics.py;

    // Camera follow
    game.flightCamera.update(game.craftMesh.position, dt);

    // Transition to flight after duration
    if (this.timer >= this.duration) {
      game.stateMachine.transition('FLIGHT');
    }
  },

  exit() {
    // Nothing to clean up
  }
};
```

---

## State: FLIGHT

The main gameplay state. Full physics simulation, player controls pitch,
environmental interactions, HUD updates.

```javascript
const FlightState = {
  enter() {
    // Camera to flight mode
    game.flightCamera.setPhase('FLIGHT');

    // Show HUD
    showHUD();

    // Start wind trail particles
    game.windTrailActive = true;

    // Initialize tracking
    this.lastThermalState = false;
    this.thermalChainTimer = 0;
    this.lowRiderTimer = 0;

    // Schedule wind gusts
    game.windGusts = scheduleWindGusts(60); // 60 seconds max flight

    showInstruction('Pitch UP/DOWN to glide!', 2.0); // Show for 2 seconds
  },

  update(dt, gameTime) {
    // === THE 15-STEP FLIGHT UPDATE ===
    // See "Per-Frame FLIGHT Update" section below for full detail.

    // 1. Update pitch from input
    game.pitchController.update();
    game.physics.setTargetPitch(game.pitchController.pitchInput);

    // 2. Handle boost input
    game.physics.boostActive = game.pitchController._keys?.['Space'] || false;

    // 3. Apply flight physics (lift, drag, gravity, AoA, stall)
    const result = game.physics.update(
      dt,
      game.thermals,
      game.windGusts,
      gameTime
    );

    // 4. Check bird collisions
    for (const flock of game.birdFlocks) {
      if (flock.checkCollision(game.physics.px, game.physics.py)) {
        const effect = flock.applyCollision(game.physics);
        if (effect.cameraShake) {
          triggerCameraShake(effect.cameraShake.intensity, effect.cameraShake.duration);
        }
        if (effect.featherBurst) {
          spawnFeatherBurst(game.physics.px, game.physics.py);
        }
      }
    }

    // 5. Track thermal chains
    let inThermalNow = false;
    for (const thermal of game.thermals) {
      if (thermal.getUpwardAccel(game.physics.px, game.physics.py) > 0) {
        inThermalNow = true;
        break;
      }
    }
    if (inThermalNow && !this.lastThermalState) {
      // Entered a thermal
      if (this.thermalChainTimer > 0) {
        game.bonuses.thermalChain++;
      } else {
        game.bonuses.thermalChain = 1;
      }
      this.thermalChainTimer = 3.0; // 3 second window for chain
    }
    if (this.thermalChainTimer > 0) {
      this.thermalChainTimer -= dt;
    }
    this.lastThermalState = inThermalNow;

    // 6. Track low-rider bonus
    if (game.physics.py < 3.0 && game.physics.py > 0) {
      game.bonuses.lowRiderTime += dt;
    }

    // 7. Update craft mesh position and rotation
    game.craftMesh.position.x = game.physics.px;
    game.craftMesh.position.y = game.physics.py;
    game.craftMesh.rotation.z = game.physics.pitch * (Math.PI / 180);

    // 8. Update wind trail particles
    if (game.windTrailActive) {
      updateWindTrail(game.physics.px, game.physics.py, game.physics.getSpeed(), dt);
    }

    // 9. Update boost flame visual
    updateBoostFlame(game.physics.boostActive && game.physics.currentBoostFuel > 0);

    // 10. Update camera
    game.flightCamera.update(game.craftMesh.position, dt);

    // 11. Update environment elements
    updateClouds(game.physics.px, dt);
    updateDistanceMarkers(game.physics.px);
    updateBoats(game.physics.px, dt);
    updateBirdFlocks(game.birdFlocks, dt, gameTime);
    updateThermalVisuals(game.thermals, game.physics.px, dt);

    // 12. Check altitude zone and update visual cues
    const altZone = getAltitudeZone(game.physics.py);
    updateAltitudeVisuals(altZone);

    // 13. Update HUD
    updateHUD(game.hud, game.physics);

    // 14. Update stall warning visuals
    if (game.physics.isStalling) {
      showStallWarning();
      triggerCraftWobble(dt);
    } else {
      hideStallWarning();
    }

    // 15. Check splash condition
    if (result === 'SPLASH') {
      game.bonuses.impactSpeed = Math.abs(game.physics.vy);
      game.stateMachine.transition('SPLASH');
    }
  },

  exit() {
    game.windTrailActive = false;
    hideStallWarning();
  }
};
```

---

## State: SPLASH

Water landing with particle effects and distance measurement.

```javascript
const SplashState = {
  enter() {
    this.timer = 0;
    this.duration = 2.0; // seconds before results

    // Lock final distance
    game.finalDistance = game.physics.distance;

    // Camera zoom to impact
    game.flightCamera.setPhase('SPLASH');

    // Trigger splash effects
    const splashResult = triggerSplash(game.physics, game.scene);

    // Hide flight HUD elements
    hideStallWarning();

    // Show big distance number
    showDistanceReveal(game.finalDistance);

    // Splash sound
    playSplashSound(splashResult.splashScale);

    // Camera shake
    triggerCameraShake(0.5 * splashResult.splashScale, 0.4);

    // Start craft bobbing
    this.bobTime = 0;
  },

  update(dt, gameTime) {
    this.timer += dt;
    this.bobTime += dt;

    // Update splash particles (droplets falling, rings expanding)
    updateSplashParticles(dt);

    // Craft bobs on water
    game.craftMesh.position.y = Math.sin(this.bobTime * 2) * 0.15;
    game.craftMesh.rotation.z = Math.sin(this.bobTime * 1.5) * 0.05;

    // Camera slowly pulls back
    game.flightCamera.update(game.craftMesh.position, dt);

    // Transition to results
    if (this.timer >= this.duration) {
      game.stateMachine.transition('RESULTS');
    }
  },

  exit() {
    clearSplashParticles();
  }
};
```

---

## State: RESULTS

Score calculation, medal award, and results screen.

```javascript
const ResultsState = {
  enter() {
    // Camera pulls way back
    game.flightCamera.setPhase('RESULTS');

    // Calculate score
    game.scoreResult = calculateScore(game.physics, game.bonuses);

    // Build results UI
    this.ui = buildResultsUI(game.scoreResult, game.finalDistance);
    document.body.appendChild(this.ui);

    // Animate score reveal (counter rolls up)
    this.scoreRevealTimer = 0;
    this.scoreRevealDuration = 2.0;
    this.bonusRevealTimer = 0;

    // Medal reveal delay
    this.medalRevealed = false;
    this.medalRevealTime = 2.5;

    // Bind retry button
    this.ui.querySelector('.retry-btn').addEventListener('click', () => {
      game.stateMachine.transition('LOADING');
    });
  },

  update(dt, gameTime) {
    this.scoreRevealTimer += dt;

    // Animate score counter
    const revealProgress = Math.min(this.scoreRevealTimer / this.scoreRevealDuration, 1.0);
    const easedProgress = 1 - Math.pow(1 - revealProgress, 3); // Ease out cubic
    const displayScore = Math.round(game.scoreResult.finalScore * easedProgress);
    this.ui.querySelector('.score-value').textContent = displayScore;

    // Reveal bonuses one by one
    const bonusList = game.scoreResult.bonusList;
    for (let i = 0; i < bonusList.length; i++) {
      const revealTime = 1.0 + i * 0.4;
      if (this.scoreRevealTimer >= revealTime) {
        const el = this.ui.querySelector(`.bonus-${i}`);
        if (el && !el.classList.contains('revealed')) {
          el.classList.add('revealed');
          playBonusSound();
        }
      }
    }

    // Medal reveal
    if (!this.medalRevealed && this.scoreRevealTimer >= this.medalRevealTime) {
      this.medalRevealed = true;
      revealMedal(this.ui, game.scoreResult.medal);
      playMedalSound(game.scoreResult.medal);
    }

    // Gentle camera movement
    game.flightCamera.update(game.craftMesh.position, dt);
  },

  exit() {
    if (this.ui && this.ui.parentNode) {
      this.ui.parentNode.removeChild(this.ui);
    }
  }
};

function buildResultsUI(scoreResult, distance) {
  const div = document.createElement('div');
  div.className = 'results-screen';
  div.innerHTML = `
    <div class="results-card">
      <h1>FLIGHT COMPLETE</h1>
      <div class="distance-result">${distance.toFixed(1)}m</div>
      <div class="score-row">
        <span>Score:</span>
        <span class="score-value">0</span>
      </div>
      <div class="multiplier-row">
        <span>Style Multiplier:</span>
        <span>${scoreResult.styleMultiplier.toFixed(2)}x</span>
      </div>
      <div class="bonuses">
        ${scoreResult.bonusList.map((b, i) => `
          <div class="bonus-item bonus-${i}">
            <span>${b.name}</span>
            <span>+${b.value}</span>
          </div>
        `).join('')}
      </div>
      <div class="medal-container">
        <div class="medal hidden"></div>
      </div>
      <button class="retry-btn">FLY AGAIN</button>
    </div>
  `;
  return div;
}
```

---

## State Transition Functions

All transitions go through the state machine, which calls `exit()` on the
current state and `enter()` on the new state.

```javascript
function initStateMachine() {
  const sm = new StateMachine();

  sm.addState('LOADING',  LoadingState);
  sm.addState('RAMP_RUN', RampRunState);
  sm.addState('LAUNCH',   LaunchState);
  sm.addState('FLIGHT',   FlightState);
  sm.addState('SPLASH',   SplashState);
  sm.addState('RESULTS',  ResultsState);

  return sm;
}

// Usage
const game = {
  stateMachine: initStateMachine(),
  // ... other game properties
};

game.stateMachine.transition('LOADING');
```

### Enter/Exit Hooks Summary

| State    | Enter                                        | Exit                          |
|----------|----------------------------------------------|-------------------------------|
| LOADING  | Init scene, load assets, derive physics      | Hide loading screen           |
| RAMP_RUN | Position craft on ramp, show power meter, bind taps | Remove tap handlers, hide power UI |
| LAUNCH   | Calculate velocity, slow-mo, crowd reaction  | --                            |
| FLIGHT   | Show HUD, start trails, schedule gusts       | Stop trails, hide warnings    |
| SPLASH   | Lock distance, spawn particles, camera zoom  | Clear particles               |
| RESULTS  | Build results UI, start score animation      | Remove results UI             |

---

## Per-Frame FLIGHT Update

The 15 steps executed every frame during the FLIGHT state. This is the heart
of the game.

### Step 1: Update Pitch from Input

```javascript
// Read current input state
game.pitchController.update();

// Map input to target pitch (-30 to +30 degrees)
game.physics.setTargetPitch(game.pitchController.pitchInput);
```

The pitch controller reads keyboard, touch, or accelerometer input and
produces a value from -1 to +1. The physics engine smoothly interpolates
the current pitch toward the target at `PITCH_SPEED` degrees per second.

### Step 2: Apply Flight Physics

```javascript
const result = game.physics.update(dt, game.thermals, game.windGusts, gameTime);
```

Inside `FlightPhysics.update()`:
- Interpolate pitch toward target
- Calculate angle of attack
- Calculate lift (with stall, ground effect, altitude modifiers)
- Calculate drag
- Apply gravity
- Apply boost if active
- Sum all forces
- Integrate velocity (F/m * dt)
- Integrate position (v * dt)
- Clamp velocities to terminal limits
- Check splash condition (py <= 0)

### Step 3: Apply Thermal Forces

Handled internally by `FlightPhysics.update()` via the thermals array. Each
thermal's `getUpwardAccel()` is evaluated at the craft's current position.

```javascript
// Inside FlightPhysics.update():
for (const thermal of thermals) {
  const upAccel = thermal.getUpwardAccel(this.px, this.py);
  envFy += upAccel * this.mass;
}
```

### Step 4: Apply Wind Gust Forces

Also handled internally. Each gust is evaluated at the current game time.

```javascript
// Inside FlightPhysics.update():
for (const gust of gusts) {
  const gustAccel = gust.getAcceleration(gameTime);
  envFx += gustAccel * this.mass;
}
```

### Step 5: Check Bird Collisions

```javascript
for (const flock of game.birdFlocks) {
  if (flock.checkCollision(game.physics.px, game.physics.py)) {
    const effect = flock.applyCollision(game.physics);
    // Trigger camera shake and feather burst
  }
}
```

### Step 6: Check Collectible Pickups

(Optional feature -- bonus items floating in the air.)

```javascript
for (const collectible of game.collectibles) {
  if (collectible.checkPickup(game.physics.px, game.physics.py)) {
    applyCollectibleEffect(collectible, game.physics);
    collectible.despawn();
  }
}
```

### Step 7: Update Craft Position

```javascript
game.craftMesh.position.x = game.physics.px;
game.craftMesh.position.y = game.physics.py;
```

### Step 8: Update Craft Visual Rotation

```javascript
// Craft rotation matches pitch angle
game.craftMesh.rotation.z = game.physics.pitch * (Math.PI / 180);
```

### Step 9: Update Wind Trail Particles

```javascript
function updateWindTrail(px, py, speed, dt) {
  // Spawn new trail particle behind craft
  if (speed > 2) {
    const particle = game.trailPool.get();
    if (particle) {
      particle.position.set(px - 1.5, py, 0);
      particle.userData.life = 0.5;
      particle.userData.age = 0;
      particle.userData.vx = -speed * 0.3 + (Math.random() - 0.5) * 2;
      particle.userData.vy = (Math.random() - 0.5) * 1;
      particle.visible = true;
      particle.scale.setScalar(0.1 + speed * 0.005);
    }
  }

  // Update existing trail particles
  game.trailPool.each(p => {
    if (!p.visible) return;
    p.userData.age += dt;
    if (p.userData.age >= p.userData.life) {
      p.visible = false;
      game.trailPool.release(p);
      return;
    }
    p.position.x += p.userData.vx * dt;
    p.position.y += p.userData.vy * dt;
    const alpha = 1.0 - (p.userData.age / p.userData.life);
    p.material.opacity = alpha * 0.6;
  });
}
```

### Step 10: Update Camera

```javascript
game.flightCamera.update(game.craftMesh.position, dt);
```

The camera smoothly follows the craft using exponential lerp, maintaining
a side-scrolling view offset.

### Step 11: Update Environment Elements

```javascript
// Scroll clouds with parallax (slower than craft)
function updateClouds(craftX, dt) {
  for (const cloud of game.clouds) {
    // Parallax: clouds move at 30% of craft speed
    cloud.position.x = cloud.userData.baseX - craftX * 0.3;

    // Wrap clouds that scroll off screen
    if (cloud.position.x < craftX - 60) {
      cloud.position.x = craftX + 60;
      cloud.userData.baseX = cloud.position.x + craftX * 0.3;
    }
  }
}

// Update distance markers (buoys on water)
function updateDistanceMarkers(craftX) {
  for (const marker of game.distanceMarkers) {
    // Show/hide based on distance from craft
    const dx = marker.userData.distance - craftX;
    marker.visible = dx > -20 && dx < 80;
  }
}

// Update boats on water
function updateBoats(craftX, dt) {
  for (const boat of game.boats) {
    // Gentle bobbing
    boat.position.y = Math.sin(game.gameTime * 1.5 + boat.userData.phase) * 0.3;
    // Scroll with parallax
    boat.position.x = boat.userData.baseX - craftX * 0.15;
  }
}

// Update bird flocks
function updateBirdFlocks(flocks, dt, gameTime) {
  for (const flock of flocks) {
    flock.update(dt, gameTime);
    // Update bird mesh positions
    for (let i = 0; i < flock.birds.length; i++) {
      const bird = flock.birds[i];
      const mesh = flock.meshes[i];
      if (mesh) {
        mesh.position.x = flock.x + bird.offsetX;
        mesh.position.y = flock.y + bird.offsetY;
        // Wing flap animation
        mesh.rotation.z = Math.sin(gameTime * bird.flapSpeed + bird.flapPhase) * 0.3;
      }
    }
  }
}

// Update thermal visual indicators
function updateThermalVisuals(thermals, craftX, dt) {
  for (const thermal of thermals) {
    const dx = thermal.x - craftX;
    // Only render nearby thermals
    if (Math.abs(dx) < 50) {
      updateThermalShimmer(thermal, dt);
      updateThermalParticles(thermal, dt);
    }
  }
}
```

### Step 12: Check Altitude Zones

```javascript
function getAltitudeZone(altitude) {
  if (altitude < 8) return 'LOW';
  if (altitude < 25) return 'MID';
  return 'HIGH';
}

function updateAltitudeVisuals(zone) {
  switch (zone) {
    case 'LOW':
      // Show water spray particles
      // Tint HUD altitude bar blue
      setAltBarColor('#4488CC');
      break;
    case 'MID':
      // Normal visuals
      setAltBarColor('#44CC44');
      break;
    case 'HIGH':
      // Slightly desaturated colors
      // Show "thin air" visual effect
      setAltBarColor('#CC8844');
      break;
  }
}
```

### Step 13: Update HUD

```javascript
updateHUD(game.hud, game.physics);
```

Updates altitude bar, distance counter, speed display, pitch indicator,
and boost fuel bar. See SKILL.md HUD section for layout.

### Step 14: Check Splash Condition

```javascript
if (result === 'SPLASH') {
  game.bonuses.impactSpeed = Math.abs(game.physics.vy);
  game.stateMachine.transition('SPLASH');
}
```

### Step 15: Render

Handled by the main game loop after `update()` returns:

```javascript
this.renderer.render(this.scene, this.camera);
```

---

## Per-Frame RAMP_RUN Update

```javascript
function rampRunUpdate(dt, gameTime) {
  // 1. Process tap input (handled via event listeners)

  // 2. Update power meter (decay when not tapping)
  game.powerMeter.update(dt);

  // 3. Calculate current speed from power level
  const speed = game.powerMeter.getLaunchSpeed();

  // 4. Advance craft along ramp
  rampProgress += (speed / rampLength) * dt;
  rampProgress = Math.min(rampProgress, 1.0);

  // 5. Position craft on ramp surface
  const rampAngleRad = rampAngle * (Math.PI / 180);
  const dx = rampProgress * rampLength;
  craftMesh.position.x = rampStartX + dx * Math.cos(rampAngleRad);
  craftMesh.position.y = rampStartY - dx * Math.sin(rampAngleRad);

  // 6. Animate team characters
  updateTeamCharacters(rampProgress, speed);

  // 7. Update crowd reactions
  updateCrowdVolume(game.powerMeter.power);

  // 8. Visual effects (speed lines, camera shake)
  updateSpeedLines(game.powerMeter.power);
  if (game.powerMeter.power > 0.5) {
    triggerCameraShake(0.02 * game.powerMeter.power, 0.05);
  }

  // 9. Update power meter UI
  updatePowerMeterUI(game.powerMeter.power, game.powerMeter.isPerfect());

  // 10. Update camera
  game.flightCamera.update(craftMesh.position, dt);

  // 11. Check launch condition
  if (rampProgress >= 1.0) {
    transitionToLaunch();
  }
}
```

---

## Visual Effects

### Wind Trails

Particle stream behind the craft during flight. Intensity scales with speed.

```javascript
function createTrailParticle() {
  const geo = new THREE.SphereGeometry(0.08, 4, 4);
  const mat = new THREE.MeshBasicMaterial({
    color: 0xFFFFFF,
    transparent: true,
    opacity: 0.6,
  });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.visible = false;
  mesh.userData = { life: 0, age: 0, vx: 0, vy: 0 };
  return mesh;
}
```

### Thermal Shimmer

Heat distortion effect near thermals. Implemented as subtle vertex displacement
on nearby geometry or a post-process effect.

```javascript
function updateThermalShimmer(thermal, dt) {
  // Shimmer mesh is a transparent plane with animated UV offset
  if (!thermal.shimmerMesh) {
    const geo = new THREE.PlaneGeometry(thermal.radius * 2, thermal.maxY);
    const mat = new THREE.MeshBasicMaterial({
      color: 0xFFFFCC,
      transparent: true,
      opacity: 0.03,
      side: THREE.DoubleSide,
    });
    thermal.shimmerMesh = new THREE.Mesh(geo, mat);
    thermal.shimmerMesh.position.set(thermal.x, thermal.maxY / 2, 0);
    game.scene.add(thermal.shimmerMesh);
  }

  // Animate opacity (pulsing)
  thermal.shimmerMesh.material.opacity = 0.02 + Math.sin(game.gameTime * 3) * 0.01;
}
```

### Splash Particles

See SKILL.md splash section. Three types:
1. **Water droplets**: Small spheres launched radially from impact point, affected by gravity
2. **Splash rings**: Expanding torus geometries that fade out
3. **Spray mist**: Very small particles that drift upward briefly

```javascript
function updateSplashParticles(dt) {
  // Update droplets
  game.splashPool.each(droplet => {
    if (!droplet.visible) return;
    droplet.userData.age += dt;
    if (droplet.userData.age >= droplet.userData.life) {
      droplet.visible = false;
      game.splashPool.release(droplet);
      return;
    }
    // Gravity on droplets
    droplet.userData.vy -= 9.8 * dt;
    droplet.position.x += droplet.userData.vx * dt;
    droplet.position.y += droplet.userData.vy * dt;

    // Remove if below water
    if (droplet.position.y < 0) {
      droplet.visible = false;
      game.splashPool.release(droplet);
    }
  });

  // Update splash rings
  for (const ring of game.splashRings) {
    if (!ring.visible) continue;
    ring.userData.age += dt;
    const scale = 1 + ring.userData.expandRate * ring.userData.age;
    ring.scale.set(scale, scale, 1);
    ring.material.opacity = Math.max(0, 0.8 - ring.userData.fadeRate * ring.userData.age);
    if (ring.material.opacity <= 0) ring.visible = false;
  }
}
```

### Camera Shake

Used on bird collision, high-power ramp run, and splash impact.

```javascript
class CameraShake {
  constructor() {
    this.intensity = 0;
    this.duration = 0;
    this.elapsed = 0;
    this.active = false;
  }

  trigger(intensity, duration) {
    this.intensity = intensity;
    this.duration = duration;
    this.elapsed = 0;
    this.active = true;
  }

  update(camera, dt) {
    if (!this.active) return;
    this.elapsed += dt;

    if (this.elapsed >= this.duration) {
      this.active = false;
      return;
    }

    const remaining = 1.0 - (this.elapsed / this.duration);
    const shakeX = (Math.random() - 0.5) * 2 * this.intensity * remaining;
    const shakeY = (Math.random() - 0.5) * 2 * this.intensity * remaining;

    camera.position.x += shakeX;
    camera.position.y += shakeY;
  }
}
```

---

## Object Pooling

Reuse objects instead of creating/destroying them each frame. Critical for
particles and environment elements.

```javascript
class ObjectPool {
  constructor(factory, initialSize) {
    this.factory = factory;
    this.pool = [];
    this.active = [];

    // Pre-allocate
    for (let i = 0; i < initialSize; i++) {
      const obj = this.factory();
      obj.visible = false;
      this.pool.push(obj);
    }
  }

  /**
   * Get an object from the pool. Returns null if pool is exhausted.
   */
  get() {
    let obj = this.pool.pop();
    if (!obj) {
      // Pool exhausted -- create new (should be rare if sized correctly)
      obj = this.factory();
    }
    obj.visible = true;
    this.active.push(obj);
    return obj;
  }

  /**
   * Return an object to the pool.
   */
  release(obj) {
    obj.visible = false;
    const idx = this.active.indexOf(obj);
    if (idx !== -1) {
      this.active.splice(idx, 1);
    }
    this.pool.push(obj);
  }

  /**
   * Iterate over all active objects.
   */
  each(callback) {
    // Iterate in reverse so callbacks can release safely
    for (let i = this.active.length - 1; i >= 0; i--) {
      callback(this.active[i]);
    }
  }

  /**
   * Release all active objects back to pool.
   */
  releaseAll() {
    while (this.active.length > 0) {
      this.release(this.active[0]);
    }
  }

  /**
   * Add all pool objects to a Three.js scene.
   */
  addToScene(scene) {
    for (const obj of this.pool) {
      scene.add(obj);
    }
    for (const obj of this.active) {
      scene.add(obj);
    }
  }
}
```

### Pool Sizing Guide

| Pool               | Recommended Size | Purpose                        |
|--------------------|------------------|--------------------------------|
| Splash droplets    | 60               | Water droplet particles        |
| Wind trail         | 40               | Trail behind craft             |
| Feathers           | 20               | Bird collision burst           |
| Thermal particles  | 30               | Rising dot particles           |
| Clouds             | 20               | Parallax cloud meshes          |
| Distance markers   | 15               | Buoys on water                 |
