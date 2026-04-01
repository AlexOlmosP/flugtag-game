# Persistence Reference

Data management, localStorage persistence, and game state for the Flugtag game.

---

## Table of Contents

1. [Storage Namespace](#storage-namespace)
2. [Data Keys](#data-keys)
3. [Default Values](#default-values)
4. [GameDataManager Class](#gamedatamanager-class)
5. [Data Migration Strategy](#data-migration-strategy)
6. [Error Handling](#error-handling)
7. [Course Unlock Logic](#course-unlock-logic)
8. [Medal Thresholds](#medal-thresholds)
9. [Debounced Save](#debounced-save)
10. [Debug Utilities](#debug-utilities)

---

## Storage Namespace

All localStorage keys use the `flugtag:` prefix to avoid collisions with other
apps on the same domain. The prefix is handled automatically by `GameDataManager`.

```
flugtag:craft
flugtag:best-distance
flugtag:highscore
flugtag:total-flights
flugtag:course-unlocks
flugtag:medals
flugtag:settings
flugtag:data-version
```

---

## Data Keys

| Key | Type | Description | Default |
|-----|------|-------------|---------|
| `flugtag:craft` | JSON object | Current craft configuration (matches craft-builder data model) | See defaults |
| `flugtag:best-distance` | JSON object | Best distance per course `{ "pier": 247, "cliff": 0 }` | `{}` |
| `flugtag:highscore` | JSON object | Best total score per course `{ "pier": 4320 }` | `{}` |
| `flugtag:total-flights` | number | Lifetime flight count across all courses | `0` |
| `flugtag:course-unlocks` | JSON array | List of unlocked course IDs | `["pier"]` |
| `flugtag:medals` | JSON object | Best medal per course `{ "pier": "silver" }` | `{}` |
| `flugtag:settings` | JSON object | User preferences (sound, controls) | See defaults |
| `flugtag:data-version` | number | Schema version for data migration | `1` |

---

## Default Values

```javascript
const DEFAULTS = {
  craft: {
    version: 1,
    name: 'My Flying Machine',
    fuselage: { type: 'bathtub', color: '#FF4444' },
    wings: { type: 'cardboard', color: '#FFFFFF' },
    propulsion: { type: 'none', enabled: false },
    decorations: [
      { type: 'team_banner', slot: 'fuselage_side', enabled: true },
      { type: 'streamers', slot: 'wing_tip', enabled: false },
      { type: 'mascot', slot: 'nose', enabled: false },
      { type: 'tail_flag', slot: 'tail', enabled: false }
    ],
    pilot: { helmet: 'aviator', helmetColor: '#2244FF', costume: 'jumpsuit' },
    stats: { lift: 2, weight: 3, style: 2 }
  },

  bestDistance: {},

  highscore: {},

  totalFlights: 0,

  courseUnlocks: ['pier'],

  medals: {},

  settings: {
    soundEnabled: true,
    musicEnabled: true,
    soundVolume: 0.8,
    musicVolume: 0.5,
    controlSensitivity: 1.0,  // 0.5 = low, 1.0 = normal, 1.5 = high
    invertPitch: false,
    hapticEnabled: true,
    effectsQuality: 'high'    // 'low' | 'medium' | 'high'
  },

  dataVersion: 1
};
```

---

## GameDataManager Class

Complete implementation with all CRUD operations, debounced saves, and error handling.

```javascript
class GameDataManager {
  constructor() {
    this.PREFIX = 'flugtag:';
    this.saveTimers = {};
    this.cache = {};
  }

  // ============================
  // Initialization
  // ============================

  init() {
    // Load all data into cache
    this.cache = {
      craft: this._load('craft', DEFAULTS.craft),
      bestDistance: this._load('best-distance', DEFAULTS.bestDistance),
      highscore: this._load('highscore', DEFAULTS.highscore),
      totalFlights: this._load('total-flights', DEFAULTS.totalFlights),
      courseUnlocks: this._load('course-unlocks', DEFAULTS.courseUnlocks),
      medals: this._load('medals', DEFAULTS.medals),
      settings: this._load('settings', DEFAULTS.settings),
      dataVersion: this._load('data-version', DEFAULTS.dataVersion)
    };

    // Run migrations if needed
    this._migrate();

    // Ensure pier is always unlocked
    if (!this.cache.courseUnlocks.includes('pier')) {
      this.cache.courseUnlocks.unshift('pier');
      this._save('course-unlocks', this.cache.courseUnlocks);
    }

    console.log('[GameData] Initialized. Flights:', this.cache.totalFlights);
  }

  // ============================
  // Craft
  // ============================

  getCraft() {
    return JSON.parse(JSON.stringify(this.cache.craft)); // deep copy
  }

  saveCraft(data) {
    if (!data || typeof data !== 'object') return;
    // Validate required fields
    if (!data.fuselage || !data.wings) {
      console.warn('[GameData] Invalid craft data, skipping save');
      return;
    }
    data.version = DEFAULTS.craft.version;
    this.cache.craft = data;
    this._debouncedSave('craft', data);
  }

  // ============================
  // Best Distance
  // ============================

  getBestDistance(course) {
    return this.cache.bestDistance[course] || 0;
  }

  getAllBestDistances() {
    return { ...this.cache.bestDistance };
  }

  saveBestDistance(course, distance) {
    if (typeof distance !== 'number' || distance < 0) return;
    const current = this.cache.bestDistance[course] || 0;
    if (distance > current) {
      this.cache.bestDistance[course] = distance;
      this._debouncedSave('best-distance', this.cache.bestDistance);
      return true; // new record
    }
    return false;
  }

  // ============================
  // High Score
  // ============================

  getHighScore(course) {
    return this.cache.highscore[course] || 0;
  }

  getAllHighScores() {
    return { ...this.cache.highscore };
  }

  saveHighScore(course, score) {
    if (typeof score !== 'number' || score < 0) return;
    const current = this.cache.highscore[course] || 0;
    if (score > current) {
      this.cache.highscore[course] = score;
      this._debouncedSave('highscore', this.cache.highscore);
      return true; // new record
    }
    return false;
  }

  // ============================
  // Medals
  // ============================

  getMedals(course) {
    return this.cache.medals[course] || null;
  }

  getAllMedals() {
    return { ...this.cache.medals };
  }

  saveMedal(course, medal) {
    const rank = { gold: 3, silver: 2, bronze: 1 };
    const current = this.cache.medals[course];
    const currentRank = rank[current] || 0;
    const newRank = rank[medal] || 0;

    // Only upgrade medals, never downgrade
    if (newRank > currentRank) {
      this.cache.medals[course] = medal;
      this._debouncedSave('medals', this.cache.medals);
      return true; // upgraded
    }
    return false;
  }

  // ============================
  // Course Unlocks
  // ============================

  getCourseUnlocks() {
    return [...this.cache.courseUnlocks];
  }

  isCourseUnlocked(course) {
    return this.cache.courseUnlocks.includes(course);
  }

  unlockCourse(course) {
    if (this.cache.courseUnlocks.includes(course)) return false;
    this.cache.courseUnlocks.push(course);
    this._save('course-unlocks', this.cache.courseUnlocks);
    console.log('[GameData] Unlocked course:', course);
    return true;
  }

  // ============================
  // Flight Counter
  // ============================

  getTotalFlights() {
    return this.cache.totalFlights;
  }

  incrementFlights() {
    this.cache.totalFlights++;
    this._debouncedSave('total-flights', this.cache.totalFlights);
    return this.cache.totalFlights;
  }

  // ============================
  // Settings
  // ============================

  getSettings() {
    return { ...this.cache.settings };
  }

  getSetting(key) {
    return this.cache.settings[key];
  }

  saveSettings(settings) {
    if (!settings || typeof settings !== 'object') return;
    // Merge with existing (partial updates allowed)
    this.cache.settings = { ...this.cache.settings, ...settings };
    this._debouncedSave('settings', this.cache.settings);
  }

  saveSetting(key, value) {
    this.cache.settings[key] = value;
    this._debouncedSave('settings', this.cache.settings);
  }

  // ============================
  // Reset
  // ============================

  resetAll() {
    const keys = [
      'craft', 'best-distance', 'highscore', 'total-flights',
      'course-unlocks', 'medals', 'settings', 'data-version'
    ];
    keys.forEach(key => {
      try {
        localStorage.removeItem(this.PREFIX + key);
      } catch (e) {
        console.warn('[GameData] Failed to remove', key, e);
      }
    });
    // Re-initialize with defaults
    this.cache = {};
    this.init();
    console.log('[GameData] All data reset to defaults');
  }

  resetProgress() {
    // Reset progress but keep settings and craft
    const keepCraft = this.getCraft();
    const keepSettings = this.getSettings();

    this.resetAll();

    this.saveCraft(keepCraft);
    this.saveSettings(keepSettings);
    console.log('[GameData] Progress reset (craft and settings preserved)');
  }

  // ============================
  // Internal: Load / Save
  // ============================

  _load(key, defaultValue) {
    try {
      const raw = localStorage.getItem(this.PREFIX + key);
      if (raw === null) return defaultValue;
      const parsed = JSON.parse(raw);
      // For objects, merge with defaults to fill missing keys
      if (typeof defaultValue === 'object' && !Array.isArray(defaultValue) && defaultValue !== null) {
        return { ...defaultValue, ...parsed };
      }
      return parsed;
    } catch (e) {
      console.warn(`[GameData] Failed to load "${key}", using default`, e);
      return defaultValue;
    }
  }

  _save(key, value) {
    try {
      localStorage.setItem(this.PREFIX + key, JSON.stringify(value));
    } catch (e) {
      console.error(`[GameData] Failed to save "${key}"`, e);
      // Handle quota exceeded
      if (e.name === 'QuotaExceededError' || e.code === 22) {
        console.warn('[GameData] Storage quota exceeded. Attempting cleanup...');
        this._handleQuotaExceeded();
      }
    }
  }

  /**
   * Debounced save - waits 300ms after the last call before writing.
   * Prevents excessive writes during rapid updates (e.g., tap sequences).
   */
  _debouncedSave(key, value) {
    if (this.saveTimers[key]) {
      clearTimeout(this.saveTimers[key]);
    }
    this.saveTimers[key] = setTimeout(() => {
      this._save(key, value);
      delete this.saveTimers[key];
    }, 300);
  }

  /**
   * Force-flush all pending debounced saves immediately.
   * Call this before page unload or critical transitions.
   */
  flushAll() {
    Object.keys(this.saveTimers).forEach(key => {
      clearTimeout(this.saveTimers[key]);
      delete this.saveTimers[key];
    });
    // Write current cache state
    this._save('craft', this.cache.craft);
    this._save('best-distance', this.cache.bestDistance);
    this._save('highscore', this.cache.highscore);
    this._save('total-flights', this.cache.totalFlights);
    this._save('course-unlocks', this.cache.courseUnlocks);
    this._save('medals', this.cache.medals);
    this._save('settings', this.cache.settings);
    this._save('data-version', this.cache.dataVersion);
  }

  _handleQuotaExceeded() {
    // Try to free space by removing non-essential data
    // In practice this is very unlikely for this game's data size
    console.warn('[GameData] Quota exceeded -- game data is minimal, something else is filling storage');
  }

  // ============================
  // Internal: Migration
  // ============================

  _migrate() {
    const version = this.cache.dataVersion;

    if (version < 1) {
      // Future: migration from v0 to v1
      // Example: rename keys, restructure data
      this.cache.dataVersion = 1;
      this._save('data-version', 1);
    }

    // Future migration example:
    // if (version < 2) {
    //   // v1 -> v2: add new field to craft data
    //   if (this.cache.craft && !this.cache.craft.newField) {
    //     this.cache.craft.newField = 'default';
    //   }
    //   this.cache.dataVersion = 2;
    //   this._save('data-version', 2);
    //   this._save('craft', this.cache.craft);
    // }
  }
}
```

---

## Data Migration Strategy

The `flugtag:data-version` key tracks the current schema version. When the game
loads, `GameDataManager._migrate()` runs and applies sequential migrations.

### Migration Rules

1. **Version field is mandatory**: Every saved craft includes `version: 1`
2. **Migrations are sequential**: v1 -> v2 -> v3, never skip
3. **Migrations are idempotent**: Running the same migration twice is safe
4. **Defaults fill gaps**: When loading objects, missing keys are filled from `DEFAULTS`
5. **Never delete user data silently**: Migrations transform, they do not destroy

### Adding a New Migration

```javascript
// In _migrate(), add a new block:
if (version < 2) {
  // Describe what changed in v2
  // e.g., "Added pilot.accessories array to craft data"

  // Transform existing data
  if (this.cache.craft && !this.cache.craft.pilot.accessories) {
    this.cache.craft.pilot.accessories = [];
  }

  // Update version
  this.cache.dataVersion = 2;
  this._save('data-version', 2);
  this._save('craft', this.cache.craft);

  console.log('[GameData] Migrated to v2');
}
```

---

## Error Handling

Every `localStorage` operation is wrapped in `try/catch`:

| Error | Cause | Handling |
|-------|-------|----------|
| `JSON.parse` failure | Corrupted data | Fall back to default value |
| `QuotaExceededError` | Storage full (5MB limit) | Log warning, attempt cleanup |
| `SecurityError` | Private browsing / iframe restrictions | Fall back to in-memory only |
| Missing key | First-time user | Return default value |
| Invalid data shape | Partial corruption | Merge with defaults to fill gaps |

### Fallback Mode

If localStorage is completely unavailable (private browsing, security restrictions),
the game operates in memory-only mode. All data works normally during the session
but is lost on page reload.

```javascript
class GameDataManager {
  constructor() {
    this.PREFIX = 'flugtag:';
    this.saveTimers = {};
    this.cache = {};
    this.storageAvailable = this._checkStorage();
  }

  _checkStorage() {
    try {
      const test = '__flugtag_test__';
      localStorage.setItem(test, '1');
      localStorage.removeItem(test);
      return true;
    } catch (e) {
      console.warn('[GameData] localStorage unavailable, running in memory-only mode');
      return false;
    }
  }

  _load(key, defaultValue) {
    if (!this.storageAvailable) return defaultValue;
    // ... (normal load logic)
  }

  _save(key, value) {
    if (!this.storageAvailable) return;
    // ... (normal save logic)
  }
}
```

---

## Course Unlock Logic

Courses are unlocked based on best distance achieved in the "pier" (default) course.

### Unlock Thresholds

| Course | Required Best Distance | Description |
|--------|----------------------|-------------|
| `pier` | 0m (always unlocked) | Default starting course |
| `cliff` | 200m in Pier | Windy Cliff - strong gusts |
| `rooftop` | 350m in Pier | Urban Rooftop - warm updrafts |
| `mountain` | 500m in Pier | Alpine Mountain - chaotic winds |

### Unlock Check Implementation

```javascript
const COURSE_UNLOCK_THRESHOLDS = {
  pier: 0,
  cliff: 200,
  rooftop: 350,
  mountain: 500
};

/**
 * Check and apply course unlocks based on current best distances.
 * Called after every flight results screen.
 * @returns {string[]} List of newly unlocked course IDs
 */
function checkCourseUnlocks() {
  const gameData = window.gameData;
  const bestPier = gameData.getBestDistance('pier');
  const newUnlocks = [];

  Object.entries(COURSE_UNLOCK_THRESHOLDS).forEach(([course, threshold]) => {
    if (bestPier >= threshold && !gameData.isCourseUnlocked(course)) {
      gameData.unlockCourse(course);
      newUnlocks.push(course);
    }
  });

  if (newUnlocks.length > 0) {
    console.log('[GameData] New unlocks:', newUnlocks);
    // Show unlock notification in UI
    newUnlocks.forEach(course => {
      showFloatingText('NEW COURSE UNLOCKED!', 'var(--gold)', { size: 2, y: 0.25, duration: 3000 });
    });
  }

  return newUnlocks;
}
```

---

## Medal Thresholds

Medals are awarded based on flight distance. Each course has its own thresholds.

| Course | Bronze | Silver | Gold | Color (B/S/G) | Icon (B/S/G) |
|--------|--------|--------|------|----------------|---------------|
| `pier` | 75m+ | 150m+ | 300m+ | `#CD7F32` / `#C0C0C0` / `#FFD700` | 1 / 2 / 3 stars |
| `cliff` | 100m+ | 200m+ | 400m+ | `#CD7F32` / `#C0C0C0` / `#FFD700` | 1 / 2 / 3 stars |
| `rooftop` | 100m+ | 200m+ | 400m+ | `#CD7F32` / `#C0C0C0` / `#FFD700` | 1 / 2 / 3 stars |
| `mountain` | 150m+ | 300m+ | 600m+ | `#CD7F32` / `#C0C0C0` / `#FFD700` | 1 / 2 / 3 stars |

```javascript
const MEDAL_THRESHOLDS = {
  pier:     { bronze: 75,  silver: 150, gold: 300 },
  cliff:    { bronze: 100, silver: 200, gold: 400 },
  rooftop:  { bronze: 100, silver: 200, gold: 400 },
  mountain: { bronze: 150, silver: 300, gold: 600 }
};

/**
 * Determine the medal earned for a given distance on a specific course.
 * @param {string} course
 * @param {number} distance
 * @returns {string|null} 'gold', 'silver', 'bronze', or null
 */
function getMedalForDistance(course, distance) {
  const thresholds = MEDAL_THRESHOLDS[course];
  if (!thresholds) return null;
  if (distance >= thresholds.gold) return 'gold';
  if (distance >= thresholds.silver) return 'silver';
  if (distance >= thresholds.bronze) return 'bronze';
  return null;
}
```

---

## Debounced Save

The debounce prevents excessive `localStorage.setItem` calls during rapid events
like the ramp tap sequence or fast score updates.

### How It Works

```
Tap 1 (t=0ms)     -> schedule save at t=300ms
Tap 2 (t=80ms)    -> cancel, reschedule at t=380ms
Tap 3 (t=150ms)   -> cancel, reschedule at t=450ms
Tap 4 (t=220ms)   -> cancel, reschedule at t=520ms
... (no more taps)
t=520ms            -> SAVE executes (one write instead of many)
```

### Flush on Page Unload

```javascript
// Ensure all data is written before the page closes
window.addEventListener('beforeunload', () => {
  if (window.gameData) {
    window.gameData.flushAll();
  }
});

// Also flush on visibility change (mobile tab switch)
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden' && window.gameData) {
    window.gameData.flushAll();
  }
});
```

---

## Debug Utilities

Helper functions for testing and debugging persistence (only available in dev mode).

```javascript
// Attach to window for console access
window.flugtag = window.flugtag || {};

window.flugtag.debug = {
  /** Print all stored data */
  dump() {
    const gd = window.gameData;
    if (!gd) { console.log('GameData not initialized'); return; }
    console.table({
      'Craft': gd.getCraft().name + ' (' + gd.getCraft().fuselage.type + ')',
      'Total Flights': gd.getTotalFlights(),
      'Courses Unlocked': gd.getCourseUnlocks().join(', '),
      'Best Pier': gd.getBestDistance('pier') + 'm',
      'Best Cliff': gd.getBestDistance('cliff') + 'm',
      'Best Rooftop': gd.getBestDistance('rooftop') + 'm',
      'Best Mountain': gd.getBestDistance('mountain') + 'm',
      'Medal Pier': gd.getMedals('pier') || 'none',
      'Medal Cliff': gd.getMedals('cliff') || 'none',
      'Sound': gd.getSetting('soundEnabled') ? 'ON' : 'OFF'
    });
  },

  /** Set a fake best distance for testing */
  setBest(course, distance) {
    window.gameData.cache.bestDistance[course] = distance;
    window.gameData._save('best-distance', window.gameData.cache.bestDistance);
    checkCourseUnlocks();
    console.log(`Set best distance for ${course} to ${distance}m`);
  },

  /** Unlock all courses */
  unlockAll() {
    ['pier', 'cliff', 'rooftop', 'mountain'].forEach(c => {
      window.gameData.unlockCourse(c);
    });
    console.log('All courses unlocked');
  },

  /** Give all gold medals */
  goldAll() {
    ['pier', 'cliff', 'rooftop', 'mountain'].forEach(c => {
      window.gameData.cache.medals[c] = 'gold';
    });
    window.gameData._save('medals', window.gameData.cache.medals);
    console.log('All gold medals awarded');
  },

  /** Reset everything */
  reset() {
    window.gameData.resetAll();
    console.log('All data reset');
  },

  /** Show raw localStorage contents */
  raw() {
    const prefix = 'flugtag:';
    for (let i = 0; i < localStorage.length; i++) {
      const key = localStorage.key(i);
      if (key.startsWith(prefix)) {
        console.log(key, JSON.parse(localStorage.getItem(key)));
      }
    }
  }
};
```

### Usage from Browser Console

```
flugtag.debug.dump()           // See all game state
flugtag.debug.setBest('pier', 600)    // Fake a 600m flight
flugtag.debug.unlockAll()      // Unlock all courses
flugtag.debug.goldAll()        // Award all gold medals
flugtag.debug.reset()          // Factory reset
flugtag.debug.raw()            // See raw localStorage entries
```
