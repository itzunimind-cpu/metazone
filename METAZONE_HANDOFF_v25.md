# METAZONE — Project Handoff & Continuity Document
**Version:** v41 — Session 41. vod.html: full appearance overhaul into a floating-chrome "read-only analysis session" tool — left floating vertical toolbar (with new Arrow draw mode), top-right floating squad panel (collapsible Enemy Teams, popover Log Event), read-only canvas context chip replacing the old breadcrumb bar, new Load Match modal, rebuilt bottom bar (skip ±10s, speed popover, fullscreen), End Match confirmation modal. REFRESH DATA / OPEN IN EDITOR dropped entirely. Not yet visually verified in a real browser — see Section 8 Session 41 entry for the caveat.
**Purpose:** Single source of truth across all chat sessions. Read this first in every new chat.

---

## HOW TO USE THIS DOCUMENT

- Read it at the start of every new chat session
- After any session where something is decided or built, update the relevant section
- **NEVER rename a function, DB column, or HTML id that already exists — check Section 6 first**
- **NEVER change existing DB column names or primary key structures**
- When a new function is agreed on, add it to Section 6 before closing that chat
- When a DB change is made in Supabase, move it from Section 3 (planned) to Section 2 (current)
- Keep Section 7 (build order) updated as pages get completed

---

## SECTION 0 — FILE VERSIONS

| File | Version | Last changed | Notes |
|------|---------|--------------|-------|
| index.html | v9.7 | Session 40 | Added delete-match action to dashboard match card as a top-right corner icon overlay on the map image (not a footer button) — cascade-delete reuses existing delete-warning modal pattern. See Section 8 changelog. |
| editor.html | v7.0 | Session 40 | Fixed openInVOD() URL param names (tournament/day/match → tournamentId/dayId/matchId) — was silently broken, masked by vod.html's now-removed dropdown fallback. See Section 8 changelog. |
| tournament-create.html | v5.0 | Session 38 | Full parchment design-system migration — header/nav/auth/sidebar match editor.html (268px). Glow effects, decorative sync-badge, and kernel/copyright footer removed. See Section 8 changelog. |
| tournament-editor.html | v1.4 | Session 25 | VOD REVIEW nav link added |
| analytics.html | v2.3 | Session 25 | VOD REVIEW nav link added, active link fixed |
| player.html | v1.2 | Session 25 | VOD REVIEW nav link added |
| vod.html | v6.0 | Session 41 | Full appearance overhaul — floating left toolbar (+ Arrow draw mode), floating top-right squad panel, read-only canvas context chip, Load Match modal, rebuilt bottom bar, End Match confirmation modal. See Section 8 changelog. |
| history.html | — | NOT BUILT | Design locked — see Section 7B |

---

## SECTION 1 — CURRENT DB STATE

### tournaments
```
id                  uuid PRIMARY KEY
name                text NOT NULL
user_id             uuid REFERENCES auth.users(id)
team_mode           text DEFAULT 'pool'
kill_point_value    numeric DEFAULT 1.0
event_id            uuid REFERENCES events(id)   ← nullable
stage               text                          ← nullable
cycle               int                           ← nullable, group_stage only
created_at          timestamptz DEFAULT now()
```

**stage values (hardcoded, never from DB):**
`qualifiers` | `group_stage` | `last_chance` | `semi_finals` | `finals`

**cycle values:** 1 | 2 | 3 | 4 — only when stage = `group_stage`. Null otherwise.

### days
```
id                  uuid PRIMARY KEY
tournament_id       uuid REFERENCES tournaments(id)
name                text NOT NULL
order_index         int
created_at          timestamptz DEFAULT now()
```

### matches
```
id                  uuid PRIMARY KEY
day_id              uuid REFERENCES days(id)
name                text NOT NULL
map_key             text
duration_seconds    int
vod_url             text       ← Session 21 SQL (run it if not done — see Section 9)
vod_start_offset    int        ← Session 21 SQL (run it if not done — see Section 9)
created_at          timestamptz DEFAULT now()
```

### teams
```
id                  uuid PRIMARY KEY
match_id            uuid REFERENCES matches(id)
name                text NOT NULL
color               text
placement           int
kills               int DEFAULT 0
is_self             boolean DEFAULT false
time_of_wipe        int        ← nullable, match-relative seconds
confirmed_position  int        ← nullable
wipe_x              float      ← nullable
wipe_y              float      ← nullable
created_at          timestamptz DEFAULT now()
```

### match_players
```
id                  uuid PRIMARY KEY
match_id            uuid REFERENCES matches(id)
team_id             uuid REFERENCES teams(id)
player_id           uuid REFERENCES players(id)
kills               int DEFAULT 0
damage              int DEFAULT 0
time_of_loss        int        ← nullable, match-relative seconds
death_x             float      ← nullable
death_y             float      ← nullable
created_at          timestamptz DEFAULT now()
```

### shapes
```
id                  uuid PRIMARY KEY
team_id             uuid REFERENCES teams(id)
match_id            uuid REFERENCES matches(id)
type                text        ← 'line'|'circle'|'arrow'|'pin'|'path'
tag                 text        ← 'zone'|'fight'|'rotation'|'pin'
time_seconds        int
path_points         jsonb       ← [{x,y,t}] for paths
cx,cy,r             float       ← circles
x1,y1,x2,y2        float       ← lines/arrows
wx,wy               float       ← pins
zone_phase          int         ← zone circles only
rotation_index      int         ← rotation lines/paths only
created_at          timestamptz DEFAULT now()
```

### players
```
id                  uuid PRIMARY KEY
tournament_id       uuid REFERENCES tournaments(id)
name                text NOT NULL
ign                 text
role                text
created_at          timestamptz DEFAULT now()
```

### events
```
id                  uuid PRIMARY KEY
name                text NOT NULL
user_id             uuid REFERENCES auth.users(id)
created_at          timestamptz DEFAULT now()
```

---

## SECTION 2 — ARCHITECTURE RULES (HARD CONSTRAINTS)

1. **Vanilla HTML/CSS/JS only** — no frameworks, no build tools, no npm
2. **Single-file pages** — all CSS and JS inline in the HTML file
3. **Supabase active project:** `qsiqgtdgpcyishtayqhy.supabase.co` — NEVER use the old URL
4. **Auth:** Magic link only. `onAuthStateChange` always uses `_booted` boolean guard
5. **Loading screen:** Hide before `getSession()`, not inside `boot()`
6. **Never mix** `class="hidden"` with inline `style="display:X"` on same element
7. **DB IDs:** Never assume local IDs — always check `String(id).startsWith('local_')` before DB writes
8. **Map images:** Use `_lo.jpg` / `_mid.jpg` / `_hi.jpg` — NOT the original `_High_Res.png` files
9. **Navigation:** All 8 nav links must be present on every page, correct `active` class on current page

---

## SECTION 3 — PENDING SQL (run in Supabase before dependent features)

```sql
-- Session 21 — VOD columns (HIGH PRIORITY — VOD page depends on these)
ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_url text;
ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_start_offset int;
```

⚠️ If these haven't been run, the VOD page will silently fail to save/restore offsets and URLs.

---

## SECTION 4 — NAVIGATION (CANONICAL — ALL PAGES)

Every page must have exactly these 8 links in this order, with the current page marked `active`:

```html
<a href="index.html" class="nav-link">DASHBOARD</a>
<a href="editor.html" class="nav-link">EDITOR</a>
<a href="vod.html" class="nav-link">VOD REVIEW</a>
<a href="tournament-create.html" class="nav-link">NEW TOURNAMENT</a>
<a href="tournament-editor.html" class="nav-link">TOURNAMENT EDITOR</a>
<a href="analytics.html" class="nav-link">TEAM ANALYTICS</a>
<a href="player.html" class="nav-link">PLAYER ANALYTICS</a>
<a href="history.html" class="nav-link">ALL TOURNAMENTS</a>
```

Session 25 fixed: VOD REVIEW was missing from analytics, player, tournament-create, tournament-editor. analytics.html had `player.html` incorrectly marked as active.

---

## SECTION 5 — MAP ASSET SYSTEM

| map_key | _lo.jpg | _mid.jpg | _hi.jpg | Original PNG |
|---------|---------|----------|---------|--------------|
| erangel | erangel_lo.jpg | erangel_mid.jpg | erangel_hi.jpg | Erangel_Main_No_Text_High_Res.png |
| miramar | miramar_lo.jpg | miramar_mid.jpg | miramar_hi.jpg | Miramar_Main_No_Text_High_Res.png |
| sanhok  | sanhok_lo.jpg  | sanhok_mid.jpg  | sanhok_hi.jpg  | Sanhok_Main_No_Text_High_Res.png  |
| rondo   | rondo_lo.jpg   | rondo_mid.jpg   | rondo_hi.jpg   | Rondo_Main_No_Text_High_Res.png   |

**Rules:**
- `_lo` = used for thumbnails, card previews, low zoom (< 0.25)
- `_mid` = mid zoom (0.25–1.0)
- `_hi` = full zoom (≥ 1.0)
- index.html `MAP_IMGS` must point to `_lo.jpg` files — NOT the original PNGs
- index.html MAP_IMGS fix was identified this session but not applied (file not uploaded). See Section 9.

---

## SECTION 6 — FUNCTION REGISTRY

### editor.html

| Function | Purpose |
|----------|---------|
| `boot()` | Auth + data load entry point |
| `loadAllData()` | Fetches all tournaments/days/matches/teams/players |
| `_activateMatch(match, statusMsg)` | Loads shapes, matchPlayers, renders everything for selected match |
| `refreshDropdowns()` | Populates all selectors; calls `_flashSelect()` on each that gets a non-empty value |
| `_flashSelect(el)` | Brief opacity fade-in on a `<select>` after populate — resets animation via reflow trick |
| `renderTeamList()` | Redraws sidebar team list; stagger-animates rows (`.animate-in`) only when container was empty |
| `draw()` | Main canvas render loop — also renders wipe pins (teams.wipe_x/y) and death pins (match_players.death_x/y) |
| `drawZoneCircle(s, isPreview)` | Forest stroke-only zone ring — no fill, no label. Color hardcoded to `rgba(44,74,46,op)`. |
| `drawArrow(x1,y1,x2,y2,stroke,lw)` | Solid line + hollow V-path arrowhead (`lineCap:'round'`). Used for rotation shapes. |
| `drawPath(points,stroke,lw,isPreview)` | Solid line + `drawMiniArrow` at each segment midpoint — no endpoint dots, no trailing arrow. |
| `drawMiniArrow(mx,my,angle,color)` | Zoom-aware filled triangle directional marker. Size `6/zoom`. Used by `drawPath`. |
| `drawPinMarker(s,isPreview,teamColor,alphaMult)` | Diamond pin marker. Branches on `s.tag`: `'fight'` → orange diamond + white X; `'pin'` → orange diamond + white X + `>X<` inward chevrons (loss marker). Respects `colorMode` for diamond fill. |
| `drawWipePin(x,y)` | Forest diamond + white circle (top) + white X (bottom) + stem + ground dot. Called from `draw()` for each `team.wipe_x/wipe_y`. |
| `drawDeathPin(x,y)` | Orange diamond + white X + `>X<` chevrons + stem + ground dot. Called from `draw()` for each `matchPlayers[].death_x/death_y`. |
| `mpx(base)` | World-space marker size helper. Returns `base * 0.075` — pure world units so markers scale 1:1 with map zoom. All marker fill sizes AND stroke widths use this. No `/zoom` corrections in marker functions. |
| `clampPan()` | Constrains `panX`/`panY` so map never leaves canvas. When map fits canvas (zoomed out): centers it. When map is larger (zoomed in): clamps to map edges. Called after every zoom and pan operation. |
| `zoomToWorld(wx, wy)` | Centers the canvas on world coords `(wx, wy)` at 20× the fit-to-canvas zoom level. Called when clicking event log entries that have marker coordinates. |
| `updateRedoBtn()` | Syncs redo button `disabled` state and opacity to `(redoStack[state.activeTeamId]||[]).length`. Called after every undo/redo. |
| `eraseAtPosition(wx, wy)` | Hit-test based erase. Pins/fights: radius `12/zoom`. Paths: any point within `10/zoom`. Lines: endpoints or midpoint within `10/zoom`. Zones: `Math.abs(dist-r)<10/zoom` (circumference only). For DB shapes calls `sb.from('shapes').delete()`. |
| `loadMapKey(key)` | Loads map at correct resolution. **Never add `map-ready` class here** — gated on `_canvasReady` flag so canvas never reveals before `initCanvas()` sizes it |
| `initCanvas()` | Sizes canvas, sets `_canvasReady=true`. If `mapImg` already loaded (from `_activateMatch` before init) → `resetView()+draw()+map-ready` immediately. Else calls `loadMapKey()`. |
| `setMapDisplayMode(hasMatch)` | Syncs `#sel-map` value to `state.mapKey` only. Map section is always a `sel-del-row` — no visibility toggling. |
| `renderTimelineTicks()` | Renders event tick marks on the timeline track; calls `renderTimelineLabels()` at end |
| `renderTimelineLabels(maxTime)` | Generates absolute-positioned 5-min interval labels in `#timeline-labels` from 0 to maxTime |
| `updateAnnotationLogLit()` | **Scrub-path only** — toggles `.future` class on existing entries by reading `data-ev-time`. Zero DOM creation. Never call this when event list actually changes. |
| `renderAnnotationLog(opts)` | Full DOM rebuild of event log. Call when events change (match load, new annotation). Stagger-animates entries if container was empty. |
| `updateAnnotationLogColors()` | Loops over entries with `data-team-color`, sets/clears icon background to team color. Called by `toggleMultiTrack()` instead of `renderAnnotationLog()` — no DOM rebuild on toggle. |
| `renderMultiTrackPanel()` | Builds multi-track team checklist with stagger animation; called when multi-track is activated |
| `toggleMultiTrack()` | Toggles multi-track mode; calls `updateAnnotationLogColors()` (not `renderAnnotationLog`) for smooth icon fade |
| `renderActiveSquadInline()` | Renders match_players inline below ACTIVE SQUAD button in right panel. Each row has a hover-reveal `⇄` swap button. |
| `openSqReplace(rowId)` | Toggles inline replacement picker for a squad row. Fetches tournament roster minus current squad, renders as inline dropdown. `_sqReplaceOpen` tracks the open row id; `_sqReplaceAvail` holds fetched candidates (null = loading). |
| `doReplacePlayer(oldRowId, newPlayerId)` | UPDATEs `match_players` row in place (`player_id` only) — kills/damage/time_of_loss are preserved for the new player. Reloads `matchPlayers` and re-renders sidebar. |
| `toggleMetaCollapse()` | Toggles Match Metadata accordion in right panel |
| `openInVOD()` | Navigates to vod.html with current tournament/day/match as URL params |
| `applyUrlParams()` | Deep-link + auto-load most recent match |
| `mzCtxWrite(extra)` | Writes tournamentId/dayId/matchId to sessionStorage |
| `mzCtxRead()` | Reads context from sessionStorage |
| `addMatch()` | Creates new match |
| `deleteSelected(type)` | Deletes selected item |
| `queueForSave(shape)` | Debounced shape save |
| `manualSave()` | Immediate shape save |
| `markSelfElim()` | Self elim modal |
| `markEnemyElim()` | Enemy elim modal |
| `markSelfLoss()` | Self loss modal |
| `markEnemyWipe()` | Enemy wipe modal |
| `markMatchEnd()` | Sets duration_seconds |
| `undo()` | Removes last shape |
| `setMode(mode)` | Switches drawing tool |
| `exportJSON()` | Exports match data |
| `downloadSnapshot()` | PNG snapshot of canvas |

### vod.html

| Function | Purpose |
|----------|---------|
| `boot()` | Auth + data load entry point |
| `loadAllData()` | Fetches all tournaments/days/matches/teams/players |
| `onMatchChange(val)` | Match selected — loads data, auto-fills URL, auto-loads video, shows dialogs |
| `refreshDropdowns()` | Populates all selectors + status icons in match dropdown |
| `_matchStatusIcon(m)` | Returns `'✓ '`\|`'◐ '`\|`''` for dropdown labels |
| `updateVodMatchStatus()` | Updates toolbar status chip |
| `_postMatchLoadDialogs(m)` | After match load: decides which dialog/banner to show |
| `_showSkipBanner(offsetSeconds)` | Shows cyan "jump to match start" banner |
| `skipToMatchStart()` | Seeks video to saved vod_start_offset |
| `dismissSkipBanner()` | Hides the skip banner |
| `_showEditorDataDialog(m, matchTeams)` | Shows "editor data found" dialog with summary |
| `continueWithEditorData()` | Dismisses dialog, highlights START MATCH button |
| `startFreshVodSession()` | Dismisses dialog, starts fresh without clearing DB data |
| `toggleMatchActive()` | START MATCH / RESET — now guards against overwriting existing offset |
| `_confirmOffsetOverwrite()` | Called after user confirms overwrite dialog |
| `_doStartMatch()` | Actually sets vodStartOffset, saves to DB, starts polling |
| `_extractYTIdSafe(url)` | Safe YouTube ID extraction |
| `_createYTPlayer(videoId)` | Creates postMessage proxy to YT iframe |
| `_onYTMessage(event)` | Handles all postMessages from YT embed |
| `startPolling()` / `stopPolling()` | Match time polling loop |
| `toggleOverlayCanvas()` | Z key — show/hide canvas |
| `togglePlayPause()` | P key / ⏯ button |
| `draw()` | Canvas render |
| `renderSquadPanel()` | Right panel: teams + players |
| `renderTimelineMarkers()` | Event dots on timeline bar |
| `seekToMatchTime(t)` | Seek video to match-relative time |
| `applyUrlParams()` | Deep-link + auto-load most recent match |
| `mzCtxRead()` | Reads context from sessionStorage |
| `loadShapesForMatch(matchId)` | Load shapes for selected match |
| `loadMatchPlayers(matchId)` | Load match_players |
| `loadSelfElims(matchId)` | Load self_elims rows for current match — DB-backed source for event log Elim entries (survives refresh) |
| `toggleCalibration()` | Enter/exit 2-point calibration mode |
| `_calInverse(wx, wy)` | World → corrected world coords for calibrated map |
| `updateMapOffset()` | Reads X/Y/scale toolbar inputs, redraws |
| `resetMapOffset()` | Zeros manual alignment |
| `setSpeed(rate)` | YT playback rate. Session 41: also updates `#btn-speed-pill` label text and closes the speed popover. |
| `setMode(mode)` | Drawing tool mode. Session 41: now also accepts `'rotation'` (Arrow tool). |
| `manualSave()` / `queueForSave(shape)` | Shape persistence |
| `updateMatchContextBar()` | Session 41: retargeted from the removed `#match-context-bar` breadcrumb to the new `#vod-context-chip` (canvas overlay). No longer touches a status sub-chip — `mcb-status`/`mcb-status-dot`/`mcb-status-label` were dropped, not relocated. |
| **`refreshMatchData()` — REMOVED Session 41.** | Was only called from the now-deleted REFRESH DATA button (`#match-context-bar`) and an icon button in the Enemy Teams card header. Both call sites dropped per this session's decision; function deleted as dead code. **Do not re-add a call to this name — it no longer exists.** |
| `openLoadMatchModal()` / `closeLoadMatchModal()` / `lmmLoadVideo()` | Session 41, new. Left toolbar's folder icon opens `#load-match-modal` (pre-fills read-only Tournament/Day/Match/Map). `lmmLoadVideo()` just calls the existing `loadVideo()` then closes the modal — the actual `#vod-url-input` element was relocated into this modal, same id, not duplicated. |
| `toggleLogEventMenu()` / `closeLogEventMenu()` | Session 41, new. Squad panel's "+ LOG EVENT" button toggles `#lep-menu` open/closed (houses the original 4 event buttons — elim/loss/wipe/ewipe — unchanged ids/onclicks). `closeLogEventMenu()` is called from inside `startEvent()` so the popover closes the moment an event type is picked. |
| `toggleEnemyTeams()` | Session 41, new. Toggles `.open` on `#sp-enemy-toggle` + `#sp-enemy-list` — Enemy Teams card is collapsed by default now. `renderSquadPanel()` was extended to update `#sp-enemy-count` (badge) alongside its existing enemy-list render. |
| `skipBack()` / `skipForward()` | Session 41, new. Bottom bar's transport buttons — `seekToMatchTime(state.time ∓ 10)`. |
| `toggleStageFullscreen()` | Session 41, new. Fullscreen API on `#vod-stage` (video + canvas only, not the floating chrome's containing element — chrome is positioned inside `#vod-stage` so it fullscreens too). |
| `toggleSpeedPopover()` / `closeSpeedPopover()` | Session 41, new. Bottom bar's speed pill (`#btn-speed-pill`) toggles `.open` on `#speed-cluster`, now a popover instead of 5 inline buttons. `setSpeed()` calls `closeSpeedPopover()` after applying the rate. |
| `openEndMatchConfirm()` / `confirmEndMatch()` | Session 41, new. Left toolbar's End Match icon no longer calls `markMatchEnd()` directly. `openEndMatchConfirm()` checks `matches.find(...).duration_seconds` — if already set (i.e. this would be the *undo* path), skips the modal and calls `markMatchEnd()` straight through since that's reversible; otherwise shows `#end-match-modal` (`modal-red`). `confirmEndMatch()` closes the modal then calls `markMatchEnd()`. `markMatchEnd()` itself is unchanged. |

---

## SECTION 7A — PAGE BUILD STATUS

| Page | Status | Notes |
|------|--------|-------|
| index.html | ✅ Complete | v9.5 — all sidebar/grid bugs fixed, skeleton loader, card reveal animation |
| editor.html | ✅ Complete | v6.3 — full UX polish pass, event log overhaul, toolbar reorganised |
| tournament-create.html | ✅ Complete | Nav updated |
| tournament-editor.html | ✅ Complete | Nav updated |
| analytics.html | ✅ Complete | Nav updated, active link fixed |
| player.html | ✅ Complete | Nav updated |
| vod.html | ✅ Complete | v6.0 — floating-chrome read-only analysis-session redesign (Session 41) |
| history.html | ❌ Not built | Deferred — needs real data first |

---

## SECTION 7B — HISTORY.HTML DESIGN (locked, not built)

- Cross-tournament match history table
- Filterable by team, map, stage, event
- Sortable columns: date, placement, kills, points
- Export CSV button
- Deferred until 2–3 real tournaments of data exist

---

## SECTION 8 — SESSION CHANGELOG

| Session | Change |
|---------|--------|
| 1–20 | See v24 handoff |
| 21 | vod.html spec locked. vod_url + vod_start_offset planned. |
| 22 | vod.html v1.0 built. index.html v9.0: dual EDITOR + VOD links on cards. |
| 23 | vod.html toolbar redesigned into 2 rows. YT iframe → postMessage proxy. Calibration tool. |
| 24 | Calibration zoom bug fixed. World-space cal values. Wheel blocked during cal. |
| 25 | **Match status chips** — editor sidebar chip: NO DATA YET / IN PROGRESS / DATA COMPLETE + VOD ✓ badge. |
| 25 | **Match status icons** — `✓ ` / `◐ ` prefix in match dropdowns on both editor and VOD. |
| 25 | **VOD auto-load video** — on match select, if vod_url saved, auto-fills URL and loads video immediately. |
| 25 | **Skip to match start banner** — cyan banner when match has saved offset. JUMP button seeks + plays. Handles pending offset if video still loading (_pendingSkipOffset). |
| 25 | **Editor data dialog** — if editor data exists but no VOD offset yet, shows dialog with data summary (teams, wipes, KIAs, duration). CONTINUE or START FRESH. |
| 25 | **Offset overwrite protection** — pressing START MATCH on a match with existing offset shows warning dialog before overwriting. |
| 25 | **VOD toolbar status chip** — NO DATA YET / IN PROGRESS / DATA COMPLETE + VOD ✓ badge. |
| 25 | **Nav standardised** — all 8 links on every page. VOD REVIEW added to analytics, player, tournament-create, tournament-editor. analytics.html active link bug fixed. |
| 25 | **VOD auto-loads most recent match** — applyUrlParams() now async. If no URL params or session context, sorts matches by created_at desc and auto-selects latest. |
| 25 | **index.html MAP_IMGS bug identified** — cards/preview panel broken because MAP_IMGS points to original `_High_Res.png` files. Fix = change to `_lo.jpg`. Not yet applied (file not uploaded this session). |
| 26 | **Git connected** — repo at github.com/itzunimind-cpu/metazone. All local files pushed. Image assets restored after accidental force-push wipe. |
| 26 | **index.html audit** — full review against design handoff + bug handoff. MAP_IMGS confirmed already using `_lo.jpg` (was already fixed). |
| 26 | **index.html: delete day/tournament fixed** — `_idmConfirm()` was nulling `_idmCb` via `closeDeleteWarning()` before calling it. Fixed by capturing callback before close. Affected all delete operations. |
| 26 | **index.html: add day shows immediately** — new day was pushed to `t.days` but DOM not rebuilt (selectDay used `patchTreeSelection` path). Fixed by calling `renderTree()` immediately after push in both `addDay()` and `addDayInline()`. |
| 26 | **index.html: day-switch flicker removed** — shimmer always showed 3 ghost cards regardless of actual count. Replaced with correct-count skeleton using `d.match_count`. Added stale fetch guard (`if(activeDayId!==dayId)return`). |
| 26 | **index.html: match count subtitle** — `selectDay` was setting `main-sub` to empty string. Now shows `d.match_count` immediately, updates with exact count after fetch. |
| 26 | **index.html: skeleton loader** — warm parchment shimmer animation. Left button slot is green-tinted (matching EDITOR button). Add Match card always present during loading. |
| 26 | **index.html: card reveal animation** — staggered `cardReveal` fade+slide-up (0.28s, 60ms stagger) triggered only after load, not on selectMatch re-renders. |
| 26 | **editor.html bug audit** — all 10 bugs from BUGSTOFIX verified in code. None fixed yet. See Section 9 for fix order. |
| 28 | **editor.html loading states reverted** — shimmer overlay + canvas opacity fade-in from Sessions 27 removed. Returned to d8002f1 (post-UI-overhaul, pre-loading-states). No loading indicators on editor page. |
| 28 | **editor.html sidebar scrollbar stability** — `#sidebar` changed from `overflow-y:auto` to `overflow-y:scroll`. Scrollbar lane always reserved so dropdown/input width never changes when teams load and make the list scrollable. |
| 28 | **editor.html right panel pre-populated empty states** — `#active-squad-inline` and `#annotation-log` now ship with their empty state HTML in the markup (`No squad registered…` and `NO EVENTS YET`). `renderActiveSquadInline()` and `renderAnnotationLog()` overwrite on first call. Right panel height is stable before any data loads — no layout shift. |
| 29 | **editor.html team row stagger animation** — `.animate-in` keyframe (opacity 0→1, translateY 5px→0, 0.22s, 45ms stagger). Only fires when `container.children.length === 0` before render — re-renders from switchActiveTeam/toggleOverlay are instant. |
| 29 | **editor.html select fade-in** — `_flashSelect(el)` helper. `refreshDropdowns()` calls it on each select that receives a non-empty value. `sel-pop` keyframe fades from 0.6→1 opacity over 0.45s ease-out. Starts at 0.6 (not 0) to avoid flicker. |
| 29 | **editor.html map zoom root cause fixed** — `_canvasReady` flag (default false). `loadMapKey()` onload only adds `canvas.map-ready` when `_canvasReady` is true. `initCanvas()` sets flag after `resizeCanvas()`, then: if `mapImg` already loaded (from `_activateMatch` running before init) → `resetView()+draw()+map-ready` immediately with correct dims. Else → calls `loadMapKey()`. Canvas stays invisible until dims are guaranteed correct. |
| 29 | **editor.html map section simplified** — removed two-state toggle (select vs display-row). Map always shows as `sel-del-row` with select + `✕` button (`.btn-del`), matching tournament/day/match pattern. `map-display-row`, `map-display-name` elements removed. `setMapDisplayMode()` reduced to one line: syncs select value only. |
| 30 | **editor.html multi-track moved to left sidebar** — MULTI TRACK button+panel moved from right panel to left sidebar, between Shape Colors and Teams sections. Same IDs, no JS changes. |
| 30 | **editor.html shape color active state** — `.btn-cm.active` changed from barely-tinted outline to solid `var(--forest)` fill with white text. Clear selected vs unselected. |
| 30 | **editor.html timeline labels** — hardcoded flex labels (`08:45 / 17:30 / 26:15`) removed. `renderTimelineLabels(maxTime)` generates absolute-positioned labels at every 5 min interval. Called from `renderTimelineTicks()`. First and last labels are edge-aligned to avoid clipping. |
| 30 | **editor.html event log scrub fix** — scrubbing calls `updateAnnotationLogLit()` (class toggle only, zero DOM creation) instead of `renderAnnotationLog()`. Full rebuild only on actual event changes. |
| 30 | **editor.html event log animation** — `renderAnnotationLog()` stagger-animates entries (30ms per entry) when container was empty before call (match load / match switch). |
| 30 | **editor.html time display stability** — `#time-display` gets `font-variant-numeric:tabular-nums; width:5ch` — all time values render identically wide, no layout shift while scrubbing. |
| 30 | **editor.html scrubber sweeps to match end on load** — `_activateMatch()` and `sel-match.onchange` both animate fill+thumb from 0% to match-end position over 1s (cubic-bezier) using inline transitions that are cleaned up after. `state.time` is set immediately for correctness; only the visuals animate. |
| 30 | **editor.html MATCH END to toolbar** — moved from bottom bar to top toolbar as `btn-tool-action btn-match-end-toolbar` with `lucide:square` icon. Red hover to signal finalising action. Bottom bar retains 4 CTA buttons: SELF ELIM (`lucide:zap`), SELF LOSS (`lucide:skull`), SELF WIPE (`lucide:shield-x`), ENEMY WIPE (`lucide:flag`). |
| 30 | **editor.html zoom floor** — `zoomAt()` minimum changed from hardcoded `0.05` to `Math.min(canvas.width/WORLD_W, canvas.height/WORLD_H)` — map can never be zoomed smaller than fit-to-canvas. |
| 30 | **editor.html sidebar width parity** — left sidebar width changed from 240px to 268px to match index.html dashboard. No layout shift on page transitions. |
| 30 | **editor.html event log icon flicker fix** — `will-change:opacity` on `.annotation-entry` pre-allocates GPU compositing layer so Iconify shadow DOM doesn't repaint on opacity transition. `transform:translateZ(0)` on `.annotation-icon` isolates icon further. |
| 30 | **editor.html event log time readability** — title and time moved into a flex `.annotation-entry-header` row. Time is right-aligned, 9px mono tabular-nums, muted — readable without competing with title. |
| 30 | **editor.html multi-track team list animation** — `renderMultiTrackPanel()` applies `animate-in` + 40ms stagger per row on every open. |
| 30 | **editor.html multi-track color on icon** — dot prepended to title text replaced by icon background color. Team color stored as `data-team-color` on each entry. `updateAnnotationLogColors()` sets/clears icon styles in-place without DOM rebuild. `toggleMultiTrack()` calls this instead of `renderAnnotationLog()`. CSS transition on `.annotation-icon` (background/border-color/color 0.25s) gives smooth fade. |
| 27 | **editor.html emoji purge** — all emoji removed across entire file (HTML, JS strings, toast messages, modal titles, annotation log event data, multi-track stats). Annotation log icons converted from `textContent` emoji to `innerHTML` Iconify SVG strings. |
| 27 | **editor.html left sidebar refactor** — Screenshot + Clear Team buttons moved from sidebar to top toolbar as `.btn-tool-action` buttons. MAP button removed from toolbar. OPEN IN VOD button added to sidebar below Map section (`.btn-open-vod` style, orange border, navigates via `openInVOD()`). |
| 27 | **editor.html match status chip removed** — `#match-status-chip` HTML, its CSS, `updateMatchStatusChip()`, and `_matchStatusLevel()` all deleted. Status symbols (`✓`, `◐`) removed from match dropdown labels in `refreshDropdowns()`. |
| 27 | **editor.html right panel: Competing Factions removed** — `#competing-factions` section HTML and `updateFactionBadges()` + its call site in `updateRightPanel()` all deleted. |
| 27 | **editor.html right panel: Match Metadata collapsible** — section wrapped in `.rp-collapse-header` button + `.rp-collapse-body` (hidden by default). `toggleMetaCollapse()` toggles body + rotates chevron icon. |
| 27 | **editor.html right panel: Active squad inline** — `#active-squad-inline` div renders player rows below ACTIVE SQUAD button via `renderActiveSquadInline()`. Shows `.sq-dead` opacity for players with `time_of_loss`. Empty note when no players registered. |
| 27 | **editor.html right panel: Add Note button prominent** — `.btn-add-annotation` now `background:var(--orange)`, full-width, uppercase mono. |
| 27 | **editor.html visibility filter buttons** — `.btn-filter.on` states updated: solid background fill per filter type (accent/red/green/orange) replacing the border-only style. |
| 27 | **editor.html loading states removed** — all overlay HTML/CSS/JS deleted: `#map-loading-overlay`, `#canvas-empty-state`, `#match-loading-bar`, `#sidebar-loading`, `showMapLoading()`, `hideMapLoading()`, `setMatchLoading()`, `setCanvasEmptyState()`, `setSidebarLoading()`. All call sites scrubbed from loadAllData, loadShapesForMatch, resetLocalState, _activateMatch, and all 3 dropdown onchange handlers. |
| 27 | **editor.html CSS shimmer** — `#canvas-wrap` now shows animated dark shimmer gradient background while map loads. `canvas` starts at `opacity:0`, gains `map-ready` class in image `onload` to fade in via `transition:opacity .4s`. No text, no spinner, no overlay. |
| 27 | **editor.html map zoom bug fixed** — `loadMapKey()` was called inline in script body before `DOMContentLoaded`. On warm cache the image fired `onload` with `canvas.width=300` (HTML default), then `initCanvas()` resized canvas → double `resetView()` → map appeared to zoom in on reload. Fix: moved `loadMapKey(state.mapKey\|\|'erangel')` call to start of `initCanvas()` after `resizeCanvas()`. Removed inline call entirely. |
| 31 | **editor.html multi-track panel expand/collapse animation** — `#multi-track-panel` switched from `display:none/flex` toggle to `max-height:0→600px` + `opacity:0→1` with `transition:.35s cubic-bezier(.4,0,.2,1)`. Panel now animates open AND closed. Teams section below slides up naturally as panel collapses — no JS needed. |
| 31 | **editor.html event log header height stability** — `#mt-log-filter` (team filter select in annotation header) changed from `display:none/block` to `visibility:hidden/opacity:0` with `.visible` class toggle. Select always occupies space in the flex header so its height never changes when multi-track is toggled. `.annotation-header` gets `min-height:40px` as additional guard. |
| 31 | **editor.html `loadMapKey()` returns a Promise** — resolves when lo-res image loads (or immediately if cached). Enables `_activateMatch()` to await map load in parallel with DB fetches. `custom` path resolves immediately. All existing call sites ignore the return value (backward-compatible). |
| 31 | **editor.html coordinated load reveal** — `_activateMatch()` now runs map image load + all DB fetches (`loadShapesForMatch`, `loadAnnotations`, `loadMatchPlayers`, `preloadKnownTeams`) in a single `Promise.all()`. Everything is ready at the same moment. `initCanvas()` runs right after and sees `mapImg` already loaded → adds `map-ready` immediately. Teams, event log, dropdowns, scrubber, and canvas all appear in the same frame burst. |
| 31 | **editor.html event log fully lit on match open** — root cause: `state.time` was being set inside the scrubber block which ran AFTER `renderAnnotationLog()`. All events appeared dimmed (`.future`) because `state.time` was still 0. Fix: split scrubber into two parts — (1) state setup (`state.time`, `slider.max`, `slider.value`, time display) runs BEFORE `renderAnnotationLog()`, (2) visual fill/thumb sweep animation runs after. Event log now builds with all entries lit when opening a finished match. |
| 31 | **editor.html dropdown pill slide animation** — `sel-pop` keyframe upgraded from `opacity:0.6→1` (subtle fade) to `opacity:0,translateY(5px)→opacity:1,transform:none` — matching `team-row-in` style. Tournament, Day, Match selects animate in via `_flashSelect()` in `refreshDropdowns()`. Map select gets `_flashSelect()` explicitly in `_activateMatch()` after data loads. Sidebar itself stays visible throughout (no hiding). |
| 32 | **[E1] manualSave saves all teams** — was only saving `shapesCache[state.activeTeamId]`. Fixed to iterate `Object.keys(shapesCache)` and flush unsaved shapes for every team. Zones (team_id=null) path unchanged. |
| 32 | **[E9] deleteSelected cleans up match_players** — tournament/day/match delete branches all now `await sb.from('match_players').delete().in('match_id', mIds)` (or `.eq` for single match) before deleting shapes/teams/matches. No more orphaned player rows after any delete. |
| 32 | **[V1+E5] confirmWipe winner no longer gets time_of_wipe** — Scenarios A and B were incorrectly setting `winner.time_of_wipe = state.time` on the surviving enemy (the match winner). Removed from both JS local state and DB update in both scenarios. Only `confirmed_position:1` is now written for the winner. |
| 32 | **[E2] markMatchEnd deferred past pin prompt** — all three scenarios (A/B/C) in `confirmWipe` were firing `setTimeout(markMatchEnd, 600)` while the wipe-location pin prompt overlay was still open. Added `_pinPromptOnClose` hook (null by default). `skipPinPrompt()` fires it 300ms after the overlay closes (click or SKIP). Scenarios now set `_pinPromptOnClose = markMatchEnd` when `_pinPromptActive` is true, fall back to the 600ms setTimeout only when no pin prompt was opened (local team path). |
| 32 | **[X1] mzCtxWrite() called on match activate** — function was defined but never called. Added `mzCtxWrite()` at the end of `_activateMatch()` so tournament/day/match IDs are written to sessionStorage on every match load, enabling deep-link context restore when navigating back to the editor. |
| 33 | **Active squad never saved to DB** — root cause: `confirmActivePlayers()` called `closeActivePlayersModal()` before capturing `selectedIds`. `closeActivePlayersModal()` resets `_apmSelectedIds = new Set()`, so `selectedIds` was always `[]`. Zero inserts ever ran. Fix: capture `const selectedIds = [..._apmSelectedIds]` before calling `closeActivePlayersModal()`. |
| 33 | **In-place player swap** — hover any squad row in the right sidebar to reveal a `⇄` icon. Click it → inline dropdown shows all tournament roster players not in the current squad. Selecting one does `UPDATE match_players SET player_id = newId WHERE id = rowId` — kills, damage, and time_of_loss all carry over to the new player. `_sqReplaceOpen` / `_sqReplaceAvail` module vars track picker state; reset in `loadMatchPlayers` on match switch. |
| 33 | **`loadMatchPlayers` role field missing** — step 2 players lookup selects only `id, name` — `role` is never fetched, so `.sq-player-role` span never renders in the sidebar. Low-priority cosmetic fix: change select to `id, name, role` and update the `matchPlayers` map to store `role`. |
| 34 | **Zone marker** — `drawZoneCircle` replaced: stroke-only forest ring (`#2C4A2E`), no fill, no label. `tagRgb('zone')` updated to `[44,74,46]`. |
| 34 | **Rotation marker** — `drawArrow` updated: connected V-path arrowhead (single `beginPath` with two `lineTo` legs), `lineCap:'round'`. |
| 34 | **Movement path** — `drawPath` replaced: solid line + `drawMiniArrow` filled triangle at each segment midpoint. Endpoint dots and trailing arrow removed. |
| 34 | **Fight marker** — `drawPinMarker` (tag:'fight'): orange diamond fill + white X strokes. `tagRgb('fight')` updated to `[200,94,10]` (#C85E0A). |
| 34 | **Loss marker** — `drawPinMarker` (tag:'pin'): orange diamond fill + white X + white `>X<` inward chevrons on all 4 sides. `tagRgb('pin')` updated to `[200,94,10]`. |
| 34 | **Wipe CTA pin rendered on canvas** — `drawWipePin(x,y)` added. `draw()` now iterates match teams and draws forest diamond + white circle (top) + white X (bottom) + stem for any team with `wipe_x/wipe_y` set. |
| 34 | **Death CTA pin rendered on canvas** — `drawDeathPin(x,y)` added. `loadMatchPlayers` now fetches `death_x, death_y` and maps them into `matchPlayers`. `draw()` renders orange diamond + `>X<` + stem for each player with `death_x/death_y` set. |
| 34 | **`drawMiniArrow` helper** — zoom-aware filled triangle (`6/zoom`). Used by `drawPath` at segment midpoints. |
| 35 | **Self loss marker colour unified** — loss marker (`tag:'pin'`) now uses orange (#C85E0A) matching the fight marker. Diamond removed from plain pin marker — only fight/loss markers use the diamond. |
| 35 | **Erase fixed for pre-session DB markers** — old `shapes.pop()` always removed last element with no DB call. Replaced with `eraseAtPosition(wx,wy)`: hit-tests click position, splices correct index, calls `sb.from('shapes').delete()` for any shape with a non-local DB id. |
| 35 | **Redo button dynamic state** — redo button was hardcoded `disabled` in HTML. Added `updateRedoBtn()` which reads `redoStack[state.activeTeamId]` length and sets `disabled`/`opacity`/`cursor`. Called after every undo/redo operation. |
| 35 | **Zone erase circumference-only** — zone hit test changed from `dist < r + 8/zoom` (anywhere inside circle) to `Math.abs(dist - r) < 10/zoom` (only near the circle edge). Prevents accidental zone deletion when clicking inside a large zone. |
| 35 | **Markers — pure world-space scaling** — `mpx(base)` simplified to `return base * 0.075`. Canvas is pre-scaled by `ctx.scale(zoom,zoom)` so markers scale exactly 1:1 with map zoom. No floor/cap clamping. All stroke `lineWidth` values also use `mpx()` (previously used `/zoom` = constant screen pixels which made markers look bloated when zoomed out). |
| 35 | **Markers — fully opaque** — all `rgba()` fill/stroke colors in `drawPinMarker`, `drawWipePin`, `drawDeathPin` replaced with `rgb()`. Opacity slider no longer affects marker appearance. |
| 35 | **Pan clamping** — added `clampPan()`. Map can never be dragged outside the canvas boundary. When map fits canvas (zoom at minimum): always centered. When zoomed in: pan constrained to map edges. Called from `zoomAt()`, pan `onMove`, and pinch-zoom handler. |
| 35 | **Zoom HUD normalized** — `updateZoomHud()` now displays `(zoom / minZoom).toFixed(2)+'×'` so 1.00× always means map fits canvas, regardless of canvas/world size ratio. |
| 35 | **Toast redesign — design handoff palette** — toasts rebuilt to match the parchment design language: `#F2EDE4` background, `#C9BFA8` warm border, 3px left accent bar per type (forest/orange/red/olive), Lexend Deca 8px ALL CAPS type label in accent colour, Manrope 11px `#1C1A14` message body, warm box-shadow, slide-in from right animation. |
| 35 | **Event log — zoom to marker on click** — `zoomToWorld(wx, wy)` added. All clickable event log entries now call `jumpToTime` + `zoomToWorld` together. Shape events (pin, fight, rotation line/path, zone circle) pass their world coordinates into the event object at render time. Wipe events pass `team.wipe_x / team.wipe_y`. Zoom level: 20× the fit-to-canvas minimum. |
| 35 | **Event log — wipe hint removed** — the overlapping inline hint text ("↩ Press SELF WIPE to undo") that appeared on wipe entry click replaced by the same jump+zoom behavior as all other entries. |
| 36 | **Event log pill redesign** — every entry click now uniformly calls `jumpToTime` + `zoomToWorld` (if it has `wx/wy`), including Self Loss (resolves its death pin via `matchPlayers`). `annotation-entry-header` row removed — title and time now stack vertically in `annotation-body` (time below title, not beside it). |
| 36 | **Event log — persistent undo button** — undoable entries (Loss/Elim/MatchEnd/Wipe/Zone/Fight/Pin/Rotation) now show a small `↩` icon button anchored to the right of the pill (`.annotation-undo-btn`), replacing the old click-to-reveal text button. Undo extended beyond Loss/Elim/MatchEnd to Wipe (`_team` ref), Zone circles (`_zone` ref), and Fight/Pin/Rotation shapes (`_shape` ref) — `undoLogEvent()` removes the underlying record from local state + DB and redraws, mirroring the existing `eraseAtPosition` delete pattern. |
| 36 | **Self Loss/Elim log entries fixed to survive refresh** — root cause: these entries were built from ephemeral, in-memory-only annotations (`local_loss_*` / `local_elim_*`) that were never persisted and got wiped every match reload, even though the underlying data (`match_players.time_of_loss/death_x/y`, `kills`, `self_elims` rows) was saved fine. Fix: Loss entries now built directly from `matchPlayers` (`time_of_loss!=null`); Elim entries built from a new `selfElims` cache loaded via `loadSelfElims(matchId)` (same pattern as wipe events sourced from `teams.time_of_wipe`). Called in `_activateMatch`'s `Promise.all` and the `sel-match` onchange handler; reset alongside `annotations=[]` in all reset/delete paths. |
| 36 | **Active shape-color button hover fix** — `.btn-cm:hover` was overriding text color to accent even when `.active` (same CSS specificity, later in source order), making the green-filled active button's text flash on hover. Added `.btn-cm.active:hover` to keep border/text colour consistent with the active state. |
| 37 | **vod.html — full parchment design-system migration.** vod.html was still on the original Phase 1 dark-navy/cyan system (`--bg:#020617`, `--accent:#22d3ee`, JetBrains Mono/Satoshi, raw emoji icons) — it was never migrated when editor.html/index.html moved to the parchment system. `:root`/fonts swapped to editor.html's exact palette; `#app-header`/nav/auth-screen replaced with the canonical 52px-forest header (verbatim from editor.html); toast CSS+JS replaced with editor's parchment toast (accent-bar/label/body); `#squad-panel` resized 220px→280px to match editor's `#right-panel`; all modals (elim/loss/wipe/ewipe/next-match/active-players/results-panel/setup-flow/offset-overwrite) restyled to the parchment modal pattern; toolbar (`#vod-toolbar` rows 1+2) and mode-pill active colors remapped to forest/olive/orange/red. |
| 37 | **vod.html — emoji sweep.** All ~25 raw emoji (▶⭕→📍✕↩👁💾⏯⏸🎯⚡💀☠⚑⏹👥📹🔗📋⏱⬡⚠) replaced with `<iconify-icon icon="lucide:*">` across static HTML and JS-built strings (`EVENT_CONFIG`, `addEventLog` calls, dynamic button text via `innerHTML`). Toast messages had emoji stripped entirely since `toast()` renders plain text. `✓`/`◐` status glyphs kept — matches existing convention from editor.html match-dropdown status icons. |
| 37 | **vod.html — canvas/timeline marker recolor.** Hardcoded marker hex constants in `drawZoneCircle`/`renderTimelineMarkers`/`addTlMarker`/calibration-loupe crosshair remapped from old cyan/red/green (`#22d3ee`/`#f87171`/`#4ade80`) to new semantic equivalents (forest `#2C4A2E`, red `#9B1C11`, orange `#C85E0A`). Calibration dots/loupe crosshair/cal-banner kept their dark HUD-chip backgrounds (mirrors editor.html's own `#zoom-hud`/`#pin-prompt-banner` precedent of staying dark over canvas/video) but use a bright `#7BC47F` green instead of dark forest for the "map point" indicator, since the brand forest green is too dark to read against near-black video overlays. `TEAM_COLORS` swatch array and canvas `ctx.font` family left untouched — out of scope for a chrome-only design pass. |
| 38 | **tournament-create.html — full parchment design-system migration.** Was still on the original Phase 1 dark-navy/cyan system, never migrated. `:root`/fonts swapped to editor.html's exact palette; removed unused `<script src="cdn.tailwindcss.com">` (confirmed zero Tailwind utility classes used anywhere in the file — the `grid-bg`/`stage-grid`/`mode-grid` matches were all custom classes); removed the cyan grid-line `body.grid-bg` background (Phase-1 "grid lines" anti-pattern). `#app-header` replaced with the canonical 52px-forest header verbatim from editor.html (was 56px translucent/blurred dark header); `#tc-sidebar` resized 280px→268px to match editor.html/index.html's left sidebar (this page's left sidebar is the step-tracker, analogous to editor/index's left nav sidebar, not a right panel). Auth screen rebuilt to match index.html's orange-diamond logo-icon pattern. |
| 38 | **tournament-create.html — anti-pattern removal.** Removed the decorative `.sync-badge` ("● SYSTEM ONLINE", hardcoded, never updated by JS — confirmed no associated update function, unlike editor.html's/vod.html's real sync-dot which are driven by actual save/video-sync state) and the entire `#app-footer` kernel-version/copyright bar (`KERNEL: v5.0.0-ALPHA`) — both are explicitly named in `METAZONE_DESIGN_HANDOFF.md`'s Phase-1 anti-patterns table ("Meaningless status dots", "Footer kernel/latency bar") and neither exists on any other redesigned page. |
| 38 | **tournament-create.html — glow removal.** Per the design handoff's locked rule #4 ("No glow effects — not for hover, not for selected state"), removed every `box-shadow:0 0 Npx rgba(...)` glow: `#progress-fill`, `.mode-btn.selected`, `.event-card.active`, `.btn-confirm` (+hover). `#progress-fill` and `.btn-confirm` (the wizard's primary CTA) recolored to orange per the "primary CTA = orange" rule, replacing the old cyan-glow treatment. All remaining cyan/red rgba tints (mode cards, event card, stage/cycle pickers, chips) remapped to forest/red equivalents. |
| 39 | **vod.html — toolbar reorganized into semantic groups.** The flat 2-row, ~30-control toolbar (`#tb-row1`/`#tb-row2`, now removed) was replaced with five bordered `.toolbar-group` clusters — Match controls, Map alignment, Draw tools, Overlay/playback, Save — separated by `.tb-divider` (restyled to match editor.html's `.toolbar-divider` exactly: 24px height, `var(--border-mid)`). Each group wraps as a unit on narrow viewports instead of controls breaking apart mid-cluster. |
| 39 | **vod.html — squad panel split into cards.** `#squad-panel`'s four sections (Our Squad, Log Event, Enemy Teams, Event Log) now each wrap a `.sp-card` (copied 1:1 from editor.html's `.meta-card`: bg/border/6px radius/10px 12px padding/8px gap), giving the panel the same card rhythm as editor's `#right-panel`. New `.sp-card-header`/`.sp-empty-note` classes replace ad hoc per-section inline header/empty-state markup. |
| 39 | **vod.html — inline styles and onmouseover/onmouseout handlers removed.** Every `style="..."` attribute and the four `onmouseover`/`onmouseout` JS color-swap handlers (SET SQUAD button, enemy-refresh button) were replaced with real CSS classes (`.map-align-input`, `.btn-set-squad`, `.icon-btn-sm`, `.toolbar-group-label`, `.hidden` utility, etc.) with proper `:hover` rules. The hand-rolled inline-styled confirm dialog in `_confirmDialog()` now emits the existing `.modal-box`/`.modal-title`/`.modal-actions`/`.btn-modal-cancel`/`.btn-modal-confirm` classes instead of re-declaring every property inline. `mcb-map`/`mcb-status` visibility toggling switched from `el.style.display` to `classList.toggle('hidden', ...)` to match. |
| 39 | **vod.html — hardcoded colors deduped into shared tokens.** Added `--overlay-dark` (`rgba(2,6,23,.85)`, for dark chrome over the video stage — zoom HUD, calibration banner/loupe, replacing literal `#040c1a` and three different ad hoc alphas), `--selected-tint`/`--hover-tint` (green-accent overlay tokens, replacing repeated literal `rgba(44,74,46,.0x)` across `.modal-player.selected`/`.apm-player-row.sel`/`.vel-entry.lit`/`.sfd-step.active`). Toast colors (`.toast`/`.toast-label`/`.toast-msg`/`.toast.info/.warn/.err/.ok`) switched from literal hex to `var(...)`; `.toast.err` snapped from the off-palette `#A02010` to `var(--red)`. |
| 39 | **vod.html — modal sizing/accent-color standardized.** Replaced the ad hoc `.modal-box.wide` + two one-off width overrides with explicit `.modal-sm`/`.modal-md`/`.modal-lg` (300/360/380px) and replaced every modal's inline `border-color`/title `color` override with `.modal-accent`/`.modal-yellow`/`.modal-red`/`.modal-orange` modifier classes (title color cascades via `.modal-accent .modal-title{color:var(--accent)}` etc., so titles no longer need a second class). Applied across elim/loss/wipe/ewipe/next-match/active-players modals and the JS-built confirm dialog. Spacing also standardized to editor.html's 4/6/8/10/12px rhythm (`.vel-entry`, `#vod-event-log`, `.sp-player`, `.sp-enemy-team`, `.sp-section`, `#vod-toolbar`). |
| 40 | **vod.html — tournament/day/match switching UI removed.** Executive decision: VOD review is being positioned as a focused, premium-feeling tool, and match navigation should live only on the dashboard (`index.html`). Removed the `#sel-tournament`/`#sel-day`/`#sel-match` `<select>` elements from the toolbar's "match" group, along with the now-unreachable `onTournamentChange()`, `onDayChange()`, `refreshDropdowns()`, and `_matchStatusIcon()` functions. There was no separate "map switcher" to remove — `state.mapKey` is already set automatically from the selected match's `map_key` inside `onMatchChange()`. `onMatchChange()` itself, `applyUrlParams()` (URL-param + auto-load-most-recent-match deep linking), `#match-context-bar` (breadcrumb/status/REFRESH DATA/OPEN IN EDITOR), and `loadAllData()`'s underlying data fetch are all unchanged — they never depended on the dropdowns, only used them as a manual fallback. `nextMatchSameVod()`'s auto-advance-to-next-match logic kept, just stripped of its `sel-match` DOM references. **editor.html's own in-page tournament/day/match selectors were deliberately left alone** — they're deeply wired into editor's cascading day/match data-loading and editor also owns a separate `#sel-map` dropdown (sets a match's map before VOD ever reads it), making removal there a bigger, separate decision not made this session. |
| 40 | **editor.html — openInVOD() param-name bug fixed.** Found while auditing vod.html's deep-link dependency: `openInVOD()` was building its URL with params named `tournament`/`day`/`match`, but `vod.html`'s `applyUrlParams()` (and `index.html`'s own vod-links) read `tournamentId`/`dayId`/`matchId`. This was silently masked by vod.html's now-removed dropdown fallback — clicking "OPEN IN VOD" would land on an unfilled page that the user could still manually navigate via dropdowns. With those dropdowns gone, this had to be fixed: the three param keys now match exactly what `applyUrlParams()` expects. |
| 40 | **index.html — delete-match added to dashboard match card.** Noticed gap while auditing the above: the dashboard's `matchCard()` had EDITOR/VOD actions but no delete, even though `editor.html` already had a full working cascade-delete for a match. Added `deleteMatch(matchId)` (~near `deleteDay`/`deleteTournament`) reusing the existing `showDeleteWarning()`/`#idx-delete-modal` confirm-dialog plumbing; cascade order mirrors editor.html's `deleteMatch`: `match_players` → `shapes` (by team_id) → `teams` → `matches` (index.html's own `deleteDay`/`deleteTournament` don't clean up `match_players` — a pre-existing gap, left as-is, out of scope here). Initially placed as a third `.card-foot` button next to EDITOR/VOD, then moved per follow-up request to a `.card-del-corner` icon button absolutely positioned top-right over the map image (24×24px, dark semi-transparent default, hover-to-red, matches the `.card-corner-tab`/overlay treatment already used there) — keeps `.card-foot` to just the two primary actions. Calls `event.stopPropagation()` so it doesn't trigger `selectMatch()`. |
| 41 | **vod.html — full appearance overhaul: read-only analysis-session tool.** Toolbar (`#vod-toolbar`) and full-width breadcrumb (`#match-context-bar`) removed; replaced with floating chrome over the canvas. **Left floating toolbar** (`#vod-left-tools`, vertically centered, left edge): Zone/Path/Pin/Arrow → divider → Erase/Undo → divider → Load Match/Calibrate/Save → divider → Start Match (green)/End Match (red). **Top-right floating panel** (`#squad-panel`, restyled in place, same ids): Map Opacity (moved to top) → Our Squad → Enemy Teams (now collapsible via `toggleEnemyTeams()`, collapsed by default, count badge) → Event Log (4 event buttons now tucked behind a "+ LOG EVENT" popover, `toggleLogEventMenu()`). **Read-only canvas context chip** (`#vod-context-chip`, top-left): Tournament › Day › Match › Map + the existing data-status chip/video-sync indicator/match timer — no status element added to the actual nav bar (`#app-header` untouched). **New Load Match modal** (`#load-match-modal`): houses the VOD URL input (still `#vod-url-input`, same id, just relocated) and the map-alignment fine-tune fields (X/Y/Scale/Reset) that used to be inline in the old toolbar — 2-point calibration click-flow unchanged, just triggered from the toolbar's crosshair icon. **Bottom bar** (`#vod-timeline`, restyled): skip back/forward ±10s (new, `skipBack()`/`skipForward()` via `seekToMatchTime`), play/pause, timecode, speed pill + popover (`toggleSpeedPopover()`, replacing 5 inline buttons), scrubber with a wipe/elim/zone legend, zoom, fullscreen (new, `toggleStageFullscreen()` on `#vod-stage`). **Arrow draw tool ported from editor.html's `rotation` mode** — drag-to-draw `type:'line'` shape with arrowhead; `drawArrow()` already existed in vod.html unused, only the `onPointerDown/Move/Up` drag handling needed porting. **Elim timeline pips** — turned out to already exist generically via the `annotations` array/`renderTimelineMarkers()` loop (handoff's prior assessment that elim had no marker was stale); recolored elim's marker + event-log accent and the new legend dot to a literal amber (`#D9A404`) to read distinctly from the existing accent/yellow/red/orange used by loss/wipe/ewipe. **End Match confirmation modal added** (`#end-match-modal`, `modal-red`) — `markMatchEnd()` itself unchanged (still a toggle: ending sets `duration_seconds`, pressing again while set undoes it); the toolbar icon now calls `openEndMatchConfirm()`, which skips the modal on the undo path (safe/reversible) and only confirms before the actual end-match action. REFRESH DATA and OPEN IN EDITOR dropped entirely per this session's decision — `refreshMatchData()` and its two call sites deleted (no longer reachable from any UI). All functional JS (`setMode`, `toggleCalibration`, `toggleOverlayCanvas`, `setMapOpacity`, `manualSave`, `toggleMatchActive`, `startEvent`, `renderSquadPanel`, `renderTimelineMarkers`, `updateMatchContextBar`, etc.) kept its original element ids — only markup position/CSS changed, so all pre-existing wiring still applies untouched. ~7 now-dead CSS rules from the old toolbar (`.tb-spacer`, `.tb-divider`, `.toolbar-group-label`, `.btn-start-match`, `.sp-match-end-btn`, `.map-opacity-label`, `.mcb-refresh`/`.mcb-open-editor`) removed. *Not yet visually verified in a real browser* — this sandbox has neither Playwright nor `chromium-cli`, and the real UI sits behind Supabase magic-link auth; verified via div-balance check, inline-`<script>` parse check, and exhaustive grep for dangling element-id references instead. |

---

## SECTION 9 — BUG TRACKER & PENDING ITEMS

> Single source of truth. BUGSTOFIX.md is retired — all bugs now tracked here.
> Bugs marked ✅ are fixed and logged in Section 8. Remove from this table once fixed.
> ⚠️ Session 41's vod.html redesign moved ~350 net lines around (toolbar/panel/timeline markup relocated, several functions inserted). Any `~line N` reference below for `vod.html` predates that and may be off — grep the function name to confirm before editing.

---

### DB — Pending migrations
| Priority | Item |
|----------|------|
| 🔴 HIGH | Run Session 21 SQL: `ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_url text; ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_start_offset int;` |
| 🟡 MED | X3 prerequisite: `ALTER TABLE matches ADD COLUMN updated_at TIMESTAMPTZ DEFAULT now();` + trigger (see X3 below). Do NOT swap sort logic until column exists. |

---

### Phase 1 — Data Integrity
> These corrupt or silently lose real data. Fix first.

| ID | ✅ | File | Function | Priority | What breaks | Fix |
|----|----|----- |----------|----------|-------------|-----|
| V1+E5 | ✅ | `editor.html` | `confirmWipe` | 🔴 | Winner gets `time_of_wipe` set in Scenarios A+B | Done S32 |
| V1 | | `vod.html` | `confirmEnemyWipe` ~line 2173 (verify before editing — see note above table) | 🔴 | Same bug — Scenario A+B set `winner.time_of_wipe = t`. Remove that line and `time_of_wipe:t` from the DB update in both scenarios. Only write `confirmed_position:1`. |
| E1 | ✅ | `editor.html` | `manualSave` | 🔴 | Only active team's shapes saved | Done S32 |
| E9 | ✅ | `editor.html` | `deleteSelected` | 🔴 | `match_players` orphaned on delete | Done S32 |
| E4 | | `editor.html` | `confirmWipe` Scenario A | 🟡 | `selfTeam.confirmed_position=2` with no collision check — if user already entered pos=2 manually, two teams get #2. Guard: `if (!selfTeam.confirmed_position) { const taken = matchTeams.some(t => t.confirmed_position===2 && t.id!==selfTeam.id); if (!taken) { assign 2; } }` |

---

### Phase 2 — Editor Logic
> Break specific workflows, no data corruption. Fix after Phase 1.

| ID | ✅ | File | Function | Priority | What breaks | Fix |
|----|----|----- |----------|----------|-------------|-----|
| E2 | ✅ | `editor.html` | `confirmWipe` (x3) | 🔴 | `markMatchEnd` fired via setTimeout while pin prompt open | Done S32 — `_pinPromptOnClose` hook |
| E3 | | `editor.html` | `confirmWipe` auto-fire | 🔴 | Auto match-end fires when only a subset of teams are registered (e.g. 5 of 16) — resolved conceptually by E2 fix since `_pinPromptOnClose` defers the action; confirm working correctly in practice |
| E8 | | `editor.html` | `confirmSelfLoss` ~line 2558 | 🟡 | `markSelfWipe` fires 300ms after last death while death-pin prompt is still open. Guard: `if (stillAlive.length===0 && !_pinPromptActive) { setTimeout(markSelfWipe,300); } else if (stillAlive.length===0) { toast('All players down — log wipe when ready','warn'); }` |
| E6 | | `editor.html` | `startRename` ~line 1976 | 🟡 | Escape calls `input.blur()` → `finish()` → saves. Should restore original. Fix: capture `originalName` before edit; `input.onkeydown`: Enter → `finish(true)`, Escape → `input.onblur = ()=>finish(false); input.blur()`. `finish(save)` takes a bool — only writes to DB when `save=true`. |
| E7 | | `editor.html` | `markSelfLoss` ~line 2509 | 🟢 | Killed-by dropdown uses `!t.time_of_wipe` (static). Change to `t.time_of_wipe==null \|\| t.time_of_wipe > state.time` (same fix already in vod.html as F3). |

---

### Phase 3 — Cross-page Navigation
> Nav bar links silently load the wrong match. Fix after Phase 2.

| ID | ✅ | File | Function | Priority | What breaks | Fix |
|----|----|----- |----------|----------|-------------|-----|
| X1 | ✅ | `editor.html` | `_activateMatch` | 🔴 | `mzCtxWrite()` never called | Done S32 |
| X2 | | `vod.html` | `onMatchChange` | 🔴 | `mzCtxWrite` doesn't exist in vod — nav to EDITOR always loads wrong match. Add function near `mzCtxRead()`, call at end of `onMatchChange` after `updateMatchContextBar()`. |
| X3 | | both | `applyUrlParams` | 🟡 | Auto-load sorts by `created_at` not `updated_at` — loads newest match, not most recently worked on. **Blocked on DB migration** (see DB section). After SQL runs: swap sort to `updated_at` in both files. |

---

### Phase 4 — Session Continuity
> Quality-of-life. Fix after Phases 1–3 are stable.

| ID | ✅ | File | Function | Priority | What breaks | Fix |
|----|----|----- |----------|----------|-------------|-----|
| P1 | | `editor.html` | `beforeunload` + nav links | 🟡 | No warning when navigating away with unsaved shapes. Add `beforeunload` listener: check `Object.values(shapesCache).flat().some(s => !s.id \|\| s.id.startsWith('local_'))`. Also intercept `.nav-link` clicks. |
| P2 | | `editor.html` | `_activateMatch` + scrubber | 🟢 | Timeline position resets to 0 on every reload. Save `state.time` to `sessionStorage('mz_last_time_'+matchId)` on scrubber input; restore in `_activateMatch` after shapes load. |
| P3 | | `vod.html` | `applyUrlParams` | 🟢 | VOD review position never persisted — always restarts from `vod_start_offset`. Save/restore last review timestamp per match in sessionStorage. |
| P4 | | both | `resetView` / match load | 🟢 | Map camera (zoom/pan) resets on every load. Save/restore `{zoom, panX, panY}` to sessionStorage per match. Fall back to `resetView()` if nothing saved. |

---

### Planned Features (not bugs)
| ID | File | Priority | Description |
|----|------|----------|-------------|
| E10 | | `editor.html` | `loadMatchPlayers` step 2 | 🟢 | `role` never fetched — `.sq-player-role` tag in sidebar always invisible. Fix: change `.select('id, name')` to `.select('id, name, role')` and update `matchPlayers` map to include `role: nameById[p.player_id]?.role \|\| null` (requires `nameById` to store objects instead of plain strings). |
| FEAT-1 | `editor.html` | 🟡 | **Self Elim fight-location pin** — SELF LOSS and Wipe both open a pin prompt after confirm; SELF ELIM has no pin. After `confirmSelfElim`, call `openPinPrompt` and push a `{type:'fight', x, y, tag:'fight', time}` shape into `shapesCache[state.activeTeamId]`. |
| FEAT-2 | `editor.html` | 🟡 | **drawPath time-scrub animation** — show path points progressively as scrubber moves (using `t` values). Mini-arrow style is done (Session 34); the point-by-point reveal is still pending. |
| FEAT-3 | all pages | 🟡 | **Welcome modal** — one-time modal on first visit per session (`sessionStorage mz_welcomed` flag). Agreed Session 20, not built. |
| FEAT-4 | — | 🟢 | **history.html** — deferred until 2–3 tournaments of real data exist |
| FEAT-5 | `vod.html` | 🟢 | **Paywall** — deferred |

---

### Calibration
| Priority | Item |
|----------|------|
| 🟡 MED | Test calibration tool in real use — verify alignment accuracy per map |

---

## SECTION 10 — VOD PAGE ARCHITECTURE (v3.0)

### Boot sequence
1. `getSession()` fast path OR `onAuthStateChange` — both call `loadAllData()` then `applyUrlParams()`
2. `applyUrlParams()` — if URL params present, deep-links to that match. If no params, auto-selects most recently created match by `created_at` desc.
3. `onMatchChange()` — loads shapes, matchPlayers, restores vod_start_offset, auto-fills vod_url, calls `_createYTPlayer()` if URL present, then calls `_postMatchLoadDialogs()`
4. `_postMatchLoadDialogs()` — decides: show editor data dialog (has data, no offset) OR show skip banner (has offset, player ready) OR set `_pendingSkipOffset` (has offset, player still loading)

### YT video sync
- No YT IFrame API dependency — uses postMessage proxy
- `_createYTPlayer(videoId)` creates iframe + proxy object
- `onReady` postMessage sets `_ytPlayerReady = true`, fires `_pendingSkipOffset` if set
- `_ytTimeSyncInterval` polls every 150ms via postMessage to keep `_ytCurrentTime` fresh
- `vodStartOffset` = seconds into video where match starts
- All event timestamps = `matchTime` = `videoCurrentTime - vodStartOffset`

### Key UI elements added in v3.0
- `#skip-to-start-banner` — cyan bar below toolbar, `.show` class toggles visibility
- `#editor-data-dialog` — fullscreen overlay, `.show` class toggles
- `#offset-overwrite-dialog` — fullscreen overlay, `.show` class toggles
- `#tb-match-status` — status chip, same id/JS since v3.0. **Relocated in Session 41**: now lives inside `#vod-context-chip` (the read-only canvas overlay chip, top-left of `#vod-stage`) instead of the old toolbar row — the horizontal toolbar it used to sit in no longer exists.
- `#match-status-chip` (editor) — status chip in sidebar, below match selector. Unaffected — editor.html wasn't touched this session.
- `#msc-vod` / `#tms-vod` — "VOD ✓" badge inside each chip

### Status level logic
```
empty   = no teams added to match
partial = teams exist but no wipes + no duration
complete = has time_of_wipe on at least one team AND duration_seconds set
VOD ✓ badge = match.vod_start_offset != null
```

### Session 41 additions — floating UI chrome (boot sequence/YT sync above unchanged)
- `#vod-left-tools` — floating vertical toolbar, left edge of `#vod-stage`, vertically centered. Draw modes (zone/path/pin/**rotation**=Arrow/erase), undo, Load Match, Calibrate, Save, Start/End Match.
- `#squad-panel` — same id as before v3.0, restyled into a floating top-right panel instead of a docked sidebar. Map Opacity moved to the top; Enemy Teams is now collapsible (`toggleEnemyTeams()`, collapsed by default); the 4 event buttons are now behind a "+ LOG EVENT" popover (`toggleLogEventMenu()`).
- `#vod-context-chip` — new, replaces the old `#match-context-bar` breadcrumb. Read-only canvas overlay, top-left of `#vod-stage`: Tournament › Day › Match › Map + `#tb-match-status` + `#vid-state` + `#tb-match-time` (all relocated here, same ids).
- `#load-match-modal` — new. Houses `#vod-url-input` (relocated, same id) and the map-align fine-tune inputs (`#map-offset-x/y`, `#map-scale-f`, also relocated, same ids). Opened via the left toolbar's folder icon.
- `#end-match-modal` — new. Confirmation gate in front of `markMatchEnd()`'s end-the-match path only (the undo path — pressing again after duration is set — skips this, since it's reversible).
- `#vod-timeline` — same id, contents rebuilt: skip ±10s (`skipBack()`/`skipForward()`), play/pause, timecode, speed pill+popover (`#speed-cluster` is now a popover, not inline buttons), scrubber + wipe/elim/zone legend, zoom, fullscreen (`toggleStageFullscreen()`).
- `REFRESH DATA` / `OPEN IN EDITOR` buttons and `refreshMatchData()` — **removed entirely**, not relocated.

---

## SECTION 11 — KNOWN ISSUES / GOTCHAS

| Issue | Status |
|-------|--------|
| index.html map images — was listed as broken, confirmed already using `_lo.jpg` | ✅ Not an issue |
| vod_url + vod_start_offset SQL not confirmed run | 🔴 Must be run before VOD save/restore works |
| history.html not built | 🟡 Deferred |
| applyUrlParams on VOD: fast-boot path only runs applyUrlParams in the `if (session)` branch of getSession, not in onAuthStateChange. If magic link auth fires onAuthStateChange, applyUrlParams is also called there (both branches covered). | ✅ Handled |
| _pendingSkipOffset: if user switches match before video loads, stale offset could fire. Cleared on onMatchChange. | ✅ Handled — onMatchChange sets `_pendingSkipOffset = null` via banner dismiss |
| vod.html Session 41 redesign (floating toolbar/panel/chip/bottom bar) was verified by div-balance check, inline-`<script>` parse check, and grep for dangling element-id references only — **no live browser render was done** (no Playwright/chromium-cli in that sandbox, real UI is behind Supabase magic-link auth). First person to open it live should sanity-check: floating panel doesn't overflow on narrow viewports, the team-select dropdown (`#vlt-team-select-wrap`, top-left under the context chip) doesn't visually collide with the context chip above it, and the Arrow tool's drag-to-draw feels right. | 🟡 Needs a real look |
