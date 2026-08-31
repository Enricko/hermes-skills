# Case study: Clipper-Apps studio "Save & Render did nothing" (2026-08-24)

Stack: FastAPI + Jinja + vanilla JS, mobile Chrome client. One symptom, three
stacked bugs found in sequence.

## Symptom chain
1. Save produced `POST /projects/<uuid>/studio/[object HTMLButtonElement]` →
   405. Cause: render buttons carry `name="action"` + `form="edit"`; the IDL
   property `form.action` resolved to the BUTTON element (DOM clobbering), and
   autosave's `fetch(form.action)` coerced it into a URL string.
   Fix: `form.getAttribute('action')`.
2. Next round: POST hit the right endpoint with 303, but no `render_clip` job
   row was ever created. Cause: the autosave submit handler called
   `preventDefault()` then re-sent `new FormData(form)` — dropping the
   submitter's `name=action value=render`. Server saw a plain save.
3. Fix attempt using `event.submitter` still failed on Android Chrome:
   submitter is unreliable for `[form]`-associated buttons inside `<dialog>`.
   Final fix: capture-phase pointerdown/click tracking of the pressed button
   PLUS direct click handlers on the export-dialog render buttons that append
   `action=render` to the FormData themselves.

## Verification method that cracked it
- `journalctl -u clipper-web | grep POST` while user reproduced → showed exact
  paths and that all POSTs were plain saves.
- MySQL check (`jobs` / `renders` tables) → no render job = client never sent
  action=render; backend exonerated.
- curl POST with full field set including `action=render` → 303 +
  `?render=queued`, job went pending→processing→100%, 22 MB MP4 appeared in
  storage renders dir. Backend proven good; problem purely client-side.
- Cache trap: browser served stale autosave JS as `304 Not Modified` after a
  fix because the version query hadn't been bumped — masked one entire fix
  round. Bump inline `?v=` versions on every JS edit and verify served bytes.

## UX lesson
After an intercepted submit whose effect is asynchronous (render/export), a
tiny status chip is not enough feedback. Re-open the progress dialog on
success so the user sees Preparing→Processing→Rendering stages immediately,
or they report "nothing happened" even when it worked.
