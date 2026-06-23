# METAZONE — Design Evolution Handoff
**Document type:** Design history & rationale  
**Covers:** v1 (dark navy) → current redesign (warm parchment)  
**Purpose:** Anyone picking up this project should understand not just what changed, but why each decision was made and what it was reacting against.

---

## The Short Version

We started with a dark navy UI that looked like every other esports dashboard ever made. We are now building something that looks like a military intelligence dossier — warm, physical, high-contrast, human-made. The functional logic of the app didn't change. The entire surface changed.

---

## Phase 1 — The Original System (v1–v8)
### What it looked like

Dark navy base (`#0a0f1e` family). Cyan accent (`#00d4ff` family). Glowing elements. Monospace font throughout. Grid lines. Status badges with pulsing dots. A "terminal readout" in the footer showing kernel version and latency. Data replication status in the sidebar.

In short: the aesthetic of every esports analytics tool, every competitive gaming dashboard, and most "AI-powered" SaaS products built in 2021–2024.

### Why it existed

It felt correct at the time. Esports = dark + neon. The navy/cyan combination reads as "technical" and "serious" to the target audience of coaches and analysts who are used to game UIs. It was also the path of least resistance — dark mode is easier to design when you're moving fast because contrast is forgiving.

### What it was actually communicating

"This was made by an AI or a template."

The specific combination of dark background + glowing cyan + monospace font + status dots is so saturated in the market that it carries no identity. It says: productivity tool, not intelligence platform. It says: someone applied a theme, not someone made a deliberate choice.

The footer bar (kernel readouts, latency display) was the clearest symptom. That information was fake — decorative technical data that served no user need. It was there because it *looked* like a serious system. That's a red flag. Real serious systems don't perform seriousness, they just work.

### What was kept from Phase 1

The structure. Three-column layout (sidebar / main grid / right panel) was correct from the start and didn't change. The hierarchy — tournament → day → match — was correct. The sidebar tree, day tabs, and match cards as the primary navigation objects all survived.

---

## Phase 2 — The Pivot Decision

### The problem statement

METAZONE is targeting coaches and analysts at established BGMI competitive organisations. These are people who watch hours of VOD, manually annotate kill positions, track rotation patterns, and argue about zone cycles. They are professionals doing knowledge work, not consumers playing a game.

The v1 aesthetic was designed for the latter. It was designed to feel exciting. An analyst who has been staring at match data for six hours doesn't want exciting. They want clear, fast, trustworthy.

The pivot question was: **what does a tool that professionals trust actually look like?**

### The reference point

Medical dashboards. Not hospital drama UI — actual clinical record systems and diagnostic tools. They share a set of properties with what METAZONE needed:

- White or off-white backgrounds (information is the content, not the chrome)
- Dense data with clear hierarchy (big numbers for the things that matter most)
- No decorative chrome (no glows, no gradients standing in for content)
- High-contrast text (you're reading this under pressure and fatigue)
- Colour as signal, not atmosphere (colour appears where it means something specific)

The medical reference was a reframe, not a literal direction. METAZONE doesn't look like a hospital. But it inherits the underlying logic: **the tool should feel like it was made by someone who respects the user's intelligence and time.**

### The anti-patterns list

These were identified as explicit things to eliminate:

| Pattern | Why it was cut |
|---|---|
| Dark background + cyan glow | Zero differentiation, generic esports visual |
| Purple-indigo gradients | The AI product cliché of 2023–2024 |
| Glowing elements as "premium" signal | Decorative, no information value |
| Cards inside cards | Visual noise, false depth |
| Emoji icons in UI | Breaks professional tone |
| Meaningless status dots | Fake technical credibility |
| Footer kernel/latency bar | Decorative data, no user value |
| Data replication sidebar line | Same — performed seriousness |
| Placeholder gradient backgrounds | Standing in for real content |

---

## Phase 3 — The New System (Current)

### Palette logic

Three-colour brand palette, strictly hierarchical:

**Forest green (`#2C4A2E`)** — authority colour. Used for the header bar, selected states, focus rings, and the EDITOR action button. The darkest brand colour. When something is forest green, it means "this is the system's primary identity."

**Olive (`#4E7C2F`)** — secondary green. Used for LOGGED badges, section eyebrows, "open" links, active indicators that aren't primary CTAs. Lighter, more neutral than forest. Means "this item is active/filled."

**Orange (`#C85E0A`)** — the only warm colour in the palette. Appears only for: the logo square, active nav underline, selected day tab, NEW TOURNAMENT button, VOD link, primary CTAs. It always signals "action" or "this is selected." Because it appears rarely, it has real signal weight. If everything were orange, nothing would be.

**Structural colours:**
- `#F2EDE4` — main canvas. Warm off-white, parchment. Not pure white (too clinical), not cream (too soft). Slightly aged paper.
- `#E8E1D5` — panels (sidebar, right panel). Barely offset from the canvas, just enough to define zones.
- `#EDE7DC` — card backgrounds. Almost identical to canvas, slight separation.

**Text:**
- `#0C0A07` — near-black, warm-tinted. For big display numbers and hero titles.
- `#1C1A14` — body text.
- `#2C4A2E` — forest green for ALL CAPS labels. Deliberate choice: using the brand colour instead of grey adds character and cohesion.
- `#6B6048` — secondary/dim text. Warm brown-khaki, not cool grey. This is important — using a cool grey dim would fight the parchment warmth.

**Borders:** `#C9BFA8` → `#A89880`. Warm-tinted tan, not neutral grey. The warmth has to run all the way through or the palette breaks.

### Font system

Three fonts, each with a specific job:

**Oswald (`--font-display`)** — condensed, high-impact, always uppercase. Used for hero numbers, map names, the page title, team kill counts. Anything you need to read at a glance from a distance. The "information at speed" font.

**Manrope (`--font-body`)** — clean, humanist sans. Used for tournament names, team names, body copy, meta values. The "readable prose" font.

**Lexend Deca (`--font-mono`)** — geometric, mono-adjacent. Used for every label, tag, badge, nav link, button text, eyebrow text. Always uppercase, tightly letter-spaced. This is the UI fabric — it does the most heavy lifting because it appears at the smallest sizes (8–10px) and needs to be readable there.

The previous system used JetBrains Mono for all three roles. Monospace fonts work well for code and numeric data. They are wrong for labels, navigation, and prose because they carry a "terminal" connotation that was fighting the direction.

### Size ladder

| Size | Font | Used for |
|------|------|----------|
| 30px | Oswald | Main page title |
| 22px | Oswald | Card stat values, map names |
| 20px | Oswald | Right panel title |
| 19px | Oswald | Logo "METAZONE" |
| 18px | Oswald | Team kill count in right panel |
| 12px | Manrope | Body text, sidebar names, meta values |
| 11px | Manrope | Team names |
| 10px | Lexend Deca | Nav links, button labels |
| 9px | Lexend Deca | Eyebrows, CTAs, day names |
| 8px | Lexend Deca | Badges, section labels, card footers |

### Component decisions

**Header bar** — forest green background, white text and nav links. Orange underline on the active nav item. This zone separation (dark header / light body) is the strongest single structural decision. It immediately tells the user "this is a system with a defined top and a content area" — it reads as an authority layer.

**Match cards** — map image with a dark-to-transparent gradient overlay (not light). Map name and match number appear in white over the dark gradient. Below the image: three stat cells (teams / kills / duration) in a bordered row. Bottom: two equal-width action buttons side by side — EDITOR (forest green, solid) and VOD (parchment background, orange text). This is a deliberate half/half split so neither action dominates visually, but the colours signal priority (go to editor = primary, VOD = secondary option).

**Add Match card** — dashed border card, same grid position as real match cards. Appears in empty state (only card in the grid) and appended after the last real card when matches exist. Orange on hover. This means the user never has to look for a button — the action lives in the content zone itself.

**Sidebar buttons** — tournament settings and delete buttons are now always visible with coloured boxes (forest green box for settings, red box for delete). Previous version hid them until hover. Hidden affordances require discovery — visible affordances with colour-coded intent (green = configure, red = danger) are faster and clearer.

**Right panel** — same dark gradient treatment on the map preview. The panel exists to surface match intelligence when a card is selected. It's a detail layer, not a navigation layer — the slightly darker panel background (`#E8E1D5`) signals this.

---

## The Core Thinking Shift

### Old thinking: "what does an esports tool look like?"
Answer: dark, glowing, technical-looking. So that's what got built.

### New thinking: "what does a tool that analysts trust look like?"
Answer: clear, legible, hierarchical, restrained. Colour appears where it means something. Numbers are big when they matter. The chrome stays out of the way.

The design anti-pattern that drove the v1 aesthetic was **performing the domain** — making the tool look like esports rather than making the tool serve the people who work in esports. The new direction is to build something that feels like it was made by someone who understood the work, not someone who watched esports streams and extracted an aesthetic.

The parchment background is a concrete expression of this. It's an unusual choice for esports software. It reads as physical, aged, like a dossier or a field report. That's intentional — METAZONE is a record-keeping and analysis system. It should feel like documentation, not a game.

---

## What Stays Sacred

These decisions are locked and should not be relitigated:

1. **Three-column layout** — sidebar / main / right panel. Tested, correct.
2. **Orange as the sole action/active signal** — never use orange for decoration. It loses meaning if it appears everywhere.
3. **Forest green for the header** — the authority layer. This is the clearest single zone-separator.
4. **No glow effects** — not for hover, not for selected state, not as a "premium" signal. Glow = dated, generic.
5. **No dark backgrounds on the body** — the parchment warmth is load-bearing. It's what makes this feel different.
6. **Colour on ALL CAPS labels = forest green, not grey** — using the brand colour for micro-labels is what ties the whole system together.
7. **Borders are warm-tinted** — `#C9BFA8` / `#A89880`, never neutral grey. Grey borders would fight the parchment warmth.

---

*Document written at the end of the dashboard redesign iteration. Update when further design decisions are made.*
