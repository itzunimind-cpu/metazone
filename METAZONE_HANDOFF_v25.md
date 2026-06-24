# METAZONE — Project Handoff & Continuity Document
**Version:** v32 — Session 32. editor.html: all 5 high-priority bugs fixed (E1 manualSave, E2 pin/matchEnd race, E9 match_players delete, V1+E5 winner wipe, X1 mzCtxWrite).
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
| index.html | v9.5 | Session 26 | All sidebar + grid bugs fixed. Skeleton loader added. Card reveal animation. |
| editor.html | v6.5 | Session 32 | All 5 high-priority bugs fixed — E1, E2, E9, V1+E5, X1 |
| tournament-create.html | v4.1 | Session 25 | VOD REVIEW nav link added |
| tournament-editor.html | v1.4 | Session 25 | VOD REVIEW nav link added |
| analytics.html | v2.3 | Session 25 | VOD REVIEW nav link added, active link fixed |
| player.html | v1.2 | Session 25 | VOD REVIEW nav link added |
| vod.html | v3.0 | Session 25 | Full QoL overhaul — see Section 8 changelog |
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
| `draw()` | Main canvas render loop |
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
| `renderActiveSquadInline()` | Renders match_players inline below ACTIVE SQUAD button in right panel |
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
| `toggleCalibration()` | Enter/exit 2-point calibration mode |
| `_calInverse(wx, wy)` | World → corrected world coords for calibrated map |
| `updateMapOffset()` | Reads X/Y/scale toolbar inputs, redraws |
| `resetMapOffset()` | Zeros manual alignment |
| `setSpeed(rate)` | YT playback rate |
| `setMode(mode)` | Drawing tool mode |
| `manualSave()` / `queueForSave(shape)` | Shape persistence |

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
| vod.html | ✅ Complete | v3.0 — full QoL overhaul |
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

---

## SECTION 9 — BUG TRACKER & PENDING ITEMS

> Single source of truth. BUGSTOFIX.md is retired — all bugs now tracked here.
> Bugs marked ✅ are fixed and logged in Section 8. Remove from this table once fixed.

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
| V1 | | `vod.html` | `confirmEnemyWipe` ~line 847 | 🔴 | Same bug — Scenario A+B set `winner.time_of_wipe = t`. Remove that line and `time_of_wipe:t` from the DB update in both scenarios. Only write `confirmed_position:1`. |
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
| FEAT-1 | `editor.html` | 🟡 | **Self Elim fight-location pin** — SELF LOSS and Wipe both open a pin prompt after confirm; SELF ELIM has no pin. After `confirmSelfElim`, call `openPinPrompt` and push a `{type:'fight', x, y, tag:'fight', time}` shape into `shapesCache[state.activeTeamId]`. |
| FEAT-2 | `editor.html` | 🟡 | **drawPath upgrade** — animate path point-by-point using `t` values as scrubber moves |
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
- `#tb-match-status` — status chip in toolbar row 1, next to match dropdown
- `#match-status-chip` (editor) — status chip in sidebar, below match selector
- `#msc-vod` / `#tms-vod` — "VOD ✓" badge inside each chip

### Status level logic
```
empty   = no teams added to match
partial = teams exist but no wipes + no duration
complete = has time_of_wipe on at least one team AND duration_seconds set
VOD ✓ badge = match.vod_start_offset != null
```

---

## SECTION 11 — KNOWN ISSUES / GOTCHAS

| Issue | Status |
|-------|--------|
| index.html map images — was listed as broken, confirmed already using `_lo.jpg` | ✅ Not an issue |
| vod_url + vod_start_offset SQL not confirmed run | 🔴 Must be run before VOD save/restore works |
| history.html not built | 🟡 Deferred |
| applyUrlParams on VOD: fast-boot path only runs applyUrlParams in the `if (session)` branch of getSession, not in onAuthStateChange. If magic link auth fires onAuthStateChange, applyUrlParams is also called there (both branches covered). | ✅ Handled |
| _pendingSkipOffset: if user switches match before video loads, stale offset could fire. Cleared on onMatchChange. | ✅ Handled — onMatchChange sets `_pendingSkipOffset = null` via banner dismiss |
