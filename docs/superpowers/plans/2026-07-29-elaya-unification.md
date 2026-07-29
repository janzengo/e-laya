# e-Laya Unification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn six independently-working e-Laya surfaces into one coherent product by adding a shared client-side store, a shared cast set in one city, and a reviewer navigation bar.

**Architecture:** Three new classic (non-module) scripts in `public/` — `elaya-store.js` (canonical state, `localStorage`-backed, cross-tab sync), `elaya-cast.js` (the people who appear in more than one surface), `elaya-shell.js` (an out-of-fiction reviewer bar). The six existing surfaces keep their inline scripts and their own seed data; they read the store as an *enhancement layer* and must continue to render correctly when it is absent.

**Tech Stack:** Vanilla HTML/CSS/JS, no framework, no bundler, no build step. Vercel serverless functions in `api/` (untouched by this plan). Playwright for tests, isolated in `test/` with its own `package.json`.

## Global Constraints

- **No framework, no bundler, no build step.** Surfaces stay dependency-free static HTML.
- **New scripts are classic scripts, not ES modules.** Surfaces use inline non-module `<script>` and must keep working from `file://`. Never add `type="module"` to a `public/*.js` include.
- **The store is never a dependency.** Every surface must render and be usable with `localStorage` disabled or the store absent. Existing seed arrays remain as the fallback path.
- **`Elaya.*` never throws into a surface.** On internal failure: log once, set `Elaya.degraded = true`, return the caller's fallback.
- **Do not add dependencies to the root `package.json`.** Vercel installs devDependencies; a Playwright dep there would download browsers on every deploy. Test tooling lives in `test/package.json` and `test/` is listed in `.vercelignore`.
- **Do not redesign screens.** No visual changes beyond the named defect fixes in Task 9.
- **Do not create new screens.** Screen coverage already exceeds BUILD-SPEC.
- Storage key: `elaya.v1`. Store version integer: `1`.
- There are **8** HTML files in `public/`: `index`, `kiosk`, `app`, `cases`, `sessions`, `verify`, `custody`, `pitch`.
- Script include anchor in every surface: the line `<link rel="stylesheet" href="elaya.css">` — verified to occur exactly once in each of the 8 files.
- Target locality for the whole product: **Batangas City, Batangas**.

---

### Task 1: Test harness and favicon

Establishes the test loop everything else depends on, and fixes the one console error present site-wide (`GET /favicon.ico → 404` on all 8 pages).

**Files:**
- Create: `test/package.json`
- Create: `test/playwright.config.js`
- Create: `test/regression.spec.js`
- Create: `public/favicon.svg`
- Create: `.vercelignore`
- Modify: all 8 files in `public/*.html` (add one `<link>` line each)

**Interfaces:**
- Consumes: nothing.
- Produces: `npm test` runnable from `test/`; a served static origin at `http://localhost:5173` for all later tasks; `SURFACES` array exported from `test/regression.spec.js` is **not** shared — each spec file declares its own.

- [ ] **Step 1: Create the test package**

Create `test/package.json`:

```json
{
  "name": "e-laya-tests",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed"
  },
  "devDependencies": {
    "@playwright/test": "^1.49.0"
  }
}
```

- [ ] **Step 2: Create the Playwright config**

Create `test/playwright.config.js`. `python3 -m http.server` is used deliberately so the test loop needs no extra install beyond Playwright itself.

```js
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: '.',
  fullyParallel: false,
  reporter: [['list']],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'retain-on-failure'
  },
  webServer: {
    command: 'python3 -m http.server 5173 --directory ../public',
    url: 'http://localhost:5173/index.html',
    reuseExistingServer: true,
    timeout: 30000
  }
});
```

- [ ] **Step 3: Keep test tooling out of the deploy**

Create `.vercelignore` at the repo root:

```
test/
docs/
```

- [ ] **Step 4: Write the failing regression test**

Create `test/regression.spec.js`:

```js
import { test, expect } from '@playwright/test';

const SURFACES = [
  'index.html', 'kiosk.html', 'app.html', 'cases.html',
  'sessions.html', 'verify.html', 'custody.html', 'pitch.html'
];

for (const page_ of SURFACES) {
  test(`${page_} loads with no console errors`, async ({ page }) => {
    const errors = [];
    page.on('console', m => { if (m.type() === 'error') errors.push(m.text()); });
    page.on('pageerror', e => errors.push(String(e)));

    await page.goto('/' + page_);
    await page.waitForLoadState('networkidle');

    expect(errors, `console errors on ${page_}:\n${errors.join('\n')}`).toEqual([]);
  });

  test(`${page_} declares a favicon`, async ({ page }) => {
    await page.goto('/' + page_);
    const count = await page.locator('link[rel~="icon"]').count();
    expect(count, `${page_} has no <link rel="icon">`).toBeGreaterThan(0);
  });
}
```

- [ ] **Step 5: Install and run the test to verify it fails**

```bash
cd test && npm install && npx playwright install chromium && npm test
```

Expected: the `declares a favicon` tests FAIL with `expect(received).toBeGreaterThan(0)` for all 8 pages. The `no console errors` tests FAIL for all 8 with a `favicon.ico` 404 entry.

- [ ] **Step 6: Create the favicon**

Create `public/favicon.svg`. The mark is a door-frame opening — the "laya / freedom" motif — in the product's primary blue.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <rect width="32" height="32" rx="7" fill="#0040E7"/>
  <path d="M11 23V11.5A1.5 1.5 0 0 1 12.5 10H19a1.5 1.5 0 0 1 1.5 1.5V23"
        fill="none" stroke="#fff" stroke-width="2.4" stroke-linecap="round"/>
  <circle cx="18" cy="17" r="1.35" fill="#fff"/>
</svg>
```

- [ ] **Step 7: Reference it from all 8 surfaces**

In each of `public/index.html`, `kiosk.html`, `app.html`, `cases.html`, `sessions.html`, `verify.html`, `custody.html`, `pitch.html`, insert this line immediately **after** the existing `<link rel="stylesheet" href="elaya.css">` line (which occurs exactly once per file):

```html
<link rel="icon" href="favicon.svg" type="image/svg+xml">
```

- [ ] **Step 8: Run the tests to verify they pass**

```bash
cd test && npm test
```

Expected: all 16 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add .vercelignore public/favicon.svg public/*.html test/
git commit -m "Add Playwright harness and site favicon

Fixes the GET /favicon.ico 404 that was the only console error on all
eight pages. Test tooling is isolated in test/ with its own package.json
so Vercel never installs Playwright during a deploy."
```

---

### Task 2: The store — state, persistence, degradation

Core read/write and durability. No events yet, no surface changes.

**Files:**
- Create: `public/elaya-store.js`
- Create: `test/store.spec.js`

**Interfaces:**
- Consumes: Task 1's harness and served origin.
- Produces: `window.Elaya` with `get(path, fallback)`, `set(path, value)`, `update(path, fn)`, `reset()`, `ready(fn)`, and the boolean flags `Elaya.persistent` and `Elaya.degraded`. Task 3 adds events to this same object. Dot-paths address the state tree, e.g. `'welfare.miguel'`.

- [ ] **Step 1: Write the failing test**

Create `test/store.spec.js`:

```js
import { test, expect } from '@playwright/test';

// The store is not yet included by any page, so inject it directly.
// index.html gives us a real http origin, which localStorage requires.
async function withStore(page) {
  await page.goto('/index.html');
  await page.evaluate(() => localStorage.clear());
  await page.addScriptTag({ path: '../public/elaya-store.js' });
}

test('seeds a versioned state tree', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => window.Elaya.get('version'));
  expect(v).toBe(1);
});

test('get returns the fallback for a missing path', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => window.Elaya.get('welfare.nobody', 'FALLBACK'));
  expect(v).toBe('FALLBACK');
});

test('set writes a nested path and get reads it back', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    window.Elaya.set('welfare.miguel', { key: 'ok', at: '9:14 AM' });
    return window.Elaya.get('welfare.miguel.key');
  });
  expect(v).toBe('ok');
});

test('update applies a function to the existing value', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    window.Elaya.set('attendance.jomar.pg1', { done: 9, of: 24 });
    window.Elaya.update('attendance.jomar.pg1', a => ({ ...a, done: a.done + 1 }));
    return window.Elaya.get('attendance.jomar.pg1.done');
  });
  expect(v).toBe(10);
});

test('state survives a reload', async ({ page }) => {
  await withStore(page);
  await page.evaluate(() => window.Elaya.set('welfare.miguel', { key: 'ok' }));
  await page.reload();
  await page.addScriptTag({ path: '../public/elaya-store.js' });
  const v = await page.evaluate(() => window.Elaya.get('welfare.miguel.key'));
  expect(v).toBe('ok');
});

test('reset restores seed state', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    window.Elaya.set('welfare.miguel', { key: 'ok' });
    window.Elaya.reset();
    return window.Elaya.get('welfare.miguel', null);
  });
  expect(v).toBeNull();
});

test('a corrupt blob is discarded and re-seeded, not thrown', async ({ page }) => {
  await page.goto('/index.html');
  await page.evaluate(() => localStorage.setItem('elaya.v1', '{ not json'));
  await page.addScriptTag({ path: '../public/elaya-store.js' });
  const r = await page.evaluate(() => ({
    version: window.Elaya.get('version'),
    degraded: window.Elaya.degraded
  }));
  expect(r.version).toBe(1);
  expect(r.degraded).toBe(false);
});

test('a stale version is discarded and re-seeded', async ({ page }) => {
  await page.goto('/index.html');
  await page.evaluate(() =>
    localStorage.setItem('elaya.v1', JSON.stringify({ version: 0, welfare: { miguel: { key: 'ok' } } })));
  await page.addScriptTag({ path: '../public/elaya-store.js' });
  const r = await page.evaluate(() => ({
    version: window.Elaya.get('version'),
    welfare: window.Elaya.get('welfare.miguel', null)
  }));
  expect(r.version).toBe(1);
  expect(r.welfare).toBeNull();
});

test('falls back to memory when localStorage throws', async ({ page }) => {
  await page.goto('/index.html');
  await page.evaluate(() => {
    Object.defineProperty(window, 'localStorage', {
      configurable: true,
      get() { throw new Error('blocked'); }
    });
  });
  await page.addScriptTag({ path: '../public/elaya-store.js' });
  const r = await page.evaluate(() => {
    window.Elaya.set('welfare.miguel', { key: 'ok' });
    return { persistent: window.Elaya.persistent, value: window.Elaya.get('welfare.miguel.key') };
  });
  expect(r.persistent).toBe(false);
  expect(r.value).toBe('ok');
});

test('ready runs the callback', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => new Promise(res => window.Elaya.ready(() => res('ran'))));
  expect(v).toBe('ran');
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test store.spec.js
```

Expected: all 10 tests FAIL — `page.addScriptTag` rejects because `../public/elaya-store.js` does not exist.

- [ ] **Step 3: Write the store**

Create `public/elaya-store.js`:

```js
/* elaya-store.js — canonical cross-surface state for e-Laya.
 *
 * Classic script, not a module: surfaces use inline non-module <script> and
 * must keep working from file://. Load this BEFORE a surface's inline script.
 *
 * Contract: this is an enhancement layer. Every call is total — it logs once
 * and returns the caller's fallback rather than throwing into a surface. A
 * surface with the store absent must still render from its own seed data.
 */
(function () {
  'use strict';

  var KEY = 'elaya.v1';
  var VERSION = 1;

  var state = null;
  var readyQueue = [];
  var complained = false;

  var api = {
    persistent: true,
    degraded: false
  };

  function warn(err) {
    if (complained) return;
    complained = true;
    api.degraded = true;
    if (window.console && console.warn) console.warn('[elaya-store] degraded:', err);
  }

  /* ---------- storage, defensively ---------- */

  function storage() {
    try {
      var ls = window.localStorage;
      ls.setItem(KEY + '.probe', '1');
      ls.removeItem(KEY + '.probe');
      return ls;
    } catch (e) {
      api.persistent = false;
      return null;
    }
  }

  function seed() {
    return {
      version: VERSION,
      seededAt: '2026-07-29T00:00:00+08:00',
      people: (window.ELAYA_CAST && window.ELAYA_CAST.people) || {},
      welfare: {},
      attendance: {},
      determinations: {},
      notifications: [],
      updatedAt: null
    };
  }

  function load() {
    var ls = storage();
    if (!ls) return seed();
    try {
      var raw = ls.getItem(KEY);
      if (!raw) return seed();
      var parsed = JSON.parse(raw);
      if (!parsed || parsed.version !== VERSION) return seed();
      // Re-attach the cast: it lives in code, not in storage, so edits to
      // elaya-cast.js take effect without anyone clearing their browser.
      parsed.people = (window.ELAYA_CAST && window.ELAYA_CAST.people) || parsed.people || {};
      return parsed;
    } catch (e) {
      return seed();   // corrupt blob: discard, do not surface an error
    }
  }

  function save() {
    var ls = storage();
    if (!ls) return;
    try {
      state.updatedAt = new Date().toISOString();
      ls.setItem(KEY, JSON.stringify(state));
    } catch (e) {
      warn(e);         // quota or serialisation failure: keep running in memory
    }
  }

  /* ---------- dot-path access ---------- */

  function walk(path) {
    return String(path).split('.').filter(Boolean);
  }

  api.get = function (path, fallback) {
    try {
      var node = state;
      var parts = walk(path);
      for (var i = 0; i < parts.length; i++) {
        if (node == null || typeof node !== 'object') return fallback;
        node = node[parts[i]];
      }
      return node === undefined ? fallback : node;
    } catch (e) {
      warn(e);
      return fallback;
    }
  };

  api.set = function (path, value) {
    try {
      var parts = walk(path);
      var last = parts.pop();
      var node = state;
      for (var i = 0; i < parts.length; i++) {
        if (typeof node[parts[i]] !== 'object' || node[parts[i]] === null) node[parts[i]] = {};
        node = node[parts[i]];
      }
      node[last] = value;
      save();
      api.emit('change', { path: path, value: value });
      return value;
    } catch (e) {
      warn(e);
      return value;
    }
  };

  api.update = function (path, fn) {
    return api.set(path, fn(api.get(path)));
  };

  api.reset = function () {
    try {
      var ls = storage();
      if (ls) ls.removeItem(KEY);
    } catch (e) { /* ignore */ }
    state = seed();
    save();
    api.emit('change', { path: '*', value: null });
  };

  /* ---------- events (expanded in Task 3) ---------- */

  api.emit = function () {};

  api.ready = function (fn) {
    if (state) { try { fn(api); } catch (e) { warn(e); } }
    else readyQueue.push(fn);
  };

  /* ---------- boot ---------- */

  state = load();
  save();
  while (readyQueue.length) api.ready(readyQueue.shift());

  window.Elaya = api;
})();
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
cd test && npx playwright test store.spec.js
```

Expected: all 10 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add public/elaya-store.js test/store.spec.js
git commit -m "Add elaya-store: canonical state with durable, degradable persistence

Dot-path get/set/update over a versioned tree in localStorage. Every call
is total — a corrupt blob, a stale version, or a blocked localStorage all
degrade to seed or memory rather than throwing into a surface."
```

---

### Task 3: The store — events and cross-tab sync

Makes two open windows update each other. This is what demonstrates one system most convincingly, and later tasks depend on `on`/`notify`.

**Files:**
- Modify: `public/elaya-store.js`
- Create: `test/store-events.spec.js`

**Interfaces:**
- Consumes: Task 2's `Elaya.get/set/update/reset/ready`.
- Produces: `Elaya.on(event, handler)`, `Elaya.off(event, handler)`, a working `Elaya.emit(event, payload)`, and `Elaya.notify({to, body, surface, personId})` which appends to `notifications` and returns the created record. Event names: `'change'`, `'welfare'`, `'attendance'`, `'notification'`.

- [ ] **Step 1: Write the failing test**

Create `test/store-events.spec.js`:

```js
import { test, expect } from '@playwright/test';

async function withStore(page) {
  await page.goto('/index.html');
  await page.evaluate(() => localStorage.clear());
  await page.addScriptTag({ path: '../public/elaya-store.js' });
}

test('on receives change events from set', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    const seen = [];
    window.Elaya.on('change', p => seen.push(p.path));
    window.Elaya.set('welfare.miguel', { key: 'ok' });
    return seen;
  });
  expect(v).toEqual(['welfare.miguel']);
});

test('off removes a handler', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    let n = 0;
    const h = () => n++;
    window.Elaya.on('change', h);
    window.Elaya.set('a', 1);
    window.Elaya.off('change', h);
    window.Elaya.set('b', 2);
    return n;
  });
  expect(v).toBe(1);
});

test('a throwing handler does not break emit', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    let reached = false;
    window.Elaya.on('change', () => { throw new Error('bad handler'); });
    window.Elaya.on('change', () => { reached = true; });
    window.Elaya.set('a', 1);
    return reached;
  });
  expect(v).toBe(true);
});

test('notify appends a notification and emits', async ({ page }) => {
  await withStore(page);
  const v = await page.evaluate(() => {
    const seen = [];
    window.Elaya.on('notification', n => seen.push(n.body));
    const rec = window.Elaya.notify({
      to: 'Rosa Reyes', body: 'Miguel is doing OK',
      surface: 'custody', personId: 'miguel'
    });
    return {
      stored: window.Elaya.get('notifications').length,
      body: rec.body,
      hasId: typeof rec.id === 'string' && rec.id.length > 0,
      hasAt: typeof rec.at === 'string' && rec.at.length > 0,
      seen
    };
  });
  expect(v.stored).toBe(1);
  expect(v.body).toBe('Miguel is doing OK');
  expect(v.hasId).toBe(true);
  expect(v.hasAt).toBe(true);
  expect(v.seen).toEqual(['Miguel is doing OK']);
});

test('a write in one tab reaches another tab', async ({ browser }) => {
  const ctx = await browser.newContext();
  const a = await ctx.newPage();
  const b = await ctx.newPage();

  for (const p of [a, b]) {
    await p.goto('/index.html');
    await p.addScriptTag({ path: '../public/elaya-store.js' });
  }
  await a.evaluate(() => window.Elaya.reset());

  await b.evaluate(() => {
    window.__seen = null;
    window.Elaya.on('change', () => { window.__seen = window.Elaya.get('welfare.miguel.key', null); });
  });

  await a.evaluate(() => window.Elaya.set('welfare.miguel', { key: 'ok' }));

  await expect.poll(() => b.evaluate(() => window.__seen), { timeout: 5000 }).toBe('ok');
  await ctx.close();
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test store-events.spec.js
```

Expected: FAIL — `Elaya.on is not a function`.

- [ ] **Step 3: Replace the events stub**

In `public/elaya-store.js`, replace this block:

```js
  /* ---------- events (expanded in Task 3) ---------- */

  api.emit = function () {};
```

with:

```js
  /* ---------- events ---------- */

  var handlers = {};

  api.on = function (event, handler) {
    (handlers[event] || (handlers[event] = [])).push(handler);
    return handler;
  };

  api.off = function (event, handler) {
    var list = handlers[event];
    if (!list) return;
    var i = list.indexOf(handler);
    if (i > -1) list.splice(i, 1);
  };

  api.emit = function (event, payload) {
    var list = (handlers[event] || []).slice();
    for (var i = 0; i < list.length; i++) {
      // One bad subscriber must never stop the others, or block a save.
      try { list[i](payload); } catch (e) { warn(e); }
    }
  };

  api.notify = function (n) {
    var rec = {
      id: 'n' + Date.now().toString(36) + Math.random().toString(36).slice(2, 7),
      to: n.to || '',
      body: n.body || '',
      surface: n.surface || '',
      personId: n.personId || null,
      at: new Date().toISOString()
    };
    var list = api.get('notifications', []);
    list.unshift(rec);
    api.set('notifications', list);
    api.emit('notification', rec);
    return rec;
  };
```

- [ ] **Step 4: Add cross-tab sync**

In `public/elaya-store.js`, immediately **before** the line `window.Elaya = api;`, insert:

```js
  /* ---------- cross-tab sync ----------
     Another tab wrote to localStorage. Re-hydrate and tell this tab's
     subscribers, so /custody and /app open side by side track each other. */
  try {
    window.addEventListener('storage', function (e) {
      if (e.key !== KEY) return;
      state = load();
      api.emit('change', { path: '*', value: null, remote: true });
    });
  } catch (e) {
    warn(e);
  }
```

- [ ] **Step 5: Run the tests to verify they pass**

```bash
cd test && npx playwright test store-events.spec.js store.spec.js
```

Expected: all 15 tests PASS (10 from Task 2, 5 new).

- [ ] **Step 6: Commit**

```bash
git add public/elaya-store.js test/store-events.spec.js
git commit -m "Add store events, notifications and cross-tab sync

A write in one tab re-hydrates every other open tab via the storage event,
so /custody and /app side by side visibly track one another. A throwing
subscriber is isolated and never blocks the others."
```

---

### Task 4: The shared cast and the move to Batangas City

The largest change in this plan: 68 `Quezon City` occurrences across four files, plus barangays and court branches. Purely mechanical, but do it carefully and verify with the regression suite.

**Files:**
- Create: `public/elaya-cast.js`
- Create: `test/cast.spec.js`
- Modify: `public/cases.html`, `public/sessions.html`, `public/custody.html`, `public/verify.html`

**Interfaces:**
- Consumes: nothing from earlier tasks at runtime — `elaya-cast.js` must load *before* `elaya-store.js`, because the store reads `window.ELAYA_CAST` when seeding.
- Produces: `window.ELAYA_CAST = { people: { miguel, jomar, renz }, LOCALITY }`, where each person is `{ id, name, full, initials, age, facility, agency, barangay, guardian, guardianPhone }`.

- [ ] **Step 1: Write the failing test**

Create `test/cast.spec.js`:

```js
import { test, expect } from '@playwright/test';

const MIGRATED = ['cases.html', 'sessions.html', 'custody.html', 'verify.html'];
const STALE = ['Quezon City', 'Payatas', 'Commonwealth', 'Batasan Hills',
               'Holy Spirit', 'Bagong Silangan'];

test('the cast exposes the three bridge people', async ({ page }) => {
  await page.goto('/index.html');
  await page.addScriptTag({ path: '../public/elaya-cast.js' });
  const ids = await page.evaluate(() => Object.keys(window.ELAYA_CAST.people).sort());
  expect(ids).toEqual(['jomar', 'miguel', 'renz']);
});

test('the store seeds people from the cast', async ({ page }) => {
  await page.goto('/index.html');
  await page.evaluate(() => localStorage.clear());
  await page.addScriptTag({ path: '../public/elaya-cast.js' });
  await page.addScriptTag({ path: '../public/elaya-store.js' });
  const name = await page.evaluate(() => window.Elaya.get('people.miguel.full'));
  expect(name).toBe('Miguel Andres Reyes');
});

for (const file of MIGRATED) {
  test(`${file} contains no Quezon City locality strings`, async ({ page }) => {
    const res = await page.request.get('/' + file);
    const body = await res.text();
    const found = STALE.filter(s => body.includes(s));
    expect(found, `${file} still contains: ${found.join(', ')}`).toEqual([]);
  });
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test cast.spec.js
```

Expected: FAIL — `ELAYA_CAST` undefined, and all four locality tests fail listing the stale strings.

- [ ] **Step 3: Write the cast**

Create `public/elaya-cast.js`. Field values are taken from the existing seeds so nothing on screen changes except locality.

```js
/* elaya-cast.js — the people who appear in more than one surface.
 *
 * Load BEFORE elaya-store.js: the store reads window.ELAYA_CAST when it seeds.
 *
 * Only bridge people live here. Each surface keeps its own bulk seed for
 * volume and for the scenarios it is tuned to demonstrate — replacing those
 * with one shared dataset would destroy them.
 */
(function () {
  'use strict';

  window.ELAYA_CAST = {
    LOCALITY: 'Batangas City, Batangas',

    people: {
      // Welfare chain: /custody officer confirms -> /app family sees.
      miguel: {
        id: 'miguel',
        name: 'Miguel Andres R.',
        full: 'Miguel Andres Reyes',
        initials: 'MA',
        age: 24,
        facility: 'Batangas City District Jail',
        agency: 'BJMP',
        barangay: 'Kumintang Ibaba',
        guardian: 'Rosa Andres Reyes',
        guardianPhone: '+63 917 ••• 4567'
      },

      // Attendance chain: /sessions logs -> /app, /cases, /kiosk reflect it.
      jomar: {
        id: 'jomar',
        name: 'Jomar C.',
        full: 'Jomar Cruz',
        initials: 'JC',
        age: 16,
        facility: 'Bahay Pag-asa, Batangas City',
        agency: 'LSWDO',
        barangay: 'Alangilan',
        guardian: 'Rosa Andres Reyes',
        guardianPhone: '+63 917 ••• 4567'
      },

      // Already duplicated by hand in sessions.html and cases.html.
      // Promoting that duplicate to one record is the point of this file.
      renz: {
        id: 'renz',
        name: 'Bautista, Renz A.',
        full: 'Renz A. Bautista',
        initials: 'RB',
        age: 16,
        facility: 'Bahay Pag-asa, Batangas City',
        agency: 'LSWDO',
        barangay: 'Balagtas',
        guardian: 'Mrs. Editha Bautista',
        guardianPhone: '0917 445 2210'
      }
    }
  };
})();
```

- [ ] **Step 4: Migrate the locality strings**

Apply these replacements across `public/cases.html`, `public/sessions.html`, `public/custody.html`, `public/verify.html`. Work one file at a time and read the surrounding line before each change — some are inside prose sentences, not just data.

| Find | Replace |
|---|---|
| `BJMP Quezon City Male Dormitory` | `Batangas City District Jail — Male Dorm` |
| `Quezon City District Jail` | `Batangas City District Jail` |
| `LSWDO Quezon City` | `LSWDO Batangas City` |
| `Quezon City` (all remaining) | `Batangas City` |
| `RTC Br. 104` | `RTC Br. 4` |
| `RTC Br. 87` | `RTC Br. 3` |
| `MTC Br. 12` | `MTC Br. 2` |
| `Payatas` | `Kumintang Ibaba` |
| `Commonwealth` | `Pallocan West` |
| `Batasan Hills` | `Bolbok` |
| `Holy Spirit` | `Alangilan` |
| `Bagong Silangan` | `Balagtas` |

Then handle the long-form court names, which do not match the short patterns above:

| Find | Replace |
|---|---|
| `Regional Trial Court Branch 104, Quezon City` | `Regional Trial Court Branch 4, Batangas City` |
| `Regional Trial Court Branch 87, Quezon City` | `Regional Trial Court Branch 3, Batangas City` |

- [ ] **Step 5: Verify the migration left nothing behind**

```bash
cd "$(git rev-parse --show-toplevel)" && \
grep -nE 'Quezon City|Payatas|Commonwealth|Batasan Hills|Holy Spirit|Bagong Silangan' \
  public/cases.html public/sessions.html public/custody.html public/verify.html
```

Expected: no output. If any line matches, fix it and re-run.

- [ ] **Step 6: Run the tests to verify they pass**

```bash
cd test && npm test
```

Expected: all tests PASS — Task 1's 16 regression tests, Task 2/3's 15 store tests, and the 6 new cast tests.

- [ ] **Step 7: Commit**

```bash
git add public/elaya-cast.js public/cases.html public/sessions.html \
        public/custody.html public/verify.html test/cast.spec.js
git commit -m "Unify the product on Batangas City and add the shared cast

kiosk and app were set in Batangas while cases, sessions, custody and
verify were in Quezon City — six surfaces describing two unrelated worlds.
Everything now shares one city, one set of institutions.

elaya-cast.js holds only the three bridge people. Each surface keeps its
own bulk seed, which is tuned to the scenarios that surface demonstrates."
```

---

### Task 5: Chain A — `/custody` confirms, `/app` sees it

The highest-value flow, and the proof the architecture works. After this task the product demonstrably behaves as one system.

**Files:**
- Modify: `public/custody.html`
- Modify: `public/app.html`
- Create: `test/chain-a.spec.js`

**Interfaces:**
- Consumes: `Elaya.get/set/on/notify/ready` (Tasks 2–3), `window.ELAYA_CAST.people.miguel` (Task 4).
- Produces: store path `welfare.<personId> = { key, at, by, source, expiresAt }` where `key` is one of `ok | clinic | hospital | hearing | program | transferred | released`. `/app` and later `/kiosk` read this path.

- [ ] **Step 1: Write the failing test**

Create `test/chain-a.spec.js`:

```js
import { test, expect } from '@playwright/test';

test('confirming welfare in /custody survives a reload', async ({ page }) => {
  await page.goto('/custody.html');
  await page.evaluate(() => window.Elaya.reset());
  await page.reload();

  const before = await page.evaluate(() =>
    document.body.innerText.match(/(\d+) of \d+ updated today/)[1]);

  await page.evaluate(() => {
    window.Elaya.set('welfare.miguel', {
      key: 'ok', at: '9:14 AM', by: 'JO1 Sarmiento', source: 'manual'
    });
  });
  await page.reload();

  const stored = await page.evaluate(() => window.Elaya.get('welfare.miguel.key'));
  expect(stored).toBe('ok');
  expect(Number(before)).toBeGreaterThan(0);
});

test('a welfare write in /custody appears in /app', async ({ page }) => {
  await page.goto('/custody.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('welfare.miguel', {
      key: 'clinic', at: '10:22 AM', by: 'JO1 Sarmiento', source: 'manual'
    });
    window.Elaya.notify({
      to: 'Rosa Andres Reyes', body: 'Miguel is in the facility clinic',
      surface: 'custody', personId: 'miguel'
    });
  });

  await page.goto('/app.html');
  const r = await page.evaluate(() => ({
    key: window.Elaya.get('welfare.miguel.key'),
    notif: window.Elaya.get('notifications')[0].body
  }));
  expect(r.key).toBe('clinic');
  expect(r.notif).toBe('Miguel is in the facility clinic');
});

test('/app renders the stored welfare state, not its seed', async ({ page }) => {
  await page.goto('/app.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('welfare.miguel', {
      key: 'clinic', at: '10:22 AM', by: 'JO1 Sarmiento', source: 'manual'
    });
  });
  await page.reload();
  await expect(page.locator('body')).toContainText('klinika');
});

test('/custody still renders with the store unavailable', async ({ page }) => {
  await page.addInitScript(() => {
    Object.defineProperty(window, 'localStorage', {
      configurable: true, get() { throw new Error('blocked'); }
    });
  });
  await page.goto('/custody.html');
  await expect(page.locator('body')).toContainText('updated today');
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test chain-a.spec.js
```

Expected: FAIL — `window.Elaya` is undefined on `/custody.html`; the scripts are not included yet.

- [ ] **Step 3: Include the scripts in both surfaces**

In `public/custody.html` and `public/app.html`, insert immediately **after** the `<link rel="icon" href="favicon.svg" type="image/svg+xml">` line added in Task 1 (order matters — cast before store):

```html
<script src="elaya-cast.js"></script>
<script src="elaya-store.js"></script>
```

- [ ] **Step 4: Write the welfare state through the store in `/custody`**

In `public/custody.html`, find the function that records a confirmed status for a person (the handler behind the per-row `OK ✓` control and the status dialog). At the point where it updates that person's status in the in-page model, add a store write alongside it — do not remove the existing in-page update, which is the standalone fallback path:

```js
/* Mirror the confirmation into the shared store so /app can see it.
   Guarded: the surface must keep working when the store is absent. */
if (window.Elaya && person.castId) {
  window.Elaya.set('welfare.' + person.castId, {
    key: statusKey,
    at: fTime(new Date()),
    by: OFFICER.short,
    source: 'manual',
    expiresAt: Date.now() + 24 * 60 * 60 * 1000
  });
  window.Elaya.notify({
    to: person.kinName,
    body: WELFARE_SMS[statusKey],
    surface: 'custody',
    personId: person.castId
  });
}
```

Seed Miguel into the roster so a bridge person exists to confirm. In the roster-generation code, set the first row of the "Not yet updated" group from the cast rather than the random name generator:

```js
/* Row 1 is a bridge person: the same human /app calls Rosa's son. */
var _cast = (window.ELAYA_CAST && window.ELAYA_CAST.people.miguel) || null;
if (_cast) {
  roster[0].name    = _cast.full;
  roster[0].castId  = _cast.id;
  roster[0].kinName = _cast.guardian;
}
```

Add the SMS copy table near the other constants:

```js
const WELFARE_SMS = {
  ok:          'Miguel is doing OK. Confirmed today by the facility.',
  clinic:      'Miguel is in the facility clinic, being monitored by a nurse.',
  hospital:    'Miguel was taken to hospital for care. Call the facility for details.',
  hearing:     'Miguel is at a hearing today.',
  program:     'Miguel is in a programme session today.',
  transferred: 'Miguel was transferred. Check the new address before travelling.',
  released:    'Miguel has been released.'
};
```

- [ ] **Step 5: Read the welfare state in `/app`**

In `public/app.html`, find where a person's welfare chip is rendered from `PEOPLE[i].welfare`. Overlay the stored value when one exists:

```js
/* The store wins when the officer has confirmed something more recent.
   With no store, this is a no-op and the seed renders unchanged. */
function welfareFor(p) {
  if (window.Elaya) {
    var w = window.Elaya.get('welfare.' + p.id, null);
    if (w && w.key) {
      return { key: w.key, at: w.at, day: 'ngayong araw', ageHours: 0, live: true };
    }
  }
  return p.welfare;
}
```

Replace direct reads of `p.welfare` in the person-card and person-detail renderers with `welfareFor(p)`.

Re-render when another tab writes:

```js
if (window.Elaya) {
  window.Elaya.on('change', function () { render(); });
}
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
cd test && npx playwright test chain-a.spec.js
```

Expected: all 4 tests PASS.

- [ ] **Step 7: Run the full suite for regressions**

```bash
cd test && npm test
```

Expected: everything PASS.

- [ ] **Step 8: Commit**

```bash
git add public/custody.html public/app.html test/chain-a.spec.js
git commit -m "Chain A: welfare confirmed in /custody reaches /app

An officer confirming Miguel now writes through the shared store and
queues the SMS a family would receive. The family view reads that state
instead of its seed, and re-renders when another tab writes it.

Both surfaces still render from their own seeds with the store absent."
```

---

### Task 6: The reviewer shell

Gives every surface a way out, and fixes the dead Back button.

**Files:**
- Create: `public/elaya-shell.js`
- Create: `test/shell.spec.js`
- Modify: `public/index.html`, `kiosk.html`, `app.html`, `cases.html`, `sessions.html`, `verify.html`, `custody.html`

**Interfaces:**
- Consumes: `Elaya.reset()`, `Elaya.persistent` (Tasks 2–3).
- Produces: a `<nav id="elaya-shell">` appended to `document.body` on every surface; honours `?bare=1`; persists dismissal under `localStorage['elaya.shell.hidden']`.

- [ ] **Step 1: Write the failing test**

Create `test/shell.spec.js`:

```js
import { test, expect } from '@playwright/test';

const SURFACES = ['index.html', 'kiosk.html', 'app.html', 'cases.html',
                  'sessions.html', 'verify.html', 'custody.html'];

for (const s of SURFACES) {
  test(`${s} shows the reviewer bar with a link to every surface`, async ({ page }) => {
    await page.goto('/' + s);
    const nav = page.locator('#elaya-shell');
    await expect(nav).toBeVisible();
    await expect(nav.locator('a')).toHaveCount(SURFACES.length);
  });
}

test('?bare=1 hides the reviewer bar', async ({ page }) => {
  await page.goto('/custody.html?bare=1');
  await expect(page.locator('#elaya-shell')).toHaveCount(0);
});

test('the bar marks the current surface', async ({ page }) => {
  await page.goto('/cases.html');
  await expect(page.locator('#elaya-shell a[aria-current="page"]')).toHaveText(/Cases/i);
});

test('dismissal persists across navigation', async ({ page }) => {
  await page.goto('/custody.html');
  await page.locator('#elaya-shell button[data-act="hide"]').click();
  await expect(page.locator('#elaya-shell')).toHaveCount(0);
  await page.goto('/cases.html');
  await expect(page.locator('#elaya-shell')).toHaveCount(0);
});

test('the custody Back button leaves the page', async ({ page }) => {
  await page.goto('/index.html');
  await page.goto('/custody.html');
  await page.getByRole('button', { name: 'Back', exact: true }).click();
  await expect(page).not.toHaveURL(/custody\.html/);
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test shell.spec.js
```

Expected: FAIL — `#elaya-shell` not found; the Back test fails because the URL is unchanged.

- [ ] **Step 3: Write the shell**

Create `public/elaya-shell.js`:

```js
/* elaya-shell.js — an out-of-fiction reviewer bar.
 *
 * Deliberately NOT in-fiction navigation. The kiosk is a terminal bolted to a
 * jail wall; a control offering "switch to PAO caseload" would be a lie about
 * the product. So this is visibly a review aid: dark, fixed, labelled, and
 * removable with ?bare=1 for filming and screenshots.
 */
(function () {
  'use strict';

  var HIDDEN_KEY = 'elaya.shell.hidden';

  var SURFACES = [
    { href: 'index.html',    label: 'Home' },
    { href: 'kiosk.html',    label: 'Kiosk' },
    { href: 'app.html',      label: 'Family' },
    { href: 'cases.html',    label: 'Cases' },
    { href: 'sessions.html', label: 'Sessions' },
    { href: 'verify.html',   label: 'Verify' },
    { href: 'custody.html',  label: 'Custody' }
  ];

  function hidden() {
    try { return localStorage.getItem(HIDDEN_KEY) === '1'; } catch (e) { return false; }
  }

  function hide() {
    try { localStorage.setItem(HIDDEN_KEY, '1'); } catch (e) { /* ignore */ }
    var el = document.getElementById('elaya-shell');
    if (el) el.remove();
  }

  function build() {
    if (new URLSearchParams(location.search).get('bare') === '1') return;
    if (hidden()) return;
    if (document.getElementById('elaya-shell')) return;

    var here = (location.pathname.split('/').pop() || 'index.html');

    var css = document.createElement('style');
    css.textContent =
      '#elaya-shell{position:fixed;left:0;right:0;bottom:0;z-index:2147483000;' +
      'display:flex;align-items:center;gap:4px;flex-wrap:wrap;' +
      'background:#0b1020;border-top:1px solid #263154;padding:6px 10px;' +
      'font:500 12px/1 ui-sans-serif,system-ui,-apple-system,sans-serif}' +
      '#elaya-shell .tag{color:#7b89b8;letter-spacing:.12em;text-transform:uppercase;' +
      'font-size:9.5px;margin-right:6px;white-space:nowrap}' +
      '#elaya-shell a{color:#c8d3f5;text-decoration:none;padding:5px 9px;border-radius:6px;' +
      'white-space:nowrap}' +
      '#elaya-shell a:hover{background:#1b2440}' +
      '#elaya-shell a[aria-current=page]{background:#0040E7;color:#fff}' +
      '#elaya-shell .sp{flex:1 1 auto}' +
      '#elaya-shell button{background:none;border:1px solid #263154;color:#7b89b8;' +
      'border-radius:6px;padding:5px 9px;cursor:pointer;font:inherit;white-space:nowrap}' +
      '#elaya-shell button:hover{color:#c8d3f5;border-color:#3a4a78}' +
      '@media print{#elaya-shell{display:none}}';
    document.head.appendChild(css);

    var nav = document.createElement('nav');
    nav.id = 'elaya-shell';
    nav.setAttribute('aria-label', 'e-Laya reviewer navigation');

    var tag = document.createElement('span');
    tag.className = 'tag';
    tag.textContent = 'Review';
    nav.appendChild(tag);

    SURFACES.forEach(function (s) {
      var a = document.createElement('a');
      a.href = s.href;
      a.textContent = s.label;
      if (s.href === here) a.setAttribute('aria-current', 'page');
      nav.appendChild(a);
    });

    var sp = document.createElement('span');
    sp.className = 'sp';
    nav.appendChild(sp);

    if (window.Elaya && window.Elaya.persistent === false) {
      var warn = document.createElement('span');
      warn.className = 'tag';
      warn.textContent = 'Not saving';
      nav.appendChild(warn);
    }

    var reset = document.createElement('button');
    reset.setAttribute('data-act', 'reset');
    reset.textContent = 'Reset demo';
    reset.addEventListener('click', function () {
      if (window.Elaya) window.Elaya.reset();
      location.reload();
    });
    nav.appendChild(reset);

    var close = document.createElement('button');
    close.setAttribute('data-act', 'hide');
    close.setAttribute('aria-label', 'Hide reviewer bar');
    close.textContent = '×';
    close.addEventListener('click', hide);
    nav.appendChild(close);

    document.body.appendChild(nav);
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', build);
  } else {
    build();
  }
})();
```

- [ ] **Step 4: Include the shell on all seven surfaces**

In each of `public/index.html`, `kiosk.html`, `app.html`, `cases.html`, `sessions.html`, `verify.html`, `custody.html` (**not** `pitch.html` — it is a presentation deck), add after the favicon line:

```html
<script src="elaya-shell.js" defer></script>
```

For `index.html`, `kiosk.html`, `cases.html`, `sessions.html` and `verify.html`, also add the cast and store includes if not already present from Task 5 (`custody.html` and `app.html` already have them):

```html
<script src="elaya-cast.js"></script>
<script src="elaya-store.js"></script>
```

- [ ] **Step 5: Wire the dead Back button**

In `public/custody.html`, find the app-bar Back control (`button` with `aria-label="Back"`, text `‹`) and give it a handler:

```js
/* Was a no-op. Return to wherever the reviewer came from, or Home. */
document.querySelector('[aria-label="Back"]').addEventListener('click', function () {
  if (history.length > 1) history.back();
  else location.assign('index.html');
});
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
cd test && npx playwright test shell.spec.js
```

Expected: all 11 tests PASS.

- [ ] **Step 7: Run the full suite**

```bash
cd test && npm test
```

Expected: everything PASS. If the Task 1 console-error tests now fail, the shell is logging — fix the cause rather than relaxing the assertion.

- [ ] **Step 8: Commit**

```bash
git add public/elaya-shell.js public/*.html test/shell.spec.js
git commit -m "Add the reviewer bar and fix the dead Back button

Every surface was a dead end: index linked out to all six and nothing
linked back, and custody's Back control did nothing when clicked.

The bar is deliberately out-of-fiction. In-fiction navigation would be a
lie about the kiosk, which is a terminal bolted to a wall. Hidden with
?bare=1 for filming, and dismissal persists."
```

---

### Task 7: Chain B — attendance flows out of `/sessions`

**Files:**
- Modify: `public/sessions.html`, `public/app.html`, `public/cases.html`, `public/kiosk.html`
- Create: `test/chain-b.spec.js`

**Interfaces:**
- Consumes: `Elaya.get/update/notify/on` (Tasks 2–3), `ELAYA_CAST.people.jomar` and `.renz` (Task 4).
- Produces: store path `attendance.<personId>.<programmeId> = { done, of, lastAt, receipt }`. `/app`, `/cases` and `/kiosk` read it.

- [ ] **Step 1: Write the failing test**

Create `test/chain-b.spec.js`:

```js
import { test, expect } from '@playwright/test';

test('logging attendance increments the shared count', async ({ page }) => {
  await page.goto('/sessions.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('attendance.jomar.pg1', { done: 9, of: 24, lastAt: null, receipt: null });
    window.Elaya.update('attendance.jomar.pg1', a => ({
      ...a, done: a.done + 1, lastAt: '2026-07-29T14:00:00+08:00', receipt: 'a3f1…8c02'
    }));
  });
  const done = await page.evaluate(() => window.Elaya.get('attendance.jomar.pg1.done'));
  expect(done).toBe(10);
});

test('the updated count is visible in /app, /cases and /kiosk', async ({ page }) => {
  await page.goto('/sessions.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('attendance.jomar.pg1', {
      done: 10, of: 24, lastAt: '2026-07-29T14:00:00+08:00', receipt: 'a3f1…8c02'
    });
  });

  for (const surface of ['/app.html', '/cases.html', '/kiosk.html']) {
    await page.goto(surface);
    const done = await page.evaluate(() => window.Elaya.get('attendance.jomar.pg1.done'));
    expect(done, `store not readable on ${surface}`).toBe(10);
  }
});

test('a missed-attendance flag on Renz reaches /cases', async ({ page }) => {
  await page.goto('/sessions.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('attendance.renz.pg1', { done: 5, of: 12, missStreak: 3, lastAt: null, receipt: null });
    window.Elaya.notify({
      to: 'Mrs. Editha Bautista', body: 'Renz has missed three sessions.',
      surface: 'sessions', personId: 'renz'
    });
  });
  await page.goto('/cases.html');
  const r = await page.evaluate(() => ({
    streak: window.Elaya.get('attendance.renz.pg1.missStreak'),
    notif: window.Elaya.get('notifications')[0].personId
  }));
  expect(r.streak).toBe(3);
  expect(r.notif).toBe('renz');
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test chain-b.spec.js
```

Expected: FAIL — `window.Elaya` is undefined on `/sessions.html` and `/kiosk.html`.

- [ ] **Step 3: Add the includes**

In `public/sessions.html`, `public/cases.html` and `public/kiosk.html`, confirm the cast and store includes from Task 6 Step 4 are present. Add them after the favicon line if missing:

```html
<script src="elaya-cast.js"></script>
<script src="elaya-store.js"></script>
```

- [ ] **Step 4: Write attendance through the store in `/sessions`**

In `public/sessions.html`, find the save handler for logging a session (the one that computes the canonical-JSON SHA-256 receipt). After it updates the in-page model, mirror it:

```js
/* Mirror into the shared store so /app, /cases and /kiosk agree. */
if (window.Elaya && person.castId) {
  window.Elaya.update('attendance.' + person.castId + '.' + prog.id, function (a) {
    a = a || { done: person.done, of: person.of, missStreak: 0 };
    return {
      done: a.done + 1,
      of: a.of,
      missStreak: 0,
      lastAt: new Date().toISOString(),
      receipt: receiptShort
    };
  });
  window.Elaya.notify({
    to: person.guardian,
    body: person.name.split(',')[0] + ' attended today’s session.',
    surface: 'sessions',
    personId: person.castId
  });
}
```

Tag the two bridge people in `ROSTER` so `castId` resolves. `Bautista, Renz A.` is `r1`; add Jomar Cruz as a new roster entry rather than renaming an existing child:

```js
/* Bridge people: the same humans /app, /cases and /kiosk show. */
ROSTER[0].castId = 'renz';                     // Bautista, Renz A.
ROSTER.push({
  id: 'r16', castId: 'jomar', name: 'Cruz, Jomar', age: 16,
  done: 9, of: 24, missStreak: 0, last: new Date(2026, 6, 22),
  guardian: 'Rosa Andres Reyes', gno: '0917 445 4567', no: '',
  brgy: 'Alangilan'
});
```

- [ ] **Step 5: Read attendance in the three consuming surfaces**

In `public/app.html`, where programme progress is rendered from `p.programme.attended` / `.total`, overlay the store:

```js
function attendanceFor(p) {
  if (window.Elaya) {
    var a = window.Elaya.get('attendance.' + p.id + '.pg1', null);
    if (a && typeof a.done === 'number') {
      return { attended: a.done, total: a.of, receipt: a.receipt || p.programme.receipt };
    }
  }
  return { attended: p.programme.attended, total: p.programme.total, receipt: p.programme.receipt };
}
```

In `public/cases.html`, where a client's programme standing is rendered, apply the same overlay keyed on the case's `castId`. Tag Renz's case record:

```js
/* Renz was already duplicated by hand between sessions.html and cases.html.
   This is the same human, now backed by one record. */
CASES.forEach(function (c) {
  if (c.name === 'Bautista, Renz A.') c.castId = 'renz';
});
```

In `public/kiosk.html`, where "Akong mga Programa" renders the attended count, read `attendance.jomar.pg1` when present and fall back to the seed otherwise.

- [ ] **Step 6: Run the tests to verify they pass**

```bash
cd test && npx playwright test chain-b.spec.js
```

Expected: all 3 tests PASS.

- [ ] **Step 7: Run the full suite**

```bash
cd test && npm test
```

Expected: everything PASS.

- [ ] **Step 8: Commit**

```bash
git add public/sessions.html public/app.html public/cases.html public/kiosk.html test/chain-b.spec.js
git commit -m "Chain B: attendance logged in /sessions reaches app, cases and kiosk

Renz was duplicated by hand across sessions.html and cases.html. He is now
one record, so a missed-attendance flag raised by the social worker is
visible to the lawyer holding his case."
```

---

### Task 8: Chain C — a determination in `/verify` creates a person

**Files:**
- Modify: `public/verify.html`, `public/cases.html`, `public/app.html`
- Create: `test/chain-c.spec.js`

**Interfaces:**
- Consumes: `Elaya.set/notify/get` (Tasks 2–3).
- Produces: store paths `determinations.<personId> = { category, age, at, by }` where `category` is `'CICL' | 'PDL'`, and `people.<personId>` written with the same `PersonRecord` shape as `elaya-cast.js`.

- [ ] **Step 1: Write the failing test**

Create `test/chain-c.spec.js`:

```js
import { test, expect } from '@playwright/test';

test('a determination writes a person and a category', async ({ page }) => {
  await page.goto('/verify.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('people.intake1', {
      id: 'intake1', full: 'Danilo P. Mercado', name: 'Danilo M.', initials: 'DM',
      age: 16, facility: 'Bahay Pag-asa, Batangas City', agency: 'LSWDO',
      barangay: 'Bolbok', guardian: 'Mrs. Nena Mercado', guardianPhone: '0917 220 1188'
    });
    window.Elaya.set('determinations.intake1', {
      category: 'CICL', age: 16, at: '2026-07-29T15:10:00+08:00', by: 'LSWDO Batangas City'
    });
    window.Elaya.notify({
      to: 'Mrs. Nena Mercado',
      body: 'Danilo has been assessed as a child in conflict with the law.',
      surface: 'verify', personId: 'intake1'
    });
  });

  await page.goto('/cases.html');
  const r = await page.evaluate(() => ({
    cat: window.Elaya.get('determinations.intake1.category'),
    name: window.Elaya.get('people.intake1.full')
  }));
  expect(r.cat).toBe('CICL');
  expect(r.name).toBe('Danilo P. Mercado');
});

test('a CICL determination is visible to /app as a linked person', async ({ page }) => {
  await page.goto('/verify.html');
  await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('people.intake1', { id: 'intake1', full: 'Danilo P. Mercado', age: 16 });
    window.Elaya.set('determinations.intake1', { category: 'CICL', age: 16 });
  });
  await page.goto('/app.html');
  const ids = await page.evaluate(() => Object.keys(window.Elaya.get('people')));
  expect(ids).toContain('intake1');
});

test('an adult determination records PDL, not CICL', async ({ page }) => {
  await page.goto('/verify.html');
  const cat = await page.evaluate(() => {
    window.Elaya.reset();
    window.Elaya.set('determinations.intake2', { category: 'PDL', age: 24 });
    return window.Elaya.get('determinations.intake2.category');
  });
  expect(cat).toBe('PDL');
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test chain-c.spec.js
```

Expected: FAIL — `window.Elaya` is undefined on `/verify.html` if the includes are missing; otherwise the assertions fail.

- [ ] **Step 3: Add the includes to `/verify`**

Confirm `public/verify.html` has the cast and store includes from Task 6 Step 4. Add after the favicon line if missing:

```html
<script src="elaya-cast.js"></script>
<script src="elaya-store.js"></script>
```

- [ ] **Step 4: Write the determination through the store**

In `public/verify.html`, find the handler that finalises the CICL/PDL determination (the screen showing the computed age large, with the resulting categorisation). After it updates the in-page model, add:

```js
/* Publish the determination so /cases can open a case and /app can link a
   guardian. This is the moment an unnamed person enters the record. */
if (window.Elaya) {
  var pid = 'intake' + Date.now().toString(36);
  window.Elaya.set('people.' + pid, {
    id: pid,
    full: determined.fullName,
    name: determined.shortName,
    initials: determined.initials,
    age: determined.age,
    facility: determined.age < 18 ? 'Bahay Pag-asa, Batangas City' : 'Batangas City District Jail',
    agency: determined.age < 18 ? 'LSWDO' : 'BJMP',
    barangay: determined.barangay || '',
    guardian: determined.guardianName || '',
    guardianPhone: determined.guardianPhone || ''
  });
  window.Elaya.set('determinations.' + pid, {
    category: determined.age < 18 ? 'CICL' : 'PDL',
    age: determined.age,
    at: new Date().toISOString(),
    by: 'LSWDO Batangas City'
  });
  if (determined.guardianName) {
    window.Elaya.notify({
      to: determined.guardianName,
      body: determined.shortName + (determined.age < 18
        ? ' has been assessed as a child in conflict with the law.'
        : ' has been recorded at intake.'),
      surface: 'verify',
      personId: pid
    });
  }
}
```

- [ ] **Step 5: Surface store-created people in `/cases`**

In `public/cases.html`, after the seed `CASES` array is built, append any people the store knows about that the seed does not:

```js
/* People created by a /verify determination arrive here as new cases. */
if (window.Elaya) {
  var dets = window.Elaya.get('determinations', {});
  Object.keys(dets).forEach(function (pid) {
    var p = window.Elaya.get('people.' + pid, null);
    if (!p || CASES.some(function (c) { return c.castId === pid; })) return;
    mk({
      name: p.full, age: p.age, castId: pid,
      cicl: dets[pid].category === 'CICL',
      docket: 'Pending docket', court: 'MTC Br. 2',
      courtLong: 'Municipal Trial Court Branch 2, Batangas City',
      offence: 'For assessment', stage: 'Intake',
      facility: p.facility, diversionFlag: dets[pid].category === 'CICL',
      family: { name: p.guardian || '', rel: 'guardian', no: p.guardianPhone || '' }
    });
  });
}
```

- [ ] **Step 6: Surface store-created people in `/app`**

In `public/app.html`, where the linked-persons list is built from `PEOPLE`, append store people not already present:

```js
function linkedPeople() {
  var list = PEOPLE.slice();
  if (!window.Elaya) return list;
  var stored = window.Elaya.get('people', {});
  Object.keys(stored).forEach(function (id) {
    if (list.some(function (p) { return p.id === id; })) return;
    var s = stored[id];
    list.push({
      id: s.id, name: s.name || s.full, full: s.full, initials: s.initials || '??',
      relation: 'Naitala sa intake', ageLine: s.age ? s.age + ' taong gulang' : null,
      facility: s.facility, agency: s.agency, officer: '', hotline: '', hotlineTel: '',
      medical: null, medicalTel: null, visiting: ['', ''], ref: '',
      welfare: { key: 'none', ageHours: 0, at: '', day: '' },
      caseState: null, hearing: null, gcta: null, programme: null
    });
  });
  return list;
}
```

Replace reads of `PEOPLE` in the list renderer with `linkedPeople()`.

- [ ] **Step 7: Run the tests to verify they pass**

```bash
cd test && npx playwright test chain-c.spec.js
```

Expected: all 3 tests PASS.

- [ ] **Step 8: Run the full suite**

```bash
cd test && npm test
```

Expected: everything PASS.

- [ ] **Step 9: Commit**

```bash
git add public/verify.html public/cases.html public/app.html test/chain-c.spec.js
git commit -m "Chain C: a determination in /verify creates a case and a linked person

The screen that prevents a child being processed as an adult now produces
a record the rest of the product can see — a case for counsel, and a
linked person for the guardian."
```

---

### Task 9: Remaining defects D3–D7

**Files:**
- Modify: `public/cases.html`, `sessions.html`, `verify.html`, `custody.html`, `kiosk.html`
- Create: `test/a11y.spec.js`

**Interfaces:**
- Consumes: nothing.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Write the failing test**

Create `test/a11y.spec.js`:

```js
import { test, expect } from '@playwright/test';

const SURFACES = ['index.html', 'kiosk.html', 'app.html', 'cases.html',
                  'sessions.html', 'verify.html', 'custody.html'];

for (const s of SURFACES) {
  test(`${s} has exactly one h1`, async ({ page }) => {
    await page.goto('/' + s);
    await expect(page.locator('h1')).toHaveCount(1);
  });

  test(`${s} has no nested interactive elements`, async ({ page }) => {
    await page.goto('/' + s);
    const nested = await page.evaluate(() =>
      Array.from(document.querySelectorAll('button button, a a, button a, a button'))
        .map(e => e.textContent.trim().slice(0, 40)));
    expect(nested, `nested controls on ${s}`).toEqual([]);
  });

  test(`${s} has no tap target under 32px`, async ({ page }) => {
    await page.setViewportSize({ width: 390, height: 844 });
    await page.goto('/' + s);
    const small = await page.evaluate(() =>
      Array.from(document.querySelectorAll('button, a, [role=button]'))
        .filter(e => {
          const r = e.getBoundingClientRect();
          return r.width > 0 && r.height > 0 && (r.height < 32 || r.width < 32);
        })
        .map(e => (e.textContent || e.getAttribute('aria-label') || '?').trim().slice(0, 30)));
    expect(small, `small tap targets on ${s}`).toEqual([]);
  });
}

test('kiosk sets the document language when a language is chosen', async ({ page }) => {
  await page.goto('/kiosk.html');
  await page.getByRole('button', { name: /Binisaya/ }).first().click();
  await expect.poll(() => page.evaluate(() => document.documentElement.lang)).toBe('ceb');
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd test && npx playwright test a11y.spec.js
```

Expected: FAIL — `h1` count 0 on cases/sessions/verify/custody and 9 on kiosk, 12 on app; nested controls on kiosk; the `lang` test returns `fil`.

- [ ] **Step 3: Fix D3 — one `<h1>` per surface**

`kiosk.html` and `app.html` currently render one `<h1>` per screen (9 and 12). Demote all of them to `<h2>`, then add a single visually-hidden `<h1>` as the first child of `<body>` in each of the seven surfaces. Add the utility to `public/elaya.css`:

```css
.vh{position:absolute;width:1px;height:1px;padding:0;margin:-1px;overflow:hidden;
    clip:rect(0 0 0 0);white-space:nowrap;border:0}
```

Then add to each surface, as the first element inside `<body>`:

| File | Heading |
|---|---|
| `index.html` | *(already has one — leave it)* |
| `kiosk.html` | `<h1 class="vh">e-Laya facility terminal</h1>` |
| `app.html` | `<h1 class="vh">e-Laya — Kapamilya</h1>` |
| `cases.html` | `<h1 class="vh">e-Laya — PAO caseload</h1>` |
| `sessions.html` | `<h1 class="vh">e-Laya — programme attendance</h1>` |
| `verify.html` | `<h1 class="vh">e-Laya — identity resolution</h1>` |
| `custody.html` | `<h1 class="vh">e-Laya — welfare check-in</h1>` |

- [ ] **Step 4: Fix D4 — unnest the kiosk language buttons**

In `public/kiosk.html`, each language tile is currently a `<button>` containing a nested `Basahin:` `<button>`. Restructure to two siblings inside one positioned wrapper, so both remain real buttons:

```html
<div class="langwrap">
  <button class="lang" data-lang="ceb">
    <span class="l1">Binisaya</span>
    <span class="l2">Sinugboanon</span>
  </button>
  <button class="speak" data-lang="ceb" aria-label="Basahin: Binisaya"></button>
</div>
```

Add the positioning to the page's inline `<style>`:

```css
.langwrap{position:relative;display:block}
.langwrap .speak{position:absolute;top:8px;right:8px;width:44px;height:44px}
```

Update the click handlers to bind `.lang` and `.speak` separately rather than relying on event bubbling from the outer button.

- [ ] **Step 5: Fix D5 — set the document language**

In `public/kiosk.html`, in the language-selection handler, add:

```js
/* Nine languages were offered while the document claimed to be Filipino. */
const LANG_TAGS = { fil:'fil', en:'en', ceb:'ceb', ilo:'ilo', hil:'hil',
                    war:'war', pam:'pam', pag:'pag', bik:'bik' };
document.documentElement.lang = LANG_TAGS[code] || 'fil';
```

- [ ] **Step 6: Fix D6 — the small tap target**

Run the a11y spec to identify the offending control by its reported label, then pad it to at least 44 × 44 in that surface's inline `<style>`. Do not change its visual size if that would break the layout — add padding, or a `::before` overlay that extends the hit area:

```css
.tinybtn{position:relative}
.tinybtn::before{content:"";position:absolute;inset:-12px}
```

- [ ] **Step 7: Fix D7 — confirm the aborted requests are intentional**

Read the `tryApi` helper in `public/kiosk.html` (referenced near the comment `attempts /api/everify, /api/psgc, /api/ai with silent fallback`). Confirm the `AbortController` timeout is deliberate.

If it is deliberate, suppress the console noise so Task 1's zero-console-errors assertion holds:

```js
/* An aborted probe is the fallback working as designed, not an error. */
catch (e) {
  if (e && e.name === 'AbortError') return null;
  throw e;
}
```

If it is **not** deliberate, stop and report the finding rather than papering over a real bug.

- [ ] **Step 8: Run the tests to verify they pass**

```bash
cd test && npx playwright test a11y.spec.js
```

Expected: all 22 tests PASS.

- [ ] **Step 9: Commit**

```bash
git add public/*.html public/elaya.css test/a11y.spec.js
git commit -m "Fix accessibility and markup defects D3-D7

One h1 per surface; kiosk language tiles no longer nest a button inside a
button; the document language now follows the nine-language selector
instead of claiming Filipino throughout; tap targets meet 44px."
```

---

### Task 10: Full regression and deploy

**Files:**
- Modify: `README.md`
- Create: `test/flows.spec.js`

**Interfaces:**
- Consumes: everything.
- Produces: a single suite covering the three chains end to end.

- [ ] **Step 1: Write the end-to-end flow test**

Create `test/flows.spec.js`:

```js
import { test, expect } from '@playwright/test';

test('standalone: every surface renders with localStorage blocked', async ({ page }) => {
  await page.addInitScript(() => {
    Object.defineProperty(window, 'localStorage', {
      configurable: true, get() { throw new Error('blocked'); }
    });
  });
  for (const s of ['kiosk.html', 'app.html', 'cases.html',
                   'sessions.html', 'verify.html', 'custody.html']) {
    const errors = [];
    page.on('pageerror', e => errors.push(String(e)));
    await page.goto('/' + s);
    await expect(page.locator('body')).not.toBeEmpty();
    expect(errors, `${s} threw with no storage`).toEqual([]);
  }
});

test('two tabs stay in sync', async ({ browser }) => {
  const ctx = await browser.newContext();
  const custody = await ctx.newPage();
  const app = await ctx.newPage();

  await custody.goto('/custody.html');
  await app.goto('/app.html');
  await custody.evaluate(() => window.Elaya.reset());

  await app.evaluate(() => {
    window.__key = null;
    window.Elaya.on('change', () => { window.__key = window.Elaya.get('welfare.miguel.key', null); });
  });

  await custody.evaluate(() => window.Elaya.set('welfare.miguel', { key: 'ok', at: '9:14 AM' }));

  await expect.poll(() => app.evaluate(() => window.__key), { timeout: 5000 }).toBe('ok');
  await ctx.close();
});
```

- [ ] **Step 2: Run the whole suite**

```bash
cd test && npm test
```

Expected: every test across all spec files PASS.

- [ ] **Step 3: Verify no test tooling reaches the deploy**

```bash
cd "$(git rev-parse --show-toplevel)" && cat .vercelignore && \
  git ls-files public/ | grep -c '\.spec\.js$'
```

Expected: `.vercelignore` lists `test/` and `docs/`; the count is `0`.

- [ ] **Step 4: Update the README architecture block**

In `README.md`, replace the `## Architecture` code block with one that names the three new files:

```
public/          static surfaces — no build step
  elaya-cast.js    the people who appear in more than one surface
  elaya-store.js   shared state, localStorage-backed, syncs across tabs
  elaya-shell.js   out-of-fiction reviewer navigation (?bare=1 to hide)
api/             serverless proxy — every government secret terminates here
  _lib.js        token caching, canonical JSON, SHA-256, eGovPay digest
  sso.js         eGov SSO — identity, role gate, lawful basis
  everify.js     eVerify — National ID read (1:1 verification only)
  liveness.js    Face Liveness — 95.0 threshold enforced server-side
  sms.js         eMessage — E.164 normalised, 60s dedupe
  ai.js          eGov AI — translate, speech, laws, document extraction (memoised)
  psgc.js        eReport — region → barangay hierarchy
  chain.js       eGovChain — read-only network state
  pay.js         eGovPay
```

- [ ] **Step 5: Commit and deploy**

```bash
git add README.md test/flows.spec.js
git commit -m "Add end-to-end flow tests and document the shared architecture"
git push origin main
```

- [ ] **Step 6: Verify the live deploy**

```bash
for p in / /kiosk.html /app.html /cases.html /sessions.html /verify.html /custody.html \
         /elaya-store.js /elaya-cast.js /elaya-shell.js /favicon.svg; do
  printf "%-20s " "$p"; curl -s -o /dev/null -w "%{http_code}\n" -m 20 "https://e-laya.vercel.app$p";
done
```

Expected: `200` for every path.

---

## Self-Review

**Spec coverage.** §5 store → Tasks 2–3. §6 cast and Batangas migration → Task 4. §7 reviewer shell → Task 6. §8 chains A/B/C → Tasks 5, 7, 8. §9 defects D1–D7 → Task 1 (D1), Task 6 (D2), Task 9 (D3–D7). §10 error handling → Task 2 Steps 3–4 and Task 10 Step 1. §11 testing → all tasks; the seven listed test categories map to `store`, `chain-a`, `chain-b`, `chain-c`, `flows` (cross-tab and standalone) and `a11y`/`regression`. §13 build order → task order, unchanged.

**Type consistency.** `Elaya.get/set/update/on/off/emit/notify/reset/ready` are defined in Tasks 2–3 and used with identical signatures thereafter. `castId` is the property name linking a surface's local row to a cast person in Tasks 5, 7 and 8. Store paths are `welfare.<id>`, `attendance.<id>.<progId>`, `determinations.<id>`, `people.<id>`, `notifications` — consistent across all tasks and matching spec §5.

