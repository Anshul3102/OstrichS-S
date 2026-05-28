# Ostrich Studios — Project Context

**Project:** Script to Shots  
**Author:** Anshul (@ostrich.cinemas)  
**GitHub:** https://github.com/Anshul3102/Ostrich-Studios  
**Branch:** `ostrichxtrelby` (active) / `main` (production)  
**File:** `index.html` — 3,873 lines, single-file web app  
**Stack:** Vanilla HTML + CSS + JS, no framework, no build step, runs on `file://`

---

## What It Does

Ostrich Studios is a filmmaker tool that converts a screenplay into an annotated shot list. The workflow is:

1. **Write** — compose a screenplay in the built-in editor (or import a Final Draft `.fdx` file)
2. **Tag** — switch to Shots mode; every word in the script is clickable; clicking opens a modal to assign cinematography metadata to that moment
3. **Export** — export the shot list as CSV, print it as a PDF, or export the script as `.fdx` or PDF

**Core concept:** The number of words a filmmaker clicks = the number of shots. One annotation per word. Each annotation is one shot.

---

## Architecture

### Single-file design
Everything — HTML, CSS, JavaScript, all data constants — lives in `index.html`. No backend. No dependencies except:
- `Cormorant Garamond` from Google Fonts (UI serif)
- `AOS` (scroll animations, CDN) — landing page only

### Modes
The app has three `currentMode` states:

| Mode | State | What's shown |
|------|-------|-------------|
| `upload` | Landing page | Logo, two CTAs (upload / write new) |
| `write` | Write editor | Screenplay editor with toolbar |
| `shots` | Shot manager | Script panel (left) + shot list panel (right) |

**Important:** `write` and `shots` now share a single `#main-app` container. The shot panel slides in via CSS `width` transition (0 → 340px). Switching is controlled by adding/removing `.write-mode` / `.shots-mode` classes to `#main-app`.

---

## HTML Structure

```
<body>
  #toast-container              ← floating notifications
  <header id="main-header">     ← sticky, always visible once app loads
    .logo                       ← SVG ostrich + "OSTRICH STUDIOS" + @ostrich.cinemas
    .mode-tabs                  ← Write / Shots tab buttons
    .header-actions             ← New Script, Export CSV, theme toggle
  </header>

  #upload-screen                ← landing page (hidden once in write/shots)
    .landing-hero               ← drag-drop zone
      Upload Script card        ← triggers file picker (.fdx only)
      Write New Script card     ← starts blank write session

  #main-app                     ← unified write+shots container
    .toolbar-area               ← fixed 44px height; two toolbars overlay each other
      #write-toolbar            ← element type badge, kbd hints, word count, download
      #stats-bar                ← word count, scenes, shots tagged, coverage %
    .app-layout (flex row)
      #script-area              ← flex:1, position:relative
        #editor-wrap            ← write mode: contenteditable screenplay editor
        #script-panel           ← shots mode: read-only tokenised script view
      #shot-panel               ← width:0 in write, 340px in shots (CSS transition)
        .panel-header           ← "SHOT LIST" label + count badge
        #shot-list              ← shot cards, grouped by scene
        .panel-footer           ← Export CSV + Print List buttons

  .modal-backdrop               ← shot annotation modal (position:fixed)
    .modal
      .modal-header             ← word being edited + ×close
      .modal-body               ← all annotation sections (scrollable)
      .modal-footer             ← Remove / Cancel / Save buttons
```

---

## CSS Design System

### Theme variables
Both dark and light mode defined via CSS custom properties on `:root` and `[data-theme="light"]`.

**Dark mode (default):**
```
--bg:           #000000
--surface:      #111111
--surface2:     #1a1a1a
--border:       #2a2a2a
--accent:       #f5f2e8    ← cream white (primary brand color)
--accent2:      #ccc8bf
--text:         #e0e0e0
--text-dim:     #888888
--on-accent:    #000000    ← text on accent-bg buttons
--scene-heading-color: #ffffff
```

**Light mode (warm parchment):**
```
--bg:           #f4f1ea
--surface:      #faf8f4
--accent:       #1a1710    ← inverted (dark ink)
--on-accent:    #f5f2e8
--scene-heading-color: #1a1710
```

Additional semantic variables: `--word-hover-bg`, `--word-hover-outline`, `--badge-neutral-bg`, `--subtle-hover`, `--card-hover-shadow`, `--annotated-bg/border`, `--revision-bg/border`.

### Typography
- **UI font:** Cormorant Garamond (serif) — branding, mode tabs, element badge
- **Script font:** Courier New (monospace) — both write editor and shots view; 13px, 1.8 line-height

### Screenplay indentation (matches industry standard)
Both write mode and shots view use identical pixel values:
```
CHARACTER:     margin/padding-left: 210px
DIALOGUE:      105px left, 80px right
PARENTHETICAL: 140px left, 110px right
TRANSITION:    text-align: right
SCENE HEADING: uppercase, font-weight 700, 22px margin-top
```

---

## JavaScript Architecture

### State variables
```js
let tokens = [];          // flat array of all tokens
let tokenMap = new Map(); // id → token (fast lookup)
let wordTokens = [];      // tokens where isWord = true
let currentMode = 'upload';
let currentScriptText = '';
let currentFilename = '';
let activeId = null;      // modal: word being edited
let draft = {};           // modal: in-progress annotation
let reviewWindow = null;  // print/review tab reference
```

### Token structure
```js
{
  id: number,
  text: string,
  isWord: boolean,
  annotation: null | AnnotationObject,
  scene: { name: string, number: number } | null,
  isHeading: boolean,
  lineNumber: number,      // 1-indexed, maps to parsedLines index
  elementType: string      // SCENE_HEADING | ACTION | CHARACTER | DIALOGUE |
                           // PARENTHETICAL | TRANSITION | SUBHEADER | EMPTY
}
```

### Annotation structure
```js
{
  shotType: string | null,      // 'xws' | 'ls' | 'mls' | 'ms' | 'cu' | 'xcu'
  angle: string | null,         // 'birds' | 'high' | 'eye' | 'low' | 'worm' | 'ots'
  cameraHeight: string | null,  // 'above-head' | 'eye-level' | 'waist-level' | 'foot-level'
  movements: string[],          // ['static','pan','tilt','dolly','handheld','steadicam','crane']
  motion: string | null,        // 'static' | 'moving'
  motionDirs: string[],         // ['straight','horizontal','diagonal','parallel','opposite','clockwise','anti-clockwise']
  notes: string,
  needsRevision: boolean
}
```

### Key functions

| Function | Purpose |
|----------|---------|
| `loadScript(text, filename, skipPersist, preParsedLines?)` | Parses script text → tokens, switches to shots mode |
| `buildTokens(parsedLines)` | Converts `{text, elementType}[]` → flat token array |
| `renderScript()` | Builds shots-mode script view: groups tokens by lineNumber → `<div class="shots-line l-ELEMENTTYPE">` |
| `refreshAnnotationDisplay()` | Updates shot badges/colors without rebuilding DOM |
| `openModal(wordId)` | Opens annotation modal for a word token |
| `saveShot()` | Persists draft → token annotation, triggers toast |
| `removeShot()` | Clears annotation, triggers toast |
| `renderShotList()` | Rebuilds the shot list sidebar |
| `exportCSV()` | Downloads shot list as CSV |
| `printShotList()` | Opens print window with full shot table |
| `switchToWrite()` | Adds `write-mode` class to `#main-app` |
| `switchToShots()` | Serializes editor → `serializeEditorToLines()` → calls `loadScript()` |
| `serializeEditorToLines()` | Extracts typed lines from write editor using `data-type` attributes |
| `showToast(msg, type)` | Creates fade-in/out toast notification |

### Script parsing pipeline
```
FDX file        → parseFDXToLines()         → [{text, elementType}]
Plain text      → parsePlainTextToLines()    → detectLineTypes() → [{text, elementType}]
Write editor    → serializeEditorToLines()   → [{text, elementType}]  ← uses data-type directly
                                                       ↓
                                              buildTokens(parsedLines)
                                                       ↓
                                              tokens[], tokenMap, wordTokens[]
```

### Line type detection (for .txt / plain text only)
Two strategies selected automatically:
- **`detectLineTypesIndented`** — if source has `maxIndent >= 14 && indentedCount >= 3` (proper screenplay `.txt` export). Uses indent zones: 0=action, 6-17=dialogue/parenthetical, 18+=character.
- **`detectLineTypesHeuristic`** — for unindented text. State-machine using ALL_CAPS + context. `afterChar` flag resets on any blank line.

**Note:** Write mode bypasses detection entirely — `data-type` attributes from the editor are used directly via `serializeEditorToLines()`.

### localStorage persistence
```
sts_script       ← raw script text
sts_filename     ← last filename
sts_annotations  ← JSON map of tokenId → annotation
sts_elements     ← write-mode editor elements array [{type, text}]
sts_mode         ← 'write' | 'shots' | 'upload'
sts_theme        ← 'dark' | 'light'
```
Session is auto-restored on page load.

---

## Cinematography Data

### Shot Types (6)
`Extreme Wide Shot`, `Long Shot`, `Medium Long Shot`, `Mid Shot`, `Close Up`, `Extreme Close Up`

### Camera Angles (6)
`Bird's Eye / Top Shot`, `High Angle`, `Eye Level`, `Low Angle`, `Worm's Eye View`, `Over-the-Shoulder`

### Camera Height (4) — single-select
`Above Head`, `Eye Level`, `Waist Level`, `Foot Level`

### Camera Movements (7) — multi-select
`Static`, `Panning`, `Tilt`, `Tracking/Dolly`, `Handheld`, `Steadicam`, `Crane/Jib`

### Shot Motion (2) — binary toggle
`Static` / `Moving`

### Motion Direction (7) — multi-select
`Straight`, `Horizontal`, `Diagonal`, `Parallel`, `Opposite`, `Clockwise`, `Anti-clockwise`

---

## Write Mode Editor

A `contenteditable`-based screenplay editor using div-per-line architecture.

### Element types (Tab cycles through)
`scene` → `action` → `character` → `dialogue` → `parenthetical` → `transition` → `note`

### Keyboard behaviour
- **Enter** → creates the next logical element (character→dialogue, dialogue→character, scene→action, etc.)
- **Shift+Enter** → always creates `action`
- **Tab / Shift+Tab** → cycle element type
- **Backspace on empty line** → removes the line

### Serialization
`serializeEditorToLines()` reads `.script-line` divs in DOM order, extracts `textContent` + `data-type`, and returns `{text, elementType}[]` — this is passed directly to `buildTokens()` without going through the heuristic detector.

### Page breaks
Calculated from cumulative character count per element (different limits per type). Rendered as non-interactive `.page-break-divider` elements. Updated on every keystroke.

---

## Import / Export

### Import
- **`.fdx` (Final Draft XML) only** — parsed by `parseFDXToLines()`, which reads `<Paragraph Type="...">` elements. Falls back to plain text parsing if XML parse fails.
- Drag-and-drop supported on the landing page.

### Export
| Format | Function | Notes |
|--------|----------|-------|
| Shot list CSV | `exportCSV()` | All shots with all metadata columns |
| Shot list Print | `printShotList()` | Opens new window with printable HTML table |
| Script FDX | `downloadScriptAsFdx()` | Writes `<FinalDraft>` XML with correct paragraph types |
| Script PDF | `downloadScriptAsPdf()` | Uses `window.print()` in a styled iframe |

---

## UI/UX Features

### Landing page
- Animated logo (float keyframe)
- Staggered entrance animations (name → tagline → cards → hint)
- Drag-and-drop FDX support
- Two CTAs: Upload Script + Write New Script

### Dark / Light mode
- Toggle button (☀ / ◗) in header top-right
- Preference saved to `localStorage`
- Warm parchment light theme (`#f4f1ea` bg, `#1a1710` accent)

### Shot modal
- Smooth entrance: `scale(0.95) + translateY(10px) → scale(1)` with spring curve
- Modal header and footer are sticky; only the body scrolls
- Context box shows ±6 word window around the selected word
- Revision flag: red left-border on card, banner in modal, red badge number

### Transitions
- **Write → Shots:** editor fades out (opacity), shots view fades in, shot panel slides in (width 0→340px) — all simultaneously, ~0.35-0.42s
- **Shots → Write:** shot panel collapses, views crossfade
- Equal 44px toolbar height maintained across both modes (no layout shift)

### Toast notifications
`showToast(msg, type)` creates slide-in notifications at bottom-right. Types: `success` (green left-border), `error` (red), `info` (accent). Auto-removes after 2.6s with fade-out animation.

### Shot cards
- Color-coded tags per category: Shot=blue, Angle=purple, Movement=green, Static=grey, Moving=orange, Height=amber, Motion Direction=cyan
- Scene grouping headers with accent left-border
- Needs-revision indicator: red left-border on card

---

## Performance Decisions

| Decision | Reason |
|----------|--------|
| No `backdrop-filter` on header | Sticky blur repaints full-width on every scroll frame |
| No `backdrop-filter` on modal | Unnecessary compositing overhead |
| No `filter: drop-shadow` on animated elements | Filter + animation = GPU repaint every frame |
| `transition: all` → explicit properties | `transition: all` watches layout-triggering properties |
| `will-change: opacity` on fading panels | Promotes to GPU layer before transition starts |
| `will-change: width` on shot panel | Promotes width-animating panel to own GPU layer |
| `contain: layout style` on script panels | Scopes reflow — layout changes can't escape boundary |
| `innerHTML` string building for `renderScript()` | Browser C++ parser is ~3-5× faster than JS DOM node creation for large token arrays |
| Single delegated click handler on `#script-content` | One listener instead of one per word token (thousands of tokens) |

---

## Known Bugs / Fixed Bugs

| Bug | Fix |
|-----|-----|
| Action lines after dialogue tagged as DIALOGUE | `afterChar` flag now resets on any blank line (not just 2) |
| Write→shots mode used heuristic detection losing element types | `serializeEditorToLines()` reads `data-type` directly from editor elements |
| Modal scroll hid Camera Height / Motion Direction sections | `overflow:hidden` on `.modal`, `flex:1; overflow-y:auto` on `.modal-body` |
| Light mode text invisible on accent buttons | `--on-accent` variable (#000 dark / #f5f2e8 light) |
| Script in shots mode all left-aligned | New `renderScript()` groups tokens by `lineNumber` into `.shots-line l-{ETYPE}` divs |
| Jarring toolbar height shift on mode switch | Fixed 44px `.toolbar-area` with both toolbars `position:absolute` overlaid |

---

## Tooling Setup (vibecoding environment)

Set up alongside this project during the same session:

```
Script to Shots/
├── src/input.css          ← Tailwind source (for future UI work)
├── dist/output.css        ← compiled Tailwind output
├── tailwind.config.js     ← brand colors, custom animations, font families
├── postcss.config.js
├── package.json           ← scripts: dev (--watch) / build (--minify)
├── template.html          ← starter page with dark mode, AOS, nav, hero, feature grid
└── .gitignore
```

**npm scripts:**
```bash
npm run dev    # tailwindcss watch mode
npm run build  # minified production output
```

**VS Code extensions installed:**
Tailwind CSS IntelliSense, Prettier, Live Server, Auto Rename Tag, Error Lens, Path Intellisense, Material Icon Theme, GitHub Copilot, ESLint, Color Highlight

**VS Code settings:** format on save, JetBrains Mono font, ligatures, smooth scroll, minimap off, auto-save on focus change, bracket pair colorization.

---

## Git History Summary (37 commits)

```
3f8fda2  Fix lag: remove backdrop-filter, transition:all; add will-change/contain
b69f009  Fix jarring mode transition: equal toolbar heights, opacity crossfade
99eaf3d  Unify write/shots layout: shot panel slides in, font consistency
e021b1f  UI/UX overhaul: glass header, modal animations, toast, micro-interactions
24184e6  Fix all light mode visual inconsistencies
ff9d501  Add dark/light mode toggle with warm parchment light theme
2c038a9  FDX-only import; add FDX export; remove PDF/TXT upload paths
46016ef  Fix write→shots: use editor data-type directly, bypass heuristic
02780cf  Fix detectLineTypes: reset afterChar on blank line
2940861  Fix screenplay indentation: ch units, left-align content block
378b9d4  Fix screenplay formatting: line-div structure per elementType
ee0646e  Add Camera Height and Motion Direction to shot modal
f125e36  Redesign landing page, remove Overview tab, PDF branding, print checkboxes
db8fb45  Perf: innerHTML+delegation for renderScript
eb6552a  Add PDF upload via PDF.js
...      (37 total from initial prototype to current)
```

---

## What's Next (potential)

- Mobile responsive layout (shot panel as bottom sheet on mobile)
- Script scene detection improvements
- Shot list reordering (drag handles)
- Keyboard shortcut to open/close modal
- Export shot list as image (for Instagram/sharing)
- Multi-script project management
- Shot coverage visual (highlight density map on script)
