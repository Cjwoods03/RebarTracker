---
name: rebartracker-context
description: Context and architecture reference for the RebarTracker warehouse management app. Consult this skill whenever the user mentions RebarTracker, rebar mill, stanchions, their shipping app, working on index.html in the Cjwoods03/RebarTracker repo, Firebase rebar-tracker, or any feature work / bug fix / session planning on this codebase. Contains repo facts, architecture patterns, item schema, Area 1/Area 22 free-stack specifics, key function locations, session history, and session-close maintenance reminders. Reading this first prevents re-asking about tenant keys, Firebase structure, what's already built, or what patterns to follow for new features.
---

# RebarTracker Context

Single-file HTML warehouse management app for a Nucor Steel rebar mill shipping department. Built and maintained by Carter Woods, an overhead crane operator, on personal time. Carter is not a developer by trade — keep explanations direct and practical, avoid unnecessary jargon.

## Repository and deployment

- **Repo:** github.com/Cjwoods03/RebarTracker
- **Branch:** `master` (not main — master is ahead)
- **Live URL:** cjwoods03.github.io/RebarTracker (GitHub Pages, rebuilds ~60 seconds after push)
- **Main file:** `index.html` (single-file app, ~9000+ lines, HTML+CSS+JS)
- **Dev workflow:** Claude Code on personal Mac, commits pushed directly to master
- **Local path:** `~/RebarTracker`

## Firebase

- **Project:** rebar-tracker
- **Type:** Realtime Database (NOT Firestore)
- **Storage:** Firebase Storage enabled for shift note and flag photos
- **All writes are tenant-scoped** through the `ref(path)` helper (~line 8295) which prefixes `tenants/{tenantKey}/`. Never write to the database root directly.

## Active tenant

- **Tenant key:** `NUCOR-SEDALIA-01`
- **Mode:** locked to `rebar` via TENANT_MODES (~line 2559)
- **Import profile:** `ebs` (Oracle E-Business Suite CSV)
- **Operator name:** Carter (stored in sessionStorage after first prompt)

## Architecture patterns — respect these in every change

### 1. Tenant scoping
Every Firebase write goes through `ref(path)` — never `db.ref(path)` directly. This ensures the tenant prefix is always applied.

```js
ref('loads').push()              // writes to tenants/NUCOR-SEDALIA-01/loads/
ref('audit/' + entryId).set(...)  // writes to tenants/NUCOR-SEDALIA-01/audit/
```

### 2. saveData vs incremental writes
`saveData()` (~line 8373) is debounced 800ms and rewrites the ENTIRE `warehouse/v1` payload. It contains the main state: `store`, `storeExtra`, `inventory`, `area1Stacks`, `area22Stacks`, slot flags, stanchion capacity, optimizer rules, runCycle. 

**Never add new features to the warehouse/v1 payload.** New features go in SEPARATE Firebase paths with incremental `.push()` / `.set()` / `.update()` writes:

- `tenants/{key}/audit/{id}` — audit log
- `tenants/{key}/shiftNotes/{id}` — shift notes
- `tenants/{key}/loads/{id}` — loads
- `tenants/{key}/shippedBundles/{uid}` — shipped bundle archive
- `tenants/{key}/slotFlags/{encoded-slotId}` — flag metadata
- `tenants/{key}/bols/{id}` — generated BOLs
- `tenants/{key}/customers/{id}` — customer directory
- `tenants/{key}/shipper` — single shipper record

### 3. LBL() for user-facing strings
The app is rebar-only now (non-rebar modes removed). All user-facing strings still route through `LBL(key)` (~line 2670) — the function exists, the rebar labels are hardcoded:

- `LBL('stan')` → "Stanchion"
- `LBL('bundles')` → "bundles"
- `LBL('bdlAbbr')` → "bdl"

Never hardcode these strings directly — always use LBL() so adding a mode in future doesn't require hunting callsites.

### 4. Firebase key encoding
Firebase keys cannot contain `. # $ [ ] / -`. Slot IDs like `RACK-1.1` or secondary keys with those characters must go through `fbEncode()` / `fbDecode()` (~line 8327). Encoded: `RACK-1.1` → `RACK__DASH__1__DOT__1`.

Helpers `encodeObj()` / `decodeObj()` apply the encoding to entire object keys. `encodeInventory()` / `decodeInventory()` handle the `.loc` field on inventory items.

### 5. orderedStanchions() returns ALL stanchions
`orderedStanchions()` (~line 3207) returns stanchions 1-38 plus BSU (skips 22). Callers MUST filter freeStack zones themselves:

```js
orderedStanchions().forEach(function(s) {
  if (stanchionCapacity[s] && stanchionCapacity[s].freeStack) return;
  // ... work on grid-addressable stanchions only
});
```

Stanchions 1 and 22 are marked `freeStack: true` in `stanchionCapacity` (~line 2846). They render as Area 1 and Area 22 open stack zones, NOT as A/B/C grid slots.

Forgetting this filter causes bugs like items disappearing into "1A" or "22B" that don't exist on the map.

## Item schema

Every bundle object in `store`, `storeExtra`, or `inventory`:

```js
{
  uid: 'genUid()-output',
  size: '#5',              // bar size in rebar mode; varies by APP_MODE
  length: '20.5',          // decimal feet as STRING (empty string for spools)
  type: 'straight',        // 'straight' | 'spool'
  grade: '60',             // '40' | '60' | '80' in rebar
  spec: 'A615',            // 'A615' | 'A706' in rebar
  bundles: 5,              // count
  pieces: 20,              // pieces per bundle
  weightPerBundleTons: 0.521,
  weightTons: 2.605,
  spoolWeightLbs: 0,       // only set for spools
  customer: 'ACME',
  heat: 'HT-1234-567',     // manual entry in main form + Area modals (Session 2) or from CSV
  lpns: 'LPN001, LPN002',  // comma-separated from CSV import
  order: 'PO-9876',
  ship: '2026-05-01',
  loc: '5A',               // slot ID, or 'AREA-1' / 'AREA-22' for free-stack items
  logged: '2:47:15 PM',    // time-of-day string (legacy; superseded by receivedDate)
  receivedDate: '2026-04-17T14:47:15.123Z',  // ISO 8601 (from Session 2)
  receivedDateBackfilled: true,              // flag for estimated dates (legacy items)
  loadId: null,            // from Session 6 — uid of load this bundle is staged to
  
  // Mode-specific fields (populated only in matching modes)
  species, finish, pcWidth, pcThickness,
  endFinish, coating, glassWidth, glassHeight,
  cableAwg, cableConductors, cableColor
}
```

## Area 1 / Area 22 — free-stack zones

These are NOT grid slots. They behave very differently from stanchions 2-21 and 23-38.

- Stored in `area1Stacks[]` and `area22Stacks[]` arrays, NOT in the `store` map
- Each element is a FULL item object plus stack metadata (`id`, `side`)
- Stack IDs: `P##` for Area 1 (from `area1NextId`), `S##` for Area 22 (from `area22NextId`)
- **item.loc field must be the literal string `'AREA-1'` or `'AREA-22'` with the DASH** — required by renderInventory and goToLoc
- Each stack has a `side` field (`'A'` or `'B'`) for the A/B column renderer
- Counter reset (Session 1 Bug E): when the array becomes empty, the counter resets to 1, flushed via saveData() + normalized again in loadData on reload

When adding new transfer paths, features, or import logic that touches Area 1 / Area 22, remember:
- Route through the A/B side picker (follow pattern from commit 658dac9)
- Always write `loc` as `'AREA-1'` / `'AREA-22'` with the dash
- Empty-array counter normalization runs in loadData (~line 9083-9086)

## Slot flag system

Four boolean maps at ~line 2923 provide slot-level status flags:

- `scrapSlots[slotId] = true` — ⬜ scrap
- `holdSlots[slotId] = true` — 🔵 hold
- `redTagSlots[slotId] = true` — 🔴 red tag
- `reservedSlots[slotId] = true` — 🟣 reserved (staged for upcoming run)

From Session 5 onward, metadata (reason, photos, setBy, setAt, notes) lives at the separate path `tenants/{key}/slotFlags/{fbEncode(slotId)}`. The booleans stay as they are — they're what drives filter logic throughout the app (suggest engine, optimizer, dropdowns). The metadata is additive.

Every place that iterates slots for availability should filter flagged slots:
```js
if (scrapSlots[id] || holdSlots[id] || redTagSlots[id] || reservedSlots[id]) continue;
```

## Key function locations

Line numbers drift. Use function names as the primary locator, `~line` as a confirmation hint.

### Core data
- `buildStore`: ~2928 — initializes `store` with null for every slot
- `orderedStanchions`: ~3207 — returns all stanchions including freeStack
- `slotAvailable`, `slotAllItems`: ~3221-3231
- `stanchionFtUsed`: ~3233 — only counts longest bar per side
- `maxFtForStanchion`: ~2863 — honors overrides (BSU=70, stan 2=70, stan 3=70)

### Suggest and place
- `calcSuggestLoc`: ~7020 — Pass 1 (merge), Pass 2 (short), Pass 3 (any)
- `suggestLocation`: ~7142 — UI wrapper
- `logRun`: ~7362 — main form submit
- `populateLocationDropdown`: ~7786 — the correct reference pattern for showing all slots with disabled state
- `autoPickLoc`: ~7727

### Transfer and movement
- `openTransferModal`: ~4737 — primary item transfer
- `openSecondaryTransfer`: ~4620 — secondary item transfer (function name may vary)
- `confirmTransfer`: dispatches primary transfers (including Area destinations since commit 658dac9)
- `confirmSecondaryTransfer`: ~4667

### Overflow and split modals
- `ofOpen`: ~6479 — overflow from main form
- `openSplitFromForm`: ~6525 — split from ⇌ button
- `ofRefreshOverflowDropdown`: ~6628 — rebuilt by Bug 2 fix (Session 1)
- `ofFillSplitSel`: ~6778 — rebuilt by Bug 2 fix (Session 1)

### Space Optimizer
- `runOptimizer`: ~6018 — Space Optimizer entry point
- `OPTIMIZER_BUILT_IN_RULES`: ~5925 — six built-in rules
- `executeOptimizerMove`: ~6405

### Import
- `runCSVImport`: ~5291 — EBS and generic CSV import
- `parseEBSLocation`: ~5258
- `IMPORT_PROFILES`: ~2577 — EBS and generic profiles

### Firebase and persistence
- `initFirebase`: ~8299
- `ref(path)`: ~8295 — tenant-scoped ref helper
- `fbEncode` / `fbDecode`: ~8327
- `saveData`: ~8373 — debounced 800ms bulk write
- `loadData`: ~8421 — includes migrations and counter normalization

### Rendering
- `renderAll`: ~3381
- `renderInventory`: ~7894
- `buildModalRows`: ~4118 — slot detail modal
- `makeSlot`: (search for `function makeSlot`)
- `makeHorizCol`: ~3545
- `buildArea1Block`, `buildArea1HorizCol`: Area 1 renderers
- `buildArea22Block`, `buildArea22HorizCol`: Area 22 renderers

## Workflow rules (quick reference)

- One commit per discrete fix. Never bundle multiple fixes.
- Push to master immediately after each commit. Credentials are cached in macOS Keychain — pushes should be silent. If a push fails with "Device not configured" or similar, the fix is for the user to push from their own Terminal once to refresh the keychain, then Claude Code's future pushes work.
- Discovery before implementation for anything non-trivial — report findings, wait for explicit approval, then implement.
- Syntax-check before committing.
- Never leave debug `console.log` statements in committed code.
- When the user shares a plan or test matrix, that's CONTEXT, not a work order. Wait for `[IMPLEMENT NOW]` or equivalent explicit instruction.

## Recent work log

Track recent significant commits here, newest at top. Trim entries older than ~30 commits to keep the file lean.

### April 2026 — Rebar-only cleanup + optimizer overhaul
- a2ae14c — Optimizer cards: replace misleading slot capacity with actual move weight (weightTons · bundles)
- 6a1fbcf — Optimizer: isStanchionEmpty helper; defrag + consolidate_same prefer occupied destinations as tiebreaker; consolidate_spools rule added (7th built-in)
- a752347 — Optimizer crane-travel: stanchionPos/stanchionDistance helpers (BSU=3.5), defrag and fill_partner prefer nearest destinations
- 3432086 — Optimizer fill_partner: picks longest source that fits instead of any fit
- 3556a47 — Optimizer cleanup: pruned 11 dead mode fields from consolidate_same key, added bundle capacity check to merges, tiered fill_partner priorities
- Shift boundaries reduced from 3 whole-hour inputs to 2 HH:MM time inputs (Day/Night)
- Dashboard: removed loads KPIs, throughput, quality/ops; added Daily Tonnage On Hand chart reading from tonnageLog path
- fbc83bd — Removed mode-switching functions and non-rebar APP_MODE branches
- 7f6766d — Removed mode-specific form fields from all modals
- 0406b8c — Removed mode selector UI and mf-* CSS
- 78f6912 — Removed non-rebar mode definitions from MODES (rebar-only now)
- 51aac36 — Shift notes: added shift/date/crew fields, archive-by-age toggle
- 2ae350c — Shift notes: removed category field

### Earlier work
- Item schema: every item has receivedDate (ISO 8601) and optionally receivedDateBackfilled: true
- Helper functions: formatReceivedDate / getItemAgeDays / getAgeClass / formatAgeShort / formatAgeLong
- Age stripe color classes applied to map slots, inventory cards, modal Age row
- PICK FIRST badge on oldest item in stacked slots

## New Firebase paths added
- tenants/{tenantKey}/tonnageLog/{YYYY-MM-DD} — { date, tons } written on app load via set (deduplicates by date)

## Notable dead code removed
- setAppMode, openModeSelector, closeModeSelector, handleModeSelectorClick
- parseLumberDims, mode-specific branches in calcBundleWeight, getFormSize, buildModalRows, logRun, renderInventory
- Dashboard KPIs: kpiReceivedInWindow, kpiShippedInWindow, kpiShippedWeightTonsInWindow, kpiBundlesOnHand, kpiTonsOnHand, kpiMTRPendingActive, kpiStaleHolds, kpiRedTagCount, kpiAvgTurnaroundDays, kpiActionsToday, kpiMostActiveOperator, kpiReceiveShipByDay, kpiLoadsShippedInWindow, kpiLoadsInProgress
