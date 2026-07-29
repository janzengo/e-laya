# e-Laya — Unification Design

**Date:** 29 July 2026
**Status:** Approved for planning
**Repo:** `uxieee/e-laya` · deploys to https://e-laya.vercel.app

---

## 1. The problem

e-Laya ships six surfaces — `kiosk`, `app`, `cases`, `sessions`, `verify`, `custody`. Each one
works. Together they do not read as a product. They read as six unrelated demos, because that is
literally what they are.

Three independent facts establish this:

**No shared state.** There is no persistence of any kind.

```
grep -riE 'localStorage|sessionStorage|indexedDB'  public/*.html   → 0 hits
grep -riE 'kv|redis|postgres|supabase|blob|prisma' api/*.js        → 0 hits
```

Verified live: in `/custody`, confirming one person moved the counter `84 → 85`. After reload it
was `84` again. The only cache in the codebase is an in-memory API-token cache in `api/_lib.js`,
which is per-invocation and ephemeral.

**No navigation.** Every surface reports zero outbound links. `index.html` links out to all six;
nothing links back. The `‹ Back` control in `custody.html` was clicked in a real browser and did
nothing — the URL never changed.

**No shared world.** The surfaces are set in different cities and share no people.

| Surface | Locality |
|---|---|
| `kiosk`, `app` | Batangas City |
| `cases`, `sessions`, `custody`, `verify` | Quezon City |

Almost no person appears in two surfaces. There is exactly one deliberate exception:
**Renz A. Bautista**, 16, exists in `sessions.html` (`missStreak: 3`) and in `cases.html`
(`cicl: true`, MTC Br. 12) as the same individual. That link is real intent, not coincidence, and
this design builds on it rather than inventing a pattern from scratch.

(`Jomar` occurs in two files but is two different people — `Jomar Cruz`, 16, in `app.html` versus
`Castañeda, Jomar L.`, 15, in `sessions.html`. A name collision, not a link.)

Even for Renz, the link is static: both files hardcode their own copy of him, so a flag raised in
`sessions` never reaches `cases`.

So an officer confirming welfare in `/custody` can never reach the family in `/app`. Not because
the two aren't linked, but because the fact was never stored, there is no route between them, and
they are not about the same person in the same city.

## 2. What is NOT the problem

Testing contradicted the assumption that the surfaces are rushed and half-built.

**Screen coverage meets or exceeds BUILD-SPEC.**

| Surface | Spec | Built |
|---|---|---|
| kiosk | K-1…K-6 (6) | 11 |
| app | A-1…A-6 (6) | 12 |
| verify | V-1…V-4 (4) | 14 |
| sessions | S-1…S-4 (4) | 6 |
| cases | C-1…C-3 (3) | 4 |
| custody | W-1…W-2 (2) | 1 + 3 dialogs |

**Responsive layout is sound.** `cases.html` at 390 × 844: `scrollWidth === clientWidth`, zero
overflowing elements, 1 of 49 controls under 32 px.

**The API integration is real.** Tapping Binisaya on the kiosk fired `POST /api/ai` → `200` and
rendered live Cebuano. `/api/psgc` returns real PSGC regions; `/api/chain` returns live block
height.

**Translation is credit-safe.** Static UI strings are pre-baked per language in source; only
dynamic content (programme names, notes) calls `/api/ai`, memoised on a content hash. The nine
languages therefore survive the 31 July credit expiry; only translated dynamic content degrades.

The conclusion that shapes this whole design: **do not rebuild the six surfaces.** Build the spine
between them and fix a short list of defects.

## 3. Non-goals

- No framework, no bundler, no build step. The surfaces stay dependency-free static HTML.
- No SPA rewrite. Converting ~354 KB of working inline JS to a client-router is a large rewrite
  with no user-visible benefit.
- No server-side database. See §5 for why this is a deliberate choice, not a shortcut.
- No visual redesign. The existing craft is good; changing it risks regressions for no gain.
- No new screens. Coverage already exceeds spec.

## 4. Architecture

Three new files, loaded by all six surfaces before their existing inline scripts:

```
public/
  elaya-store.js     canonical state + persistence + cross-tab sync
  elaya-cast.js      the shared cast: people who appear in more than one surface
  elaya-shell.js     the reviewer bar (navigation between surfaces)
  favicon.svg        fixes the site-wide 404
```

Each is a classic script (not an ES module) defining a single global, because the surfaces use
inline non-module scripts and must keep working from `file://`.

```
                    ┌──────────────────┐
                    │  elaya-store.js  │  window.Elaya
                    │  localStorage    │
                    └────────┬─────────┘
                             │ read / write / subscribe
   ┌──────────┬──────────┬───┴────┬──────────┬──────────┐
 kiosk       app       cases   sessions   verify    custody
   └──────────┴──────────┴────────┴──────────┴──────────┘
                             │
                    ┌────────┴─────────┐
                    │  elaya-shell.js  │  reviewer bar, injected into each
                    └──────────────────┘
```

## 5. The shared store — `elaya-store.js`

### Why client-side, not a database

A server-side store would make every visitor share one mutable world. A stranger could set every
person's welfare to "released" and leave the deployed demo permanently broken, with no auth system
to prevent it. Client-side state gives each viewer their own coherent world, requires no auth,
works offline, and keeps the surfaces deployable as static files.

The store is written behind a narrow interface so a server adapter can replace the persistence
layer later without touching any surface.

### Interface

```js
window.Elaya = {
  get(path, fallback),        // Elaya.get('welfare.miguel')
  set(path, value),           // persists + notifies, returns value
  update(path, fn),           // read-modify-write
  on(event, handler),         // 'change' | 'welfare' | 'attendance' | 'notification'
  off(event, handler),
  emit(event, payload),
  notify({to, body, surface}),// appends to the notification log
  reset(),                    // restore seed state
  ready(fn)                   // runs fn once store is hydrated
}
```

### Shape

```js
{
  version: 1,
  seededAt: '2026-07-29T00:00:00+08:00',
  people:        { [id]: PersonRecord },   // from elaya-cast.js, mutable
  welfare:       { [personId]: { key, at, by, source, expiresAt } },
  attendance:    { [personId]: { [programmeId]: { done, of, lastAt, receipt } } },
  determinations:{ [personId]: { category, age, at, by } },   // CICL | PDL, from /verify
  notifications: [ { id, to, body, at, surface, personId } ],
  updatedAt: '…'
}
```

`PersonRecord` follows the shape already used by `app.html`'s `PEOPLE` array, which is the richest
existing model (identity, facility, agency, officer, hotlines, visiting hours, welfare, caseState,
hearing, gcta, programme). Other surfaces project a subset of it.

### Persistence and degradation

- Key `elaya.v1` in `localStorage`.
- If `localStorage` throws or is unavailable (private mode, some `file://` origins), the store
  falls back to in-memory and sets `Elaya.persistent === false`. Every surface keeps working; only
  cross-reload durability is lost. Nothing is allowed to throw at the surface level.
- On a `version` mismatch the store discards the stored blob and re-seeds. No migrations.

### Cross-tab sync

The store listens for the `storage` event and re-hydrates, then emits `change`. Two windows open
side by side — `/custody` and `/app` — update each other live. This is the single most convincing
demonstration that the six are one system, and it costs one event listener.

## 6. The shared cast — `elaya-cast.js`

### Unify on Batangas City

The six surfaces must be set in one place. Batangas City is the choice: `kiosk` and `app` already
use it, and it is the team's own province (Team Ala-Eh).

This requires retargeting locality strings in `cases`, `sessions`, `custody` and `verify` —
facility names, court branches, barangays, office names. The work is mechanical find-and-replace,
low risk, and it is the single largest change in this design.

| Currently | Becomes |
|---|---|
| BJMP Quezon City Male Dormitory | Batangas City District Jail |
| RTC Br. 87 / 104, Quezon City | RTC Br. 3 / 4, Batangas City |
| LSWDO Quezon City | LSWDO Batangas City |
| Payatas, Commonwealth, Batasan Hills, Holy Spirit, Bagong Silangan | Kumintang Ibaba, Alangilan, Balagtas, Pallocan West, Bolbok |

### Bridge cast, not a full merge

Each surface's seed is deliberately tuned to demonstrate that surface's scenarios — `cases` has a
hand-built tier of "attended but never recorded", `sessions` has specific `missStreak` patterns,
`custody` needs 128 people at 84 confirmed. Replacing all of that with one shared dataset would
destroy those demonstrations.

So only a **bridge cast** is shared. Four people exist in the canonical store and appear in every
surface where they belong; each surface keeps its remaining seed as local volume.

| Person | Appears in | Carries |
|---|---|---|
| **Miguel Andres Reyes**, 24 | `custody` roster · `app` (Rosa's son) · `cases` (PAO client) | The welfare chain |
| **Jomar Cruz**, 16 | `sessions` roster · `app` (Rosa's nephew) · `kiosk` | The attendance chain |
| **Renz A. Bautista**, 16 | `sessions` (missStreak 3) · `cases` (`cicl: true`) | The missed-attendance flag. **Already linked by hand** — this promotes the existing duplicate to a single store-backed record. |
| **Unnamed intake** → acquires a name on determination | `verify` → `cases` → `app` | The identity chain |

Bridge people are read from `Elaya.get('people.<id>')`. Non-bridge seed rows stay exactly as they
are in each file.

## 7. The reviewer shell — `elaya-shell.js`

### An out-of-fiction bar, not in-fiction navigation

The kiosk is a public terminal bolted to a wall in a jail. A nav control offering "switch to PAO
caseload" would be a lie about the product. The same applies to the family app.

The shell therefore renders a visually distinct **reviewer bar** — clearly not part of any
surface's fiction — that lets a reviewer move between the six. It:

- is fixed to the bottom edge, dark, compact, and labelled as a review aid;
- lists the six surfaces plus Home, marking the current one;
- exposes `Reset demo state` (calls `Elaya.reset()`) and a persistence indicator;
- is dismissible, and hides on `?bare=1` for clean screenshots and filming;
- remembers dismissal in `localStorage` under `elaya.shell.hidden`.

In-fiction controls stay in-fiction: `custody`'s existing `‹ Back` button is wired to
`history.length > 1 ? history.back() : location.assign('index.html')` rather than being replaced.

`index.html` keeps its current role as the overview and gains return links from every surface.

## 8. The flows that prove one system

These three chains are the acceptance criteria for the whole design.

```
A · WELFARE
   /custody  officer confirms Miguel "OK"
      → Elaya.set('welfare.miguel', {key:'ok', at, by:'JO1 Sarmiento', source:'manual'})
      → Elaya.notify({to:'Rosa Reyes', body:'Miguel is doing OK', surface:'custody'})
   /app      person card shows "Ayos naman siya · confirmed 2 minutes ago"
   /app      notifications list shows the SMS that would have been sent
   (open in two tabs — /app updates live via the storage event)

B · ATTENDANCE
   /sessions social worker logs Jomar present
      → Elaya.update('attendance.jomar.pg1', a => ({...a, done: a.done+1, receipt}))
      → Elaya.notify(guardian)
   /app      programme progress advances 9/24 → 10/24
   /cases    Jomar's case file shows the updated attendance standing
   /kiosk    "Akong mga Programa" reflects the same count

C · IDENTITY
   /verify   determination returns CICL, age 16
      → Elaya.set('determinations.<id>', {category:'CICL', age:16, at, by})
      → Elaya.set('people.<id>', personRecord)
   /cases    the person now exists as a case, flagged for diversion
   /app      guardian sees a new linked person
```

Where a chain's target surface has no live data feed (agency systems do not exist in the sandbox),
the receiving surface reads the store — which is exactly how it would read a real agency feed. The
honesty boundary in `index.html` and the READMEs is unchanged and still accurate.

## 9. Defect fixes

Each was observed directly in a browser.

| # | Defect | Evidence | Fix |
|---|---|---|---|
| D1 | No favicon | `GET /favicon.ico → 404`, the only console error, on all 7 pages | Add `favicon.svg` + `<link rel="icon">` |
| D2 | `custody` Back is a no-op | Clicked; URL unchanged | Wire to `history.back()` / `index.html` |
| D3 | No `<h1>` on 4 surfaces | `cases`, `sessions`, `verify`, `custody` report `h1count: 0` | Add one visually-hidden `<h1>` per surface |
| D4 | Nested `<button>` in kiosk | `button "Basahin: Filipino"` inside `button "…Filipino Wikang Filipino"` | Restructure to two sibling `<button>`s inside one positioned wrapper — the language card and the `Basahin` speak control. Keeps both as real buttons; `role="button"` on a `div` was rejected as strictly worse for assistive tech. |
| D5 | `lang` incorrect | `kiosk` hardcodes `lang="fil"` while offering 9 languages | Set `document.documentElement.lang` on language selection |
| D6 | 1 tap target < 32 px | `cases` at 390 px | Pad to 44 × 44 |
| D7 | Aborted in-flight requests | `POST /api/ai`, `/api/everify` → `net::ERR_ABORTED` on screen change | Confirm the AbortController is intentional; suppress the console noise |

D7 is likely by design (`tryApi` with silent fallback) — confirm before changing.

## 10. Error handling

- The store never throws into a surface. Every `Elaya.*` call is total: on internal failure it logs
  once, sets `Elaya.degraded = true`, and returns the fallback.
- Surfaces keep their existing seeds as the fallback path. If the store is empty, unavailable or
  corrupt, each surface renders exactly as it does today. **The store is an enhancement layer, never
  a dependency.** This preserves the "every surface works standalone" property the README claims.
- A corrupt `localStorage` blob (bad JSON, wrong version) is discarded and re-seeded rather than
  surfaced as an error.

## 11. Testing

A Playwright script, `test/flows.spec.js`, driven against a local static server:

1. **Persistence** — confirm welfare in `/custody`, reload, assert the counter holds.
2. **Chain A** — confirm in `/custody`, open `/app`, assert the person card and the notification.
3. **Chain B** — log attendance in `/sessions`, assert the count in `/app`, `/cases`, `/kiosk`.
4. **Chain C** — complete a determination in `/verify`, assert the case appears in `/cases`.
5. **Cross-tab** — two contexts, write in one, assert the other updates without reload.
6. **Standalone** — with `localStorage` disabled, assert all six still render and are usable.
7. **Regression** — for each of the 7 pages: zero console errors, no horizontal overflow at 390 px,
   exactly one `<h1>`, no nested interactive elements.

## 12. Risks and trade-offs

| Risk | Assessment |
|---|---|
| **Per-browser coherence.** `/custody` on a laptop and `/app` on a phone won't see each other. | Accepted. Invisible for a demo, a reviewer clicking around, or a portfolio piece. The adapter swap in §5 is the escape hatch. |
| **The Batangas migration touches 4 files.** | Mechanical string replacement, covered by the §11 regression pass. Largest change here; worth flagging before starting. |
| **Load order.** Surfaces' inline scripts run before the store hydrates. | `Elaya.ready(fn)` — surfaces register their first render through it. Store hydration is synchronous from `localStorage`, so this is ordering hygiene rather than async complexity. |
| **Bridge-cast drift.** A surface edits its local copy of a bridge person. | Bridge people are read from the store at render time, never copied into local seed arrays. |
| **Scope creep into a real backend.** | Explicitly out of scope. §3. |

## 13. Build order

1. `favicon.svg` + D1 — smallest change, removes the only console error site-wide.
2. `elaya-store.js` with `Elaya.ready`, persistence, degradation, cross-tab sync. No surface changes.
3. `elaya-cast.js` + the Batangas migration across `cases`, `sessions`, `custody`, `verify`.
4. Chain A (`custody` → `app`) end to end — the highest-value flow, and it proves the architecture.
5. `elaya-shell.js` reviewer bar + D2 across all six.
6. Chains B and C.
7. Remaining defects D3–D7.
8. `test/flows.spec.js`, then a full regression pass.

Each step is independently shippable and leaves the site working.

Steps 1–4 form the first milestone: after step 4 the product demonstrably behaves as one system,
because Chain A works end to end. Steps 5–8 broaden that from one proven chain to all six surfaces.
If the work has to stop early, it should stop on a step boundary — never mid-migration in step 3.
