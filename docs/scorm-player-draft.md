# SCORM Player — Draft Concept

Status: **Draft for review.** Nothing is built yet. This document is the human-readable
design note; `docs/gemini-prompt.md` is the build prompt handed to Gemini.

## 1. Goal in one line

Drop a SCORM `.zip` onto a page and the learning module starts playing immediately —
no server, no upload, no LMS — with two switchable views: **Learner View** and **Editor View**.

## 2. Hard constraints

| Constraint | Decision |
|---|---|
| Weight | Target < 150 KB gzipped for the whole player. No React/Angular/Vue. Vanilla JS + `fflate` for unzip. |
| Backend | None. 100% client-side. The zip never leaves the browser. |
| Install | Single bundle usable as `<script>` + one `<div>`, or as an npm package. "Plugin", not an app. |
| Input | Any SCORM 1.2 or SCORM 2004 (2nd–4th ed.) package. Graceful fallback for plain-HTML/AICC-ish zips. |
| Autostart | Zip in → manifest parsed → first SCO launched. Zero clicks between drop and playback. |

## 3. The two views

Both views share **one** running module instance (same iframe, same SCORM state).
Switching views never restarts the course.

### Learner View
What a real student sees. Course chrome only: slide canvas, play/pause, seek, volume,
Prev/Next, menu/outline, progress, completion status, and a resume point.
Navigation respects the manifest's sequencing rules (locked next until a slide completes,
if the package says so) — with an "unlock navigation" toggle for reviewers.

### Editor View
The same playback, plus an inspector built around it. Four panels:

1. **Slide navigator** — every slide/SCO/page as a thumbnail list. Click = jump straight
   there, out of order, ignoring lock rules. Shows slide id, title, duration, asset count.
2. **Audio panel** — every audio file used on the current slide (and a global list for the
   whole package): filename, duration, size, inline `<audio>` player, download button,
   and a "replace audio" slot.
3. **Text panel** — all readable text on the slide, extracted and listed. Each string is
   **editable in place**; edits write back into the package model and update the live
   iframe immediately so you see the result without a rebuild.
4. **Interactions panel** — the "what does this button do" panel. For every interactive
   element on the slide: its label, its type (button / hotspot / next / jump / audio
   trigger / quiz option), **where it is** (hover the list item → the element is outlined
   on the slide itself), and **what it does** (jump to slide 7, play audio_04.mp3, show
   layer "Feedback-Correct", submit answer, open URL). Clicking a list row can fire the
   action so you can test it.

Plus: **Export** — rebuild the `.zip` with the edits applied, byte-identical for untouched files.

## 4. How the hard parts get solved

- **Serving the unzipped course.** A Service Worker registers a virtual scope
  (`/__scorm/<id>/...`) and answers every request out of the in-memory file map.
  That keeps relative paths, `XMLHttpRequest`, and `fetch` inside the course working
  untouched. Fallback when SW is unavailable: blob-URL rewriting of HTML/CSS references.
- **SCORM runtime.** A tiny in-memory LMS shim exposes `window.API` (1.2) and
  `window.API_1484_11` (2004) on the parent window so the SCO's API discovery walk finds it.
  CMI data persists in `localStorage`, keyed per package hash, and is visible as a live
  data-model table in Editor View.
- **Reading the slides and triggers.** Three layers, best-effort, in order:
  1. Authoring-tool-aware parsers for the common exports (Articulate Storyline / Rise,
     Adobe Captivate, iSpring, Camtasia) which ship their structure as JSON/XML.
  2. Generic DOM walk of the loaded slide for text nodes, `<audio>`/`<video>`, and
     elements with handlers or `role="button"`.
  3. Runtime instrumentation — wrap `addEventListener` inside the iframe to record what
     is actually bound where.
  Anything that cannot be confidently resolved is shown as "unknown action" rather than guessed.
- **Highlighting position.** Element bounding boxes are read from inside the iframe and
  drawn on an overlay layer aligned to the iframe — nothing is injected into the course DOM.

## 5. Out of scope for v1

Server-side LMS reporting, xAPI/cmi5, multi-user, authoring new slides from scratch,
timeline/animation editing, video re-encoding.

## 6. Next step

Hand `docs/gemini-prompt.md` to Gemini as the build prompt.
