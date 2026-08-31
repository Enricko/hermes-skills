---
name: web-editor-autosave-debugging
description: Debug JS autosave/save flows when saves 405 or no-op.
version: 1.0.0
tags: [frontend, forms, javascript, autosave, debugging]
---

# Web editor autosave & save-flow debugging

Class of task: custom "autosave + manual Save/Save&Render" flows in server-rendered web apps (Jinja/HTMX-era stacks, no framework). Symptoms like "Not saved: HTTP 405", "save does nothing", or "no indication it worked" usually trace to one of three structural bugs below.

## The three recurring failure modes

1. **DOM clobbering of `form.action`.** If any control inside the form (or associated via `form="edit"` from OUTSIDE it) has `name="action"`, then the JS property `form.action` resolves to that ELEMENT, not the endpoint URL. `fetch(form.action)` coerces the element to the string `[object HTMLButtonElement]`, which the browser resolves as a RELATIVE path against the current page → POSTs to garbage like `/parent/path/[object%20HTMLButtonElement]` → 405.
   Fix: always read `form.getAttribute('action')`.

2. **Submitter loss in intercepted submits.** An autosave pattern (`submit` listener → `preventDefault()` → rebuild body with `new FormData(form)` and fetch) silently DROPS the pressed submit button's name/value. A "Save & Render" button carrying `action=render` degrades to a plain save: server responds 303 OK, chip says saved, render never queues. User sees "nothing happened".
   Fix: capture `event.submitter` and append its name/value to the FormData; if a save is already in flight, re-queue WITH the submitter rather than coalescing into a plain save.

3. **No post-action feedback.** Even when the save+render request succeeds, if nothing re-opens a progress dialog or changes a status chip beyond "saved", users report the feature as broken. After an action-carrying save succeeds, surface state explicitly (re-open the export/progress modal that listens to SSE/polling).

## Debugging path that finds these fast

1. Server access log is ground truth: `journalctl -u <service> | grep POST` shows exactly what URL the client sent. `[object ...]` in a path = DOM clobbering; repeated plain saves after a Save&Render press = submitter loss.
2. Grep templates for buttons with `name="action"` (or names colliding with any property you read off the form) and for `form="id"` associations from outside the form element.
3. Reproduce with curl but note `-L` re-POSTs after 303 redirects and fakes a second failure — test without `-L` first.

## Pitfalls

- Mobile browsers cache aggressively: bump cache-buster query versions (`app.js?v=...`) on EVERY js/css change or fixed bugs reproduce on phones.
- Off-canvas drawers (mobile inspector panels): tab clicks must also toggle the drawer-open class AND the panel needs a solid background, scrim, and content-sized height, else users report "invisible layout" or dead empty space.

## Related
- Project-specific architecture and verification workflow live in the user-owned `clipper-apps-rebuild` skill (needs `hermes curator adopt` before agents may update it).
