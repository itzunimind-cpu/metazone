# METAZONE — Project Handoff & Continuity Document
**Version:** v25 — Session 25. Match status chips, VOD QoL overhaul, nav standardised across all pages, VOD auto-loads most recent match.
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
| index.html | v9.1 | Session 25 | MAP_IMGS fix pending (not uploaded this session — see Section 9) |
| editor.html | v5.7 | Session 25 | Match status chip, status icons in dropdown, updateMatchStatusChip() |
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
| `refreshDropdowns()` | Populates all selectors + status icons in match dropdown |
| `_matchStatusLevel(match, matchTeams)` | Returns `'empty'`\|`'partial'`\|`'complete'` |
| `updateMatchStatusChip()` | Updates the status chip below match selector |
| `renderTeamList()` | Redraws sidebar team list |
| `draw()` | Main canvas render loop |
| `loadMapKey(key)` | Loads map at correct resolution |
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
| index.html | ✅ Complete | MAP_IMGS bug identified — needs `_lo.jpg` fix (see Section 9) |
| editor.html | ✅ Complete | v5.7 — status chip added |
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

---

## SECTION 9 — PENDING ITEMS (next session)

| Priority | Item |
|----------|------|
| 🔴 HIGH | Run Session 21 SQL: `ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_url text; ALTER TABLE matches ADD COLUMN IF NOT EXISTS vod_start_offset int;` |
| 🔴 HIGH | Fix index.html MAP_IMGS: change all 4 values from `_High_Res.png` to `_lo.jpg`. One-line fix in script block. |
| 🟡 MED | Test calibration tool in real use — verify alignment accuracy per map |
| 🟡 MED | editor.html drawPath upgrade: animate path point-by-point using `t` values when scrubber moves |
| 🟡 MED | Welcome modal on all pages (sessionStorage `mz_welcomed` flag) — agreed in Session 20, not yet built |
| 🟢 LOW | history.html — deferred until 2–3 tournaments of real data |
| 🟢 LOW | vod.html paywall — deferred |

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
| index.html map images broken (uses High_Res.png, should use _lo.jpg) | 🔴 Identified, fix is one line in MAP_IMGS const, not yet applied |
| vod_url + vod_start_offset SQL not confirmed run | 🔴 Must be run before VOD save/restore works |
| history.html not built | 🟡 Deferred |
| applyUrlParams on VOD: fast-boot path only runs applyUrlParams in the `if (session)` branch of getSession, not in onAuthStateChange. If magic link auth fires onAuthStateChange, applyUrlParams is also called there (both branches covered). | ✅ Handled |
| _pendingSkipOffset: if user switches match before video loads, stale offset could fire. Cleared on onMatchChange. | ✅ Handled — onMatchChange sets `_pendingSkipOffset = null` via banner dismiss |
