---
name: web-form-submit-pipeline
description: Debug form submits gone silent under autosave/fetch JS.
version: 1.0.0
tags: [frontend, javascript, html-forms, debugging]
---

# Form submit pipelines with JS interception

Class of bug: an app where form submits are intercepted by JS (autosave,
SPA-style saves) and the user-visible symptom is "I clicked Save/Render and
nothing happened" or a 405 to a garbage URL.

## Diagnostic order
1. **Server logs first.** Grep access logs for the POSTs arriving while the
   user reproduces. The exact path + status tells you which layer failed:
   - Garbage path like `[object HTMLButtonElement]` → DOM clobbering or an
     element stringified into a URL.
   - Correct path but wrong payload effect → submitter value was dropped.
   - No POST at all → JS never fired; check cache (304 Not Modified means the
     browser kept stale JS — bump the cache-buster query).
2. **Check for named controls shadowing form IDL properties.** Any control
   with `name="action"` (or `name="method"` etc.) inside/associated with the
   form makes `form.action` return that ELEMENT, not the URL string. Always
   read endpoints via `form.getAttribute('action')`.
3. **Submit handlers that preventDefault + re-send FormData drop the
   submitter's name/value.** A "Save & Render" button (`name=action`,
   value=render) silently degrades to a plain save — chip says saved, no job
   created server-side, no error anywhere. Pass `event.submitter` through and
   append it: `body.set(submitter.name, submitter.value)`.
4. **event.submitter is unreliable on mobile Chrome** for `[form]`-associated
   buttons inside `<dialog>`. Belt-and-suspenders: track the pressed button
   via capture-phase pointerdown/click, AND give critical buttons their own
   click handler that builds the request directly instead of relying on form
   submit semantics.
5. **Reproduce server-side before blaming the client**: curl the endpoint
   with the full field set including the action value; confirm 303/job-row/
   artifact appears. This splits backend vs frontend cleanly.
6. **Feedback loop matters**: if success has no visible signal beyond a tiny
   status chip, users report "nothing happened". After an intercepted submit
   with side effects (render/export), re-open the progress dialog or surface
   state explicitly.
7. **Verify fixes reached the browser**: bump inline cache-buster versions on
   every js/css edit, then confirm over curl what the served file contains;
   304s can mask two rounds of fixes.

## Off-platform variant
Mobile drawer/off-canvas panels: tab clicks must programmatically open the
drawer (add the open class), the drawer needs a solid background + scrim +
its own close control, and panels must size to content or dead space shows
below the last section.

## Worked example
See `references/clipper-studio-case.md` for the full case study these rules
came from (FastAPI/Jinja studio editor, three stacked bugs producing one
symptom).
