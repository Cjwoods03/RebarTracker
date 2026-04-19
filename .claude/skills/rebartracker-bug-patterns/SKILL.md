---
name: rebartracker-bug-patterns
description: Pattern library of common bugs, root causes, and fixes for the RebarTracker codebase. Consult this skill whenever debugging a RebarTracker issue, investigating unexpected behavior, or when a fix attempt didn't work as expected. Covers footage rule false blocks, Area zone data shape errors, stale counter bugs, dropdown visibility issues, freeStack filter omissions, Firebase persistence gaps, and modal field wiring misses. Reading this first often identifies the root cause without needing to read index.html end-to-end.
---

# RebarTracker Bug Patterns

A living pattern library built from real bugs encountered during development. Each pattern has a symptom, root cause, and the correct fix approach. Check here before reading index.html end-to-end — the root cause is often one of these.

---

## Pattern 1 — Footage rule blocking a valid write

**Symptom:** A bundle placement fails with an error like "⛔ This run is Xft but Stanchion N only has Y.0ft remaining" even though the placement should be physically valid.

**Root cause:** The footage check in `logRun` (~line 7362) runs before accounting for two valid exceptions:

1. **Same-product merge.** If the new bundle is identical product to what's already on the slot side, it merges rather than adding new footage. Footage check should be skipped entirely.
2. **Secondary stack shorter than primary.** A secondary item stacked on top of the primary doesn't change `stanchionFtUsed` — that function only counts the LONGEST bar per side. If the new item's length is ≤ the primary's length, the delta to remaining footage is zero.

**Correct fix approach:**
- Capture `isSameProductMerge` and `isSecondary` flags BEFORE the footage check runs
- For same-product merge: skip the footage check entirely
- For secondary stack where `lenNum <= currentMaxLenOnSide`: skip the footage check entirely
- For secondary stack where `lenNum > currentMaxLenOnSide`: only block if `(lenNum - currentMaxLenOnSide) > remainFt` (delta logic)
- Reference: `confirmSecondaryTransfer` (~line 4667) already uses this delta logic correctly — mirror it

**Where to look:** `logRun` footage check block. Also `stanchionFtUsed` (~line 3233) to confirm it returns max-per-side, not sum.

---

## Pattern 2 — Area 1 / Area 22 loc field has wrong format

**Symptom:** A bundle was added to Area 1 or Area 22 but doesn't appear on the map, doesn't show up in inventory, or `goToLoc` navigates to the wrong place.

**Root cause:** The `item.loc` field was stored as `'AREA1'` or `'AREA22'` (no dash) instead of `'AREA-1'` or `'AREA-22'` (with dash). Multiple places in the renderer check for the dashed form:

```js
var isArea22 = item.loc === 'AREA-22';  // renderInventory ~line 7969
var isArea1  = item.loc === 'AREA-1';   // renderInventory ~line 7970
```

If the dash is missing, `isArea1` and `isArea22` are both false, and the item falls through to stanchion-slot logic where it doesn't belong.

**Correct fix approach:**
- Any function that creates or moves an item to Area 1/22 must write `loc: 'AREA-1'` or `loc: 'AREA-22'` with the dash
- The dropdown `value` attribute can use `'AREA1'`/`'AREA22'` for routing, but the stored item.loc must use the dashed form
- Check: `area1Stacks.push({ ...item, loc: 'AREA-1', side: side, id: 'P' + area1NextId })`
- Grep for `'AREA1'` and `'AREA22'` (no dash) in any new save function — those are bugs

**Where to look:** Any function that pushes to `area1Stacks` or `area22Stacks`. The transfer confirm path (`confirmTransfer`, `_doPrimaryAreaTransfer`). The Area modal save functions.

---

## Pattern 3 — Stale counter IDs (P07 with one stack)

**Symptom:** Area 1 or Area 22 shows a high stack ID like P07 or S15 even though there are only 1-2 stacks in the zone, or the zone is empty.

**Root cause:** `area1NextId` and `area22NextId` only ever increment — they never reset when stacks are removed. When the array empties and new stacks are added later, the counter continues from wherever it left off. This is compounded by Firebase persistence: the stale high value gets written to `warehouse/v1` and is restored on every page reload even after the fix.

**Correct fix approach (two parts):**

1. **On any removal that empties the array:**
```js
if (area1Stacks.length === 0) area1NextId = 1;
if (area22Stacks.length === 0) area22NextId = 1;
saveData(); // must persist AFTER the reset
```

2. **Load-time normalization in `loadData`** (~line 9083): after reading from Firebase, if the array is empty but the counter is > 1, reset and flush:
```js
if (area1Stacks.length === 0 && area1NextId > 1) { area1NextId = 1; needsSave = true; }
if (area22Stacks.length === 0 && area22NextId > 1) { area22NextId = 1; needsSave = true; }
if (needsSave) saveData();
```

3. **EBS import wipe** (line ~6251): the import wipe that clears `store`/`inventory` must also reset area state:
```js
area1Stacks = []; area22Stacks = []; area1NextId = 1; area22NextId = 1;
```

**Do NOT renumber existing stacks mid-flight** — only reset the counter when the array is completely empty. Renumbering breaks any in-flight reference to a specific stack ID.

---

## Pattern 4 — Dropdown hides incompatible slots instead of showing them disabled

**Symptom:** The overflow modal, split modal, or transfer modal destination dropdown doesn't show some slots at all. Operator can't see what's stored there or why they can't use it.

**Root cause:** The populate function uses early `return` to skip incompatible slots:
```js
// WRONG — silently hides the slot
if (!isSame) return;
```

The correct behavior (from `populateLocationDropdown` at ~line 7786) is to show ALL slots but disable the incompatible ones with a ⊗ prefix and `opt.disabled = true`:
```js
// CORRECT — shows slot as disabled so operator knows it exists
var opt = document.createElement('option');
opt.value = id;
opt.textContent = '⊗ ' + id + ' — ' + reason;
opt.disabled = true;
opt.style.opacity = '0.5';
sel.appendChild(opt);
return; // after appending the disabled option
```

Exceptions that SHOULD use silent `return` (don't append anything):
- Slots with active flags (scrap, hold, redTag, reserved)
- The source slot itself (can't transfer to yourself)
- freeStack zones when showing grid destinations

**Where to look:** `ofRefreshOverflowDropdown` (~line 6628), `ofFillSplitSel` (~line 6778), and any new dropdown populate function. Compare against the `populateLocationDropdown` pattern as the reference.

---

## Pattern 5 — freeStack filter missing from a loop

**Symptom:** Stanchion 1 or stanchion 22 appears in a dropdown, suggestion list, or optimizer output where it shouldn't. Items placed there disappear because 1A/1B/22A/22B don't exist as grid slots.

**Root cause:** `orderedStanchions()` returns ALL stanchions including freeStack zones (1 and 22). Any loop over `orderedStanchions()` that builds a dropdown or does slot math must filter them:

```js
orderedStanchions().forEach(function(s) {
  // REQUIRED — skip freeStack zones in grid loops
  if (stanchionCapacity[s] && stanchionCapacity[s].freeStack) return;
  // ... grid-slot logic here
});
```

Forgetting this filter is how "stanchion 1" ends up in transfer modal dropdowns, optimizer suggestions, or footage calculations.

**Exception:** Area 1 and Area 22 as free-stack DESTINATIONS should be added as explicit named entries (not via the loop):
```js
// Add these BEFORE the stanchion loop, as special-cased options
sel.add(new Option('★ Area 1 — Open Stack Zone', 'AREA1'));
sel.add(new Option('★ Area 22 — Open Stack Zone', 'AREA22'));
```

**Where to look:** Any function that calls `orderedStanchions()`. Also `calcSuggestLoc` (~line 7020) and `runOptimizer` (~line 6018).

---

## Pattern 6 — Firebase reset didn't persist (in-memory only)

**Symptom:** A value gets reset correctly in memory (you can see it in DevTools Console immediately), but after a page reload it reverts to the old value.

**Root cause:** `saveData()` is called BEFORE the reset line runs, so the old value gets written to Firebase. On reload, `loadData` reads the old value back. The reset happened in memory only.

```js
// WRONG ORDER
saveData();
area1NextId = 1;  // reset runs, but saveData already wrote the old value

// CORRECT ORDER
area1NextId = 1;  // reset first
saveData();       // then persist the new value
```

Also check: if the Firebase path is read by `loadData` with a hardcoded fallback, make sure the fallback is also 1 (not the stale value).

**Where to look:** Any function that resets counters, clears arrays, or modifies state that needs to survive a reload. Always verify by: (1) make the change, (2) check the value in DevTools Console, (3) reload the page, (4) check the value again. If step 4 reverts, `saveData()` was called in the wrong order.

---

## Pattern 7 — Field added to form but not wired to auto-calc

**Symptom:** A new field appears in a modal (weight, pieces, length) but typing in it does nothing — the linked field doesn't update.

**Root cause:** The HTML element exists but no `oninput`/`onchange` event listener was wired to it. In the main LOG RUN form, pieces ↔ weight auto-calc runs via `onPiecesInput()` and `calcPiecesFromWeight()`. When fields with matching names are added to other modals (Area 1, Area 22, edit modal), they need their own wiring with unique element IDs.

The chain toggle is also per-modal — each modal has its own `_a1Linked` / `_a22Linked` boolean. Toggling the link in one modal must not affect another.

**Correct fix approach:**
- Each modal gets its own set of calc functions (e.g. `onA1PiecesInput`, `calcA1PiecesFromWeight`) OR the existing functions get a prefix argument
- Field IDs must be unique: `a1-bdl-wt`, `a1-pieces`, `a22-bdl-wt`, `a22-pieces` — not `inp-bdl-wt` (which belongs to the main form)
- For spool type: skip the calc wiring entirely — coil weight is a fixed dropdown, no auto-calc

**Where to look:** `onPiecesInput` and `calcPiecesFromWeight` in the main form. Mirror those for any new modal that captures weight/pieces.

---

## Pattern 8 — Area modal saved item missing fields

**Symptom:** A bundle logged via the Area 1 or Area 22 modal is missing grade, spec, heat, length with inches, or weight when you open its detail modal later. The main LOG RUN form captures all these but the Area modal didn't.

**Root cause:** The Area 1 / Area 22 modals were originally built as "quick stack" forms with fewer fields. When fields were added to the HTML later (Bug C, Session 1), the save function also needs updating to READ those new fields and include them in the pushed stack object.

**Correct fix approach:**
- Area save functions must read every field that was added to the modal HTML
- Length in the save function must combine feet + inches: `(parseFloat(ftVal) || 0) + (parseFloat(inVal) || 0) / 12`
- The stack object pushed to `area1Stacks` should have the same field set as an item object from `logRun`
- After adding fields to save, verify by logging a bundle and checking `area1Stacks[area1Stacks.length-1]` in DevTools Console — all fields should be present

---

## Pattern 9 — A/B side split not rendering in horizontal view

**Symptom:** Area 1 or Area 22 stacks all appear in one column with no A/B visual divider, even though the stack objects have the correct `side: 'A'` or `side: 'B'` field.

**Root cause:** The horizontal view renderer (`buildArea1HorizCol`, `buildArea22HorizCol`) wasn't filtering stacks by side before rendering them into their respective columns. The block view had the split, but the horizontal view iterated all stacks flat.

**Correct fix approach:**
```js
// WRONG — renders all stacks in one column
area1Stacks.forEach(function(stack) { colA.appendChild(makeArea1Stack(stack)); });

// CORRECT — filter by side into separate columns
area1Stacks.filter(function(s) { return s.side === 'A'; })
  .forEach(function(stack) { colA.appendChild(makeArea1Stack(stack)); });
area1Stacks.filter(function(s) { return s.side === 'B'; })
  .forEach(function(stack) { colB.appendChild(makeArea1Stack(stack)); });
```

**Where to look:** `buildArea1HorizCol` (~line 4127) and `buildArea22HorizCol` (~line 3767). Fix both at the same time — they have the same structure.

---

## Adding new patterns

When a new bug is found and fixed that doesn't match an existing pattern, add it here at the end of the session during the skill update. Format:

```
## Pattern N — [Short descriptive name]

**Symptom:** What the operator or developer sees

**Root cause:** Why it happens, with code reference

**Correct fix approach:** What to do, with code snippet if useful

**Where to look:** Which functions / line ranges to check
```

Patterns should describe the bug SHAPE, not a specific instance. "Footage rule blocking secondary" is a pattern. "Bug A from April 17 session" is not.
