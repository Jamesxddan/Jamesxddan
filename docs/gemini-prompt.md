# Build Prompt for Gemini — Lightweight SCORM Player with Learner & Editor Views

> Copy everything below the line into Gemini as a single prompt.

---

## ROLE

You are a senior front-end engineer specialising in e-learning runtimes, browser
sandboxing and extremely small JavaScript bundles. You write production code, not
pseudocode. You do not add dependencies unless they are unavoidable, and you justify
every byte.

## WHAT I WANT

Build **SCORM Lite Player** — a single, dependency-light, 100% client-side JavaScript
plugin. The user drops a SCORM `.zip` into it, the learning module starts playing
immediately, and the user can switch between two views of that same running module:
a **Learner View** and an **Editor View**.

Deliver working code, not a plan.

## NON-NEGOTIABLE CONSTRAINTS

1. **Lightweight above all.** Hard budget: **≤ 150 KB gzipped** for the entire player
   (JS + CSS, excluding the course content itself). Report the actual gzipped size in
   your final output. If a feature would blow the budget, make it a lazily-loaded chunk
   and say so.
2. **No framework.** Vanilla JavaScript (ES2020+) and plain CSS. No React, Vue, Angular,
   Svelte, jQuery, Tailwind, Bootstrap. If you want component ergonomics, use small
   template literal helpers you write yourself.
3. **Allowed dependency: exactly one.** `fflate` for ZIP decompression (~8 KB gz, faster
   and far smaller than JSZip). Everything else you implement: XML parsing uses the
   built-in `DOMParser`, audio uses `<audio>`, nothing else.
4. **Zero backend.** No server, no upload, no network calls. The `.zip` is read with the
   `File`/`Blob` API and stays in the browser's memory. The player must work from
   `file://` and from any static host.
5. **Plugin-shaped, not app-shaped.** Two usage modes, both must work:
   - UMD/IIFE: `<script src="scorm-lite.js"></script>` then
     `ScormLite.mount('#container', { view: 'learner' })`
   - ESM: `import { mount } from 'scorm-lite'`
   Expose a small public API: `mount()`, `load(fileOrArrayBuffer)`, `setView('learner'|'editor')`,
   `goToSlide(id)`, `export()`, `destroy()`, plus an `on(event, cb)` event bus.
6. **Autostart.** From file-drop (or file-picker, or a programmatic `load()`) the player
   must: unzip → parse `imsmanifest.xml` → resolve the launch resource → mount the SCO in
   an iframe → start playback. **Zero clicks in between.** Show a progress indicator
   during unzip for large packages.
7. **Browser support.** Latest Chrome, Edge, Firefox, Safari. No IE, no polyfills.

## SCORM SUPPORT

Support **SCORM 1.2** and **SCORM 2004 (2nd, 3rd, 4th edition)**. Detect the version
from the manifest's schema/schemaversion and the namespaces present.

- Parse `imsmanifest.xml`: `<organizations>` → the default `<organization>` → the nested
  `<item>` tree (this is your slide/page outline, preserving hierarchy), and `<resources>`
  → `<resource>` with `href` and `<file>` entries. Resolve `xml:base` on both manifest
  and resource. Resolve every href relative to the package root.
- Build an internal normalised model:
  ```
  Package {
    version: '1.2' | '2004',
    title, identifier,
    files: Map<path, Uint8Array>,          // the whole unzipped package
    items: Item[]                           // flat + tree, in order
  }
  Item { id, title, href, parentId, depth, resourceId, children[], scormType,
         prerequisites, masteryScore, completionThreshold }
  ```
- **Fallback:** if `imsmanifest.xml` is missing or unparseable, do not fail. Scan the zip
  for HTML entry points (`index.html`, `story.html`, `story_html5.html`, `scormdriver/indexAPI.html`,
  `res/index.html`, `multiscreen.html`), pick the most likely one, and show a
  non-blocking banner: "No SCORM manifest found — playing as plain HTML package."

## RUNTIME: SERVING THE UNZIPPED COURSE

This is the hardest part. Get it right.

- **Primary strategy — Service Worker virtual filesystem.** Register a Service Worker that
  claims the scope `./__scorm/`. Each loaded package gets an id; the player posts the file
  map to the SW. The SW's `fetch` handler answers any request under
  `./__scorm/<pkgId>/<path>` out of that map, with a correct `Content-Type` derived from
  the file extension (html, htm, js, css, json, xml, mp3, m4a, ogg, wav, mp4, webm, png,
  jpg, jpeg, gif, svg, webp, woff, woff2, ttf, otf, swf → `application/octet-stream`).
  The SCO iframe then has `src = ./__scorm/<pkgId>/<launchHref>` and **every relative path,
  `fetch`, and `XMLHttpRequest` inside the course works unchanged.** This is what makes
  real-world Storyline/Captivate output actually run.
- **Fallback strategy — blob rewriting.** If Service Workers are unavailable (`file://`,
  private mode, no HTTPS), fall back to: create `blob:` URLs for every asset, then rewrite
  `src`/`href`/`url()` references inside HTML and CSS files to point at those blobs, and
  mount the entry HTML as a blob too. Document the known limitation that runtime-constructed
  paths will not resolve in this mode, and surface it as a one-line warning.
- Revoke blob URLs and unregister/clear the SW cache for a package on `destroy()` and when a
  new package is loaded. No memory leaks across loads.

## RUNTIME: THE SCORM API SHIM

The SCO looks for an LMS API by walking `window.parent` / `window.opener` chains. Give it one.

- Expose **`window.API`** (SCORM 1.2) and **`window.API_1484_11`** (SCORM 2004) on the page
  that hosts the iframe, *before* the iframe loads.
- Implement the full method sets with correct signatures and return values (strings, not booleans):
  - 1.2: `LMSInitialize`, `LMSFinish`, `LMSGetValue`, `LMSSetValue`, `LMSCommit`,
    `LMSGetLastError`, `LMSGetErrorString`, `LMSGetDiagnostic`
  - 2004: `Initialize`, `Terminate`, `GetValue`, `SetValue`, `Commit`,
    `GetLastError`, `GetErrorString`, `GetDiagnostic`
- Back it with an in-memory CMI data model with the common elements populated sensibly
  (`cmi.core.lesson_status` / `cmi.completion_status`, `cmi.core.score.raw` / `cmi.score.scaled`,
  `cmi.core.lesson_location` / `cmi.location`, `cmi.suspend_data`, `cmi.core.session_time` /
  `cmi.session_time`, `cmi.interactions.n.*`, `cmi.objectives.n.*`, `cmi.core.student_name`,
  `cmi.mode`, `cmi.entry`, `cmi.exit`, `cmi.credit`).
- Return correct error codes (0 no error, 101 general exception, 201 invalid argument,
  301 not initialised, 401 not implemented, 403 data model element value not initialized,
  404 read only, 405 write only, 406 type mismatch).
- **Persist** the data model to `localStorage` keyed by a hash of the package, so reopening
  the same course resumes (`cmi.entry = "resume"`, suspend_data restored). Provide a
  "Reset progress" control.
- **Log every API call** (method, element, value, return, error, timestamp) into a ring buffer.
  Editor View shows this log live.

## VIEW 1 — LEARNER VIEW

The experience a real student gets. Clean, uncluttered, the slide canvas dominant.

- Slide canvas (the iframe), scaled to fit its container, aspect-ratio preserved, with
  letterboxing. Responsive down to tablet width.
- Playback bar: play/pause, seek/scrub where the content exposes a timeline, elapsed/total,
  volume + mute, fullscreen, captions toggle if the package has them.
- Prev / Next navigation and a collapsible course outline/menu built from the manifest item tree,
  with per-item completion ticks driven by the CMI model.
- Progress bar + "x of y slides" + overall completion status.
- **Respect sequencing.** If the manifest declares prerequisites or `adlseq`/`imsss` sequencing
  rules, lock forward navigation accordingly and show *why* an item is locked. Include an
  "unlock navigation" toggle (off by default) for reviewers.
- Resume: on reopening a package, offer "Resume where you left off / Start over".
- Keyboard: ←/→ prev/next, Space play/pause, F fullscreen, M mute, Esc exit fullscreen.

## VIEW 2 — EDITOR VIEW

**Critical: the editor also has full playback.** It is the learner experience *plus* an
inspector around it. Switching between views must **never reload or restart** the module —
same iframe, same SCORM session, same current slide and playhead.

Layout: left sidebar = slide navigator, centre = the live slide canvas with playback
controls, right sidebar = tabbed inspector, bottom = collapsible SCORM data/log drawer.

### 2a. Slide navigator (left)
- Every slide / page / SCO from the manifest item tree, hierarchical and collapsible.
- Each row: index, title, id, duration (if known), small badges for asset counts
  (🔊 audio, 🎬 video, 🖼 image, ⚡ interactions).
- **Click any row to jump straight to that slide, in any order, ignoring lock rules.**
- Thumbnail preview generated lazily (render slide offscreen or use the package's own
  thumbnail assets when present — Storyline ships them). Never block on thumbnails.
- Search/filter by title or by text content.

### 2b. Audio tab
- Every audio file referenced by the current slide, **and** a "whole package" mode listing
  all audio in the zip.
- Per file: filename, path, format, size, duration, and an inline `<audio>` player with
  play/pause/seek/volume.
- Show *where* the audio is used — which slides reference it, and which trigger plays it.
- Download button per file, and "Replace audio…" which swaps the bytes in the package model
  (used by Export).

### 2c. Text tab — must be editable
- Extract **all** readable text on the current slide: text nodes in the slide DOM, `alt`
  and `title` attributes, `aria-label`s, and strings in the authoring tool's data files
  (e.g. Storyline's `data.js` / frame JSON, Captivate's project JS).
- Render as a list of editable fields grouped by origin, each showing its source
  (`file.html › div.caption`) so the user knows what they are changing.
- Editing a field must **immediately** update the live slide in the iframe (write through
  to the DOM node) **and** the in-memory package model, so the change survives navigation
  and lands in the export. Provide undo/redo (Ctrl+Z / Ctrl+Shift+Z) and a per-field revert.
- Where text lives inside a rendered canvas/SVG and cannot be safely patched, mark it
  read-only and say why, instead of silently failing.

### 2d. Interactions tab — "what does this button do, and where is it?"
This is the headline feature. Be thorough.

- List **every interactive element on the current slide**. For each one show:
  - **Label** — its visible text, aria-label, or a generated name.
  - **Type** — button / hotspot / next / previous / menu jump / audio trigger / layer
    toggle / quiz option / submit / external link / drag target / unknown.
  - **Position** — the element's rect. **Hovering a list row draws a highlight outline over
    that element on the live slide**, with a label callout and a leader line. Clicking the
    row scrolls it into view and keeps the outline pinned. Also offer a "show all" mode
    that outlines and numbers every interactive element at once, like a heat map.
  - **Action — what happens if you click it.** Resolve to a human sentence:
    "Jumps to slide 7 — *Safety Checklist*", "Plays `audio_04.mp3`",
    "Shows layer *Feedback – Correct*", "Sets variable `attempts = attempts + 1`",
    "Submits the answer and calls `LMSSetValue('cmi.core.score.raw', …)`",
    "Opens https://… in a new tab".
  - **Test button** — fires the action so the user can see the result, with an "undo /
    return to previous slide" affordance.
- Resolve actions with three layers, in this order, and **always say which layer produced
  the answer** (a confidence badge: *parsed* / *inferred* / *unknown*):
  1. **Authoring-tool parsers.** Detect and read the structured data these tools ship:
     - **Articulate Storyline** — `story_content/frame.xml`, `data.js` / `story.js`
       (slide, layer, trigger, variable objects)
     - **Articulate Rise** — the course JSON under `course/`
     - **Adobe Captivate** — `cp*.js`, `project.js`, `assets/`
     - **iSpring**, **Camtasia**, **Lectora**, **Elucidat** — best effort on their data files
     Walk their trigger structures: event (click / hover / timeline-start / variable-change)
     → action (jump to slide / play media / show layer / set variable / submit) → target.
  2. **Generic DOM analysis.** Walk the iframe DOM for `<a>`, `<button>`, `[role=button]`,
     `[onclick]`, `[tabindex]`, elements with cursor:pointer, and `<area>` map regions; read
     their `href`/`onclick` and pattern-match common SCORM nav calls.
  3. **Runtime instrumentation.** Inside the iframe, wrap `EventTarget.prototype.addEventListener`
     *before the course scripts run* to record which elements bind which event types, and wrap
     `HTMLMediaElement.prototype.play` and the SCORM API calls to attribute runtime effects
     back to the element that was clicked.
- Never guess with false confidence. An unresolved element is listed as
  "Unknown action — handler bound at `script.js:1420`", not invented.

### 2e. Assets tab
Full file tree of the package with size, type, and a preview for images/video/audio/text.
Total package size, largest files, and unreferenced ("orphan") file detection.

### 2f. SCORM drawer (bottom)
Live CMI data-model table (element / value / last-set), editable for testing, plus the API
call log (method, element, value, result, error, timestamp), filterable, with a clear button.

### 2g. Export
Rebuild the `.zip` with all text and audio edits applied, re-zipping with `fflate`. Files
that were not edited are copied through byte-for-byte. Download as
`<original-name>-edited.zip`. Show a diff summary of what changed before the download.

## UX DETAILS

- **Drop zone**: full-page drag-and-drop plus a "Choose file" button. Accept `.zip`.
  Visual drag state. Reject non-zip with a clear message.
- **View switch**: one obvious toggle, top-right — `Learner | Editor`. Persist the choice.
- **Dark and light themes**, following `prefers-color-scheme`, with a manual override.
  All colours as CSS custom properties on `:root`, redefined for dark mode.
- **Errors never blank the screen.** Corrupt zip, missing manifest, missing launch file,
  blocked iframe, SW registration failure — each produces a readable message explaining
  what happened and what still works.
- **Accessibility**: keyboard reachable, visible focus rings, ARIA roles on the panels and
  the slide list, `prefers-reduced-motion` respected.
- Handle large packages (500 MB+): stream/chunk the unzip, keep the UI responsive, show
  progress, and do the heavy unzip work in a Web Worker (`fflate` supports this) so the
  main thread never locks.

## DELIVERABLES

Produce a complete, runnable project:

```
scorm-lite/
  src/
    index.js            # public API: mount, load, setView, goToSlide, export, destroy, on
    zip.js              # fflate wrapper, worker-based unzip, re-zip
    manifest.js         # imsmanifest.xml parsing → normalised Package model
    vfs.js              # Service Worker registration + file map + blob fallback
    sw.js               # the Service Worker itself
    scorm-api.js        # SCORM 1.2 + 2004 shim, CMI model, error codes, persistence
    player.js           # iframe mount, playback control, navigation, sequencing rules
    views/
      learner.js
      editor.js
    inspector/
      slides.js         # navigator + thumbnails
      audio.js
      text.js           # extraction + in-place editing + undo stack
      interactions.js   # trigger resolution + overlay highlighting
      assets.js
      scorm-log.js
    analyzers/
      storyline.js
      rise.js
      captivate.js
      generic-dom.js
      runtime-hooks.js
    ui/
      styles.css
      components.js
  demo/
    index.html          # drop a zip, it plays
  README.md
  package.json
  vite.config.js        # build to ESM + UMD, report gzip size
```

Along with the code, give me:
1. The **actual gzipped bundle size**, and a per-module breakdown.
2. A short **README** with install, the public API, the events, and the known limitations
   (be honest about what the blob fallback cannot do, and which authoring tools are
   fully parsed vs. best-effort).
3. A **test plan**: which SCORM packages to try (SCORM 1.2 single-SCO, 2004 multi-SCO with
   sequencing, Storyline export, Rise export, Captivate export, a zip with no manifest,
   a corrupt zip) and the expected behaviour for each.

## HOW TO WORK

- Write the real code, file by file, complete — no `// TODO: implement` in the core path.
- Start with the pipeline that proves the whole thing works end to end: drop → unzip →
  manifest → VFS → iframe → SCORM shim → playing. Get that running first, then build the
  Learner View, then the Editor View panels in the order listed above.
- Comment the non-obvious parts: the SW fetch handling, the API discovery walk, the
  trigger resolution heuristics.
- At every step, ask yourself whether the code you just wrote is the smallest thing that
  works. If it is not, rewrite it. **Lightweight is the first requirement, not the last.**
