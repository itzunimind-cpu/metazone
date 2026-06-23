# METAZONE — BUGS TO FIX
> Generated from full codebase audit. Every bug verified against live code before inclusion.
> Last verified: vod.html (post-patch v26 + 7 applied fixes), editor.html (unmodified).

---

## ALREADY FIXED — vod.html (applied, do not redo)

These 7 fixes are confirmed present in the current `vod.html`:

| ID | What was fixed |
|----|----------------|
| F1 | `btn-apm-confirm` never re-enabled after successful SET SQUAD → now re-enabled on success path |
| F2 | Enemy team wiped state in squad panel ignored `state.time` → now time-aware |
| F3 | Killed-by + enemy wipe dropdowns used static `!time_of_wipe` filter → now `time_of_wipe > state.time` |
| F4 | `renderEventLog()` missing from 250ms polling loop → event log lit/unlit state now reactive to scrubbing |
| F5 | `setVideoState('idle')` unconditionally overwrote `'loading'` at end of `onMatchChange` → now guarded by `if (!_ytPlayerReady)` |
| F6 | Guard block (`clearTimeout`, `_ytPlayerReady=false`) ran AFTER async annotation dialog → moved to first line of `onMatchChange` |
| F7 | `updateMatchContextBar()` missing from `_doStartMatch()` → added after `updateVodMatchStatus()` |

---

## REMAINING BUGS

### How to read this document

- **File** — which HTML file to open
- **Function** — exact JS function name to search for (`Ctrl+F` in your editor)
- **What breaks** — user-facing symptom
- **Root cause** — exact code problem
- **Fix** — what to change

---

## PHASE 1 — Data Integrity
> Fix these first. These corrupt or silently lose real data.

---

### BUG V1 + E5 — Winner gets `time_of_wipe` in auto-match-end
**Files:** `vod.html` AND `editor.html`
**Functions:** `confirmEnemyWipe` (vod, ~line 847) | `confirmWipe` (editor, ~line 2343)
**Severity:** 🔴 HIGH — corrupts placement data in DB

**What breaks:**
When you mark the second-to-last team as wiped, the system auto-resolves the last surviving enemy as the winner. It correctly gives them `confirmed_position = 1` but ALSO sets `time_of_wipe = current_time`. The winner never died — they should have no wipe time.

Downstream damage from this one line:
- Winner appears crossed-out/eliminated in the squad panel
- `calcWipePosition()` counts the winner in `wipedSoFar`, making every subsequent position suggestion off by 1
- Timeline shows a wipe marker for the winner at match-end time
- Enemy wipe dropdown shows winner as `"↩ TeamName (undo wipe)"` — clicking them accidentally clears their #1 placement

**Root cause (both files, Scenario A and B):**
```js
// vod.html confirmEnemyWipe — Scenario A (~line 893) and B (~line 901)
winner.confirmed_position = 1;
winner.time_of_wipe = t;        // ← WRONG. winner survived, remove this line.

// editor.html confirmWipe — Scenario A and B
winner.confirmed_position = 1;
winner.time_of_wipe = state.time; // ← WRONG. remove this line.
```

**Fix:** Remove `winner.time_of_wipe` assignment in both Scenario A and B blocks in both files. Only set `confirmed_position = 1`. Compare with Scenario C (self wins) which correctly does NOT set `selfTeam.time_of_wipe`.

```js
// CORRECT pattern — match what Scenario C does:
winner.confirmed_position = 1;
// no time_of_wipe — winner survived
await sb.from('teams').update({ confirmed_position: 1 }).eq('id', winner.id);
```

---

### BUG E1 — `manualSave` only saves the active team's shapes
**File:** `editor.html`
**Function:** `manualSave` (~line 589)
**Severity:** 🔴 HIGH — silent data loss

**What breaks:**
Draw shapes for Team A. Switch to Team B. Draw more shapes. Press SAVE. Only Team B's shapes (currently active) get written to the DB. Team A's new shapes have no DB ID and are permanently lost on the next page reload. No error, no warning.

**Root cause:**
```js
async function manualSave() {
  if(state.activeTeamId) {
    const shapes = shapesCache[state.activeTeamId] || []; // ← only active team
    const toInsert = shapes.filter(s => !s.id || String(s.id).startsWith('local_'));
    // shapes for all other teams are never touched
  }
}
```

**Fix:** Iterate ALL keys in `shapesCache`, not just the active team:
```js
async function manualSave() {
  // Save ALL teams' unsaved shapes
  for (const [tid, shapes] of Object.entries(shapesCache)) {
    const toInsert = shapes.filter(s => !s.id || String(s.id).startsWith('local_'));
    if (!toInsert.length) continue;
    const rows = toInsert.map(s => shapeToDbRow(s, tid, state.matchId));
    const { data, error } = await sb.from('shapes').insert(rows).select();
    if (error) throw error;
    data.forEach((row, i) => { toInsert[i].id = row.id; });
  }
  // rest of save (zones etc.) unchanged
}
```

---

### BUG E9 — `deleteSelected` orphans `match_players` rows
**File:** `editor.html`
**Function:** `deleteSelected` (~line 1549), match deletion block (~line 1612)
**Severity:** 🔴 HIGH — orphaned DB rows (if no ON DELETE CASCADE set up)

**What breaks:**
When deleting a match, the code deletes `shapes` → `teams` → `matches` in order. But `match_players` rows for that match are never deleted. If Supabase doesn't have `ON DELETE CASCADE` on the `match_players.match_id` FK (marked as pending in the handoff), these rows orphan in the DB. Same problem exists for the day delete and tournament delete paths.

**Root cause (match delete block ~line 1620):**
```js
const tIds = teams.filter(t => t.match_id === m.id).map(t => t.id);
if (tIds.length) await sb.from('shapes').delete().in('team_id', tIds);
if (tIds.length) await sb.from('teams').delete().in('id', tIds);
// match_players DELETE is missing here ↑
await sb.from('matches').delete().eq('id', m.id);
```

**Fix:** Add `match_players` cleanup before deleting the match:
```js
// Add before the matches.delete() line:
await sb.from('match_players').delete().eq('match_id', m.id);
```
Apply the same pattern in the day delete and tournament delete blocks — collect all `match_ids` for the day/tournament, then delete `match_players` for all of them before deleting the matches.

---

### BUG E4 — Position collision: two teams get the same `confirmed_position`
**File:** `editor.html`
**Function:** `confirmWipe` (~line 2343), Scenario A block
**Severity:** 🟡 MEDIUM — corrupt placement data, no validation

**What breaks:**
With 2 enemies remaining, the user opens the wipe modal for one of them and manually types `2` as the position (meaning that team finished 2nd). When saved: `survivingEnemies.length === 1` → Scenario A fires and assigns `selfTeam.confirmed_position = 2`. Now two teams have `confirmed_position = 2` in the DB. No deduplication check exists anywhere.

Also possible with any manually-entered position that clashes with auto-assignment.

**Root cause (Scenario A):**
```js
selfTeam.confirmed_position = 2; // assigned with no check if #2 is already taken
await sb.from('teams').update({ confirmed_position: 2 }).eq('id', selfTeam.id);
```

**Fix:** Before auto-assigning self's position, check the slot isn't already taken:
```js
if (!selfTeam.confirmed_position) {
  const matchTeams = teams.filter(t => t.match_id === state.matchId);
  const pos2taken = matchTeams.some(t => t.confirmed_position === 2 && t.id !== selfTeam.id);
  if (!pos2taken) {
    selfTeam.confirmed_position = 2;
    await sb.from('teams').update({ confirmed_position: 2 }).eq('id', selfTeam.id);
  }
}
```

---

## PHASE 2 — Editor Logic Bugs
> These break specific workflows without corrupting data. Fix after Phase 1.

---

### BUG E2 — Auto match-end fires 600ms into an active pin prompt
**File:** `editor.html`
**Function:** `confirmWipe` (~line 2343) — all three scenario blocks
**Severity:** 🔴 HIGH — two modal states stack, canvas taps intercepted

**What breaks:**
After confirming a wipe, `openPinPrompt` opens (user is supposed to tap the wipe location on the map). This sets `_pinPromptActive = true` and returns immediately. Within the same synchronous block, if the last-team condition is met, `setTimeout(()=>markMatchEnd(), 600)` is queued. 600ms later, the results panel opens while the user is mid-tap on the map. Since `_pinPromptActive = true`, all canvas interactions during this window are intercepted by the pin handler, not the results panel.

There are **3 occurrences** of this pattern in `confirmWipe` — one per scenario (A, B, C).

**Root cause:**
```js
// confirmWipe — Scenario A (and B, C similarly)
openPinPrompt(`${team.name} wipe location`, true, async (wx, wy) => { ... });
// ↑ returns immediately, _pinPromptActive = true

// runs right after, 600ms timer queued:
if (!_wipeIsSelf && state.matchId) {
  setTimeout(() => markMatchEnd(), 600); // fires while pin prompt still active
}
```

**Fix — replace auto-fire with a prompt in all 3 scenarios:**
```js
// Instead of setTimeout(markMatchEnd, 600), show a non-blocking toast:
setTimeout(() => {
  if (!_pinPromptActive) {
    toast('🏆 Last team standing — press MATCH END when ready', 'ok');
  } else {
    // defer: re-check after pin prompt resolves
    const origDone = _pinDoneCallback;
    _pinDoneCallback = () => {
      if (origDone) origDone();
      toast('🏆 Last team standing — press MATCH END when ready', 'ok');
    };
  }
}, 200);
```
This mirrors the VOD page's `_showMatchEndPrompt` pattern (which already does this correctly).

---

### BUG E3 — Auto match-end fires prematurely with partial team registration
**File:** `editor.html`
**Function:** `confirmWipe` (~line 2343), all scenario checks
**Severity:** 🔴 HIGH — closes match while real teams may still be alive

**What breaks:**
`survivingEnemies` = teams in the system with `time_of_wipe == null`. In a 16-team match, if the user only registered 5 teams (SELF + 4 enemies), marking the 3rd enemy triggers Scenario A: the 4th enemy auto-gets `#1 + time_of_wipe`, SELF auto-gets `#2`, `duration_seconds` is written to the DB, results panel opens. The 11 real teams that were never registered don't exist in the system — they can't block the trigger.

**Simulated proof:**
- Register 5 of 16 teams
- Mark E4 wiped → survEnemies = 3, no trigger
- Mark E3 wiped → survEnemies = 2, no trigger
- Mark E2 wiped → survEnemies = 1 → **FIRES** → E1 auto #1, SELF auto #2, match closed
- E1 was the last registered enemy, not necessarily the real winner

**Root cause:** The trigger condition has no awareness of how many total teams are expected:
```js
const survivingEnemies = matchTeams.filter(t => !t.is_self && t.time_of_wipe == null);
if (survivingEnemies.length === 1 && selfAlive) { /* auto-fires */ }
```

**Fix (connects to E2 fix):** Since E2 changes auto-fire to a prompt anyway, this problem largely resolves. The prompt lets the user decide when the match is actually over, regardless of registration count. Additionally, consider showing a warning in the prompt: *"1 registered enemy remaining — if more teams exist in this match, register them first."*

---

### BUG E8 — Auto-wipe fires while death pin prompt is active
**File:** `editor.html`
**Function:** `confirmSelfLoss` (~line 2558)
**Severity:** 🟡 MEDIUM — two modal states stack

**What breaks:**
When the last player on your team dies (`confirmSelfLoss`), the function:
1. Opens the death pin prompt → `_pinPromptActive = true`
2. Immediately checks if all squad players are dead
3. If yes: `setTimeout(markSelfWipe, 300)` → the wipe modal opens 300ms later

The user is trying to tap the death location on the map. 300ms later, the self-wipe modal appears. Both `_pinPromptActive` and the wipe modal are active simultaneously. Canvas taps go to the pin handler, not the map area of the wipe modal.

**Root cause:**
```js
// confirmSelfLoss
openPinPrompt(`${player.name} death`, false, async (wx, wy) => { ... });
// ↑ returns immediately

const stillAlive = matchPlayers.filter(p => p.time_of_loss == null || p.time_of_loss > t);
if (stillAlive.length === 0) {
  setTimeout(markSelfWipe, 300); // fires while pin prompt is still open
}
```

**Fix:** Guard the auto-wipe trigger:
```js
if (stillAlive.length === 0 && !_pinPromptActive) {
  setTimeout(markSelfWipe, 300);
} else if (stillAlive.length === 0) {
  // defer until pin resolves — toast hint instead
  toast('☠ All players down — log your wipe when ready', 'warn');
}
```

---

### BUG E6 — Pressing Escape during team rename saves instead of cancelling
**File:** `editor.html`
**Function:** `startRename` (~line 1976)
**Severity:** 🟡 MEDIUM — unexpected data write

**What breaks:**
You double-click a team name to rename it. You type a new name. You change your mind and press Escape. Instead of restoring the original name, Escape calls `input.blur()` which triggers `onblur` which triggers `finish()` which saves whatever is currently in the input to the DB. Escape = save. This is the opposite of expected behavior.

**Root cause:**
```js
input.onblur = finish; // finish() saves to DB
input.onkeydown = e => {
  if (e.key === 'Enter') input.blur();
  if (e.key === 'Escape') input.blur(); // ← triggers finish() = saves
};
```

**Fix:** Track original name, restore on Escape:
```js
function startRename(teamId) {
  const team = teams.find(t => t.id === teamId);
  if (!team) return;
  const originalName = team.name; // capture before edit
  // ... create input element ...
  const finish = async (save) => {
    const newName = save ? (input.value.trim() || originalName) : originalName;
    team.name = newName;
    if (save && sb && !String(team.id).startsWith('local_'))
      await sb.from('teams').update({ name: newName }).eq('id', team.id);
    renderTeamList(); updateFactionBadges();
  };
  input.onblur = () => finish(true);
  input.onkeydown = e => {
    if (e.key === 'Enter') input.blur();
    if (e.key === 'Escape') { input.onblur = () => finish(false); input.blur(); }
  };
}
```

---

### BUG E7 — `markSelfLoss` killed-by dropdown uses static wipe filter
**File:** `editor.html`
**Function:** `markSelfLoss` (~line 2509)
**Severity:** 🟢 LOW — wrong teams in dropdown when reviewing past events

**What breaks:**
The "killed by" dropdown excludes any team that has `time_of_wipe` set, regardless of WHEN they were wiped. If you scrub back to a time before a team's wipe and log a loss, that team won't appear in the dropdown — even though they were alive at that point in the match.

**Root cause:**
```js
teams.filter(t => t.match_id === state.matchId && !t.is_self && !t.time_of_wipe)
//                                                                ↑ static — ignores state.time
```

**Fix:**
```js
teams.filter(t =>
  t.match_id === state.matchId &&
  !t.is_self &&
  (t.time_of_wipe == null || t.time_of_wipe > state.time) // time-aware
)
```

Note: this same fix was already applied to `vod.html` in the previous patch session (F3).

---

## PHASE 3 — Cross-Page Navigation
> Both pages can load the wrong match silently. Fix these together — they're paired.

---

### BUG X1 — `mzCtxWrite()` is defined in editor but never called
**File:** `editor.html`
**Function:** `mzCtxWrite` (~line 1648) — defined but 0 call sites
**Severity:** 🔴 HIGH — nav bar always loads wrong match

**What breaks:**
`mzCtxWrite` exists and correctly writes `{ tournamentId, dayId, matchId }` to `sessionStorage`. But it's never invoked. `sessionStorage` stays empty all session. When the user clicks the "VOD REVIEW" nav bar link (plain `href="vod.html"`), VOD reads sessionStorage → empty → falls back to auto-loading the match with the highest `created_at`. If that's not the match the user was working on, they silently land on the wrong match with no error.

**Root cause:**
```js
function mzCtxWrite(extra) { // defined here
  const ctx = { tournamentId: state.tournamentId, dayId: state.dayId, matchId: state.matchId, ...(extra||{}) };
  try { sessionStorage.setItem('mz_ctx', JSON.stringify(ctx)); } catch(e) {}
}
// Called 0 times anywhere in editor.html
```

**Fix:** Call it from `_activateMatch()` — this runs every time a match loads, so sessionStorage stays current:
```js
async function _activateMatch(match, statusMsg) {
  // ... existing code ...
  updateMatchStatusChip();
  mzCtxWrite(); // ← add this one line at the end
}
```

---

### BUG X2 — `mzCtxWrite` doesn't exist in `vod.html` at all
**File:** `vod.html`
**Function:** `onMatchChange` (~line 1251) — context never written to sessionStorage
**Severity:** 🔴 HIGH — nav bar to editor always loads wrong match

**What breaks:**
Same problem in reverse. When the user clicks "EDITOR" from the nav bar on the VOD page, editor reads sessionStorage → empty → auto-loads by `created_at`. The function doesn't exist in vod.html, not even as a stub.

The only reliable cross-page navigation currently is the **OPEN IN EDITOR** / **OPEN IN VOD** context bar buttons, which carry `?tournamentId=X&dayId=Y&matchId=Z` in the URL. Plain nav bar links always lose context.

**Fix — add the function to vod.html and call it from `onMatchChange`:**

```js
// Add this function near mzCtxRead() in vod.html:
function mzCtxWrite() {
  try {
    sessionStorage.setItem('mz_ctx', JSON.stringify({
      tournamentId: state.tournamentId,
      dayId:        state.dayId,
      matchId:      state.matchId,
    }));
  } catch(e) {}
}
```

Then at the end of `onMatchChange` (after `updateMatchContextBar()`):
```js
updateMatchContextBar();
mzCtxWrite(); // ← add this
```

---

### BUG X3 — Auto-load sorts by `created_at` not `updated_at`
**Files:** `editor.html` (`applyUrlParams`, ~line 1656) | `vod.html` (`applyUrlParams`, ~line 1592)
**Severity:** 🟡 MEDIUM — blocked on pending DB change

**What breaks:**
When either page loads with no URL params and no sessionStorage context, they auto-load the "most recently edited" match — but the sort is by `created_at` (when the match was created), not `updated_at` (when it was last worked on). A user who's been editing Match A all session will always be dropped on Match B if Match B was created more recently.

**Root cause (both files):**
```js
const sorted = [...matches].sort((a, b) => new Date(b.created_at) - new Date(a.created_at));
```

**Fix:** Swap to `updated_at` — **but this requires the pending SQL migration first:**
```sql
ALTER TABLE matches ADD COLUMN updated_at TIMESTAMPTZ DEFAULT now();
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$ BEGIN NEW.updated_at = now(); RETURN NEW; END; $$ language 'plpgsql';
CREATE TRIGGER update_matches_updated_at BEFORE UPDATE ON matches
FOR EACH ROW EXECUTE PROCEDURE update_modified_column();
```
After that SQL is in place, swap both sort lines to `updated_at`. Do not swap until the column exists.

---

## PHASE 4 — Session Continuity
> Quality-of-life. Fix after Phases 1–3 are stable. These are the "where did I leave off" problems.

---

### BUG P1 — No warning when navigating away from editor with unsaved shapes
**File:** `editor.html`
**Severity:** 🟡 MEDIUM — silent data loss path

**What breaks:**
Shapes drawn in the editor are NOT auto-saved. They stay as in-memory objects with no DB ID until the user manually presses SAVE. If the user clicks any nav link (or closes the tab) without saving, all unsaved shapes are gone. There's no `beforeunload` warning, no dirty-state indicator, no prompt.

**Context:** The VOD page already has this pattern — `onMatchChange` warns about unsaved annotations before switching. Editor has no equivalent for unsaved shapes.

**Fix (two parts):**
1. Track dirty state: after any shape is added/removed, set a flag `let _hasUnsavedShapes = false`. Clear it in `manualSave()` on success.
2. Add a `beforeunload` listener:
```js
window.addEventListener('beforeunload', e => {
  const hasLocal = Object.values(shapesCache).flat().some(s => !s.id || String(s.id).startsWith('local_'));
  if (hasLocal) {
    e.preventDefault();
    e.returnValue = 'You have unsaved drawings. Leave anyway?';
  }
});
```
3. Also wire the same check to the nav links (intercept clicks on `.nav-link` and show a confirm dialog before navigating).

---

### BUG P2 — Editor timeline position resets to 0 on every page load
**File:** `editor.html`
**Severity:** 🟢 LOW — forces user to re-scrub to where they were

**What breaks:**
`state.time` initialises to `0` on load. There's no persistence of the last timeline scrubber position. Every session starts from the beginning of the match timeline. If a user was reviewing shapes at the 20-minute mark, they have to manually scrub back there every time.

**Fix:** Save `state.time` to sessionStorage when it changes (on scrubber input), restore it in `_activateMatch`:
```js
// On scrubber change:
sessionStorage.setItem('mz_last_time_' + state.matchId, state.time);

// In _activateMatch, after shapes load:
const lastTime = parseInt(sessionStorage.getItem('mz_last_time_' + match.id) || '0');
state.time = lastTime;
const slider = document.getElementById('temporal-slider');
if (slider) slider.value = lastTime;
draw();
```

---

### BUG P3 — VOD review position never persisted
**File:** `vod.html`
**Severity:** 🟢 LOW — forces full re-sync on every return

**What breaks:**
There's no record of where in the video the user was last reviewing. On return, `vod_start_offset` is restored (the match start point in the video), but not where they were reviewing up to. If they got through 8 minutes of a 20-minute match, they have to re-seek there manually next session. `state.time` starts at 0 on every load.

**Note:** This is lower priority than P2 because the skip banner already jumps to `vod_start_offset` (the match start), which is the most useful jump point. A "resume at last review point" would be an improvement but is not a breaking issue.

**Fix:** Similar to P2 — save `matchTime` (or the video timestamp) to sessionStorage or to the `matches` table as `last_review_position`. On load, if the skip banner is shown, also offer "Resume at 18:24" as a second option.

---

### BUG P4 — Map camera (zoom/pan) never saved or restored
**Files:** `editor.html` AND `vod.html`
**Severity:** 🟢 LOW — minor friction

**What breaks:**
Both pages initialise `zoom = 1, panX = 0, panY = 0` on every load, then immediately run `resetView()` which fits the whole map to canvas. There's no memory of where the user was zoomed/panned to. No "fit to last activity" or "center on most recent shape" logic exists in either file.

**Fix:** Save and restore `{ zoom, panX, panY }` to sessionStorage per match, similar to P2. On match load, restore the saved camera position if it exists, otherwise call `resetView()` as the default.

---

## MISSING FEATURE (not a bug, no data loss — implement when bugs are done)

### FEAT — Self Elim has no fight location pin in editor
**File:** `editor.html`
**Function:** `confirmSelfElim`

**Context:**
- SELF LOSS → has a death pin prompt (saves `death_x`/`death_y` to `match_players`) ✓
- Wipe → has a wipe location pin prompt (saves to `teams`) ✓
- **SELF ELIM → no pin at all** — fight/kill location never recorded in editor

The VOD page works around this via `pendingEventCoords` (map click before pressing button → saves a `fight`-type shape to `shapesCache`). The editor has the pin prompt system but it's never hooked up to self elim. Analytically, knowing WHERE kills happened is just as important as knowing where deaths happened.

**Implementation:** After `confirmSelfElim` records the kill, call `openPinPrompt` and on resolution push a `{ type: 'fight', x: wx, y: wy, tag: 'fight', time: state.time }` shape into `shapesCache[state.activeTeamId]`.

---

## PHASE SUMMARY

### Phase 1 — Data Integrity (do first)
| ID | File(s) | Function | Priority |
|----|---------|----------|----------|
| V1+E5 | `vod.html` + `editor.html` | `confirmEnemyWipe` / `confirmWipe` | 🔴 |
| E1 | `editor.html` | `manualSave` | 🔴 |
| E9 | `editor.html` | `deleteSelected` | 🔴 |
| E4 | `editor.html` | `confirmWipe` Scenario A | 🟡 |

### Phase 2 — Editor Logic (do second)
| ID | File | Function | Priority |
|----|------|----------|----------|
| E2 | `editor.html` | `confirmWipe` (x3 scenarios) | 🔴 |
| E3 | `editor.html` | `confirmWipe` (auto-fire guard) | 🔴 — resolved by E2 fix |
| E8 | `editor.html` | `confirmSelfLoss` | 🟡 |
| E6 | `editor.html` | `startRename` | 🟡 |
| E7 | `editor.html` | `markSelfLoss` | 🟢 |

### Phase 3 — Cross-Page Navigation (do third)
| ID | File(s) | Function | Priority |
|----|---------|----------|----------|
| X1 | `editor.html` | `_activateMatch` | 🔴 |
| X2 | `vod.html` | `onMatchChange` | 🔴 |
| X3 | both | `applyUrlParams` | 🟡 — needs DB migration first |

### Phase 4 — Session Continuity (do last)
| ID | File(s) | Function | Priority |
|----|---------|----------|----------|
| P1 | `editor.html` | `beforeunload` + nav intercepts | 🟡 |
| P2 | `editor.html` | `_activateMatch` + scrubber handler | 🟢 |
| P3 | `vod.html` | `applyUrlParams` + skip banner | 🟢 |
| P4 | both | `resetView` / match load | 🟢 |

---

## QUICK REFERENCE — Files to touch per phase

```
Phase 1:  editor.html  (manualSave, deleteSelected, confirmWipe)
          vod.html     (confirmEnemyWipe)

Phase 2:  editor.html  (confirmWipe, confirmSelfLoss, startRename, markSelfLoss)

Phase 3:  editor.html  (_activateMatch — add mzCtxWrite() call)
          vod.html     (add mzCtxWrite function, call from onMatchChange)
          [SQL]        (updated_at column + trigger — prerequisite for X3)

Phase 4:  editor.html  (beforeunload, _activateMatch, scrubber handler)
          vod.html     (applyUrlParams, skip banner, onMatchChange)
```

---

*All bugs verified against code before inclusion. No speculative entries.*
