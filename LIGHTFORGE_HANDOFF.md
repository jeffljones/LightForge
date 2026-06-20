# LightForge — Project Handoff

A phone-first strength-training tracker for a volleyball athlete. Single self-contained
HTML file, no build step, deployed on Firebase Hosting. This doc is the starting context
for a Claude Code session; the actual app is in `lightforge.html` (which becomes
`public/index.html` on deploy).

---

## 1. What it is

A workout logger with the training program baked in. Two day-templates (Day A lower /
Day B upper), each a sequence of exercises grouped into blocks. You log sets per exercise,
it shows last time + your best so you know what to beat, flags PRs, saves automatically,
and keeps a session history. Built to be used on a phone at the gym between sets.

The companion (separate) project is a volleyball app called **SIDEOUT** (Firebase project
`sideout-vb`, live at `sideout-vb.web.app`). LightForge is intentionally its **own**
Firebase project — they do not share data or code.

---

## 2. Tech stack

- **One file.** `lightforge.html` contains all HTML, CSS, and JS. No framework, no bundler,
  no npm. Vanilla DOM JS. This is a deliberate constraint — keep it a single static file
  unless there's a strong reason not to.
- **Fonts:** Google Fonts — `Oswald` (display + all numbers, scoreboard feel) and
  `Hanken Grotesk` (body/UI), loaded via `<link>`.
- **Backend:** Firebase, loaded as **compat** SDK v`12.11.0` from gstatic CDN
  (`firebase-app-compat`, `firebase-auth-compat`, `firebase-firestore-compat`).
- **Auth:** Firebase **Anonymous** auth. No login screen. Data is tied to the browser's
  anonymous UID — intentional ("reliable, one device is fine"). Clearing site data or
  switching devices resets the link (data stays in Firestore but unlinked).
- **DB:** Cloud Firestore.
- **Hosting:** Firebase Hosting → `lightforge-jj.web.app`.

Firebase project id: **`lightforge-jj`**. The web config is already pasted into the
`window.FB_CONFIG` block near the top of the `<script>`.

---

## 3. Storage architecture (read this before touching data)

All persistence goes through three async helpers so the backend is swappable:

```
kvGet(key)        -> string | null
kvSet(key, val)   -> bool        (val is always a JSON string)
kvDel(key)
```

They write to **Firestore if configured + signed in, else `localStorage`** (device-only
fallback so the file still works before deploy). The whole rest of the app is
backend-agnostic and only talks to these three functions.

**Firestore layout:** ONE shared document at `logs/{SHARED_ID}` that *every* device for this
user reads and writes (no per-device login). `SHARED_ID` is a fixed constant in the app. The
kv keys are stored as *fields* on that one doc, each holding a JSON **string**:

- field `courtlog-v1`        → `JSON.stringify({ sessions: [...] })`
- field `courtlog-draft-v1`  → `JSON.stringify({ day, today, prep })`
- field `lightforgeLive`     → denormalized read-only feed (the monitor link)

> **Why shared, not `logs/{uid}`:** anonymous auth gives each device a *different* UID, so the
> original per-UID layout couldn't show the same data on phone + desktop. We still sign in
> anonymously (so writes are authenticated), but point all devices at one fixed doc. Tradeoff:
> security is "obscure doc id" rather than per-user isolation — fine here (non-private data,
> single user). To go private later, switch to real auth and key the doc by `uid` again.

> Gotcha: the session/draft key constants are still named `courtlog-*` (the app was renamed
> Court Log → LightForge but the storage keys were left alone so existing data isn't orphaned).
> Don't rename them without a migration. Constants: `STORE_KEY`, `DRAFT_KEY`, `SHARED_ID`.

> Recovery: on first cloud sync, if the shared doc has never held a `courtlog-v1` field but the
> browser has local sessions, the app lifts them into the cloud (`migrateLocalToCloud`). Guarded
> on field-absent so an intentional "Clear all" is never undone. History also has Import (merges
> a backup `.json` by session id) as a manual restore path.

A one-shot snapshot cache (`_snapCache`) avoids a second Firestore read on startup; `kvSet`
/`kvDel` keep it in sync.

---

## 4. Data model

**Session** (appended to the `sessions` array, newest last):
```
{ id: <Date.now()>, day: "A" | "B", date: <ISO string>,
  entries: [ { key, name, type, sets: [...] } ] }
```

**Set shape depends on exercise `type`:**
- `weight` → `{ weight, reps }`
- `reps`   → `{ reps }`
- `time`   → `{ seconds }`
- `check`  → not logged as sets (prep/mobility; tracked only in the live draft via `prep`)

Values may be strings (from inputs) or numbers (from steppers); `metric()` coerces with
`Number()`. PR/best logic lives in `metric()` (weight ranks first, reps as tiebreak),
`bestSet(key,type)`, and `lastEntry(key)`.

**Draft** (the in-progress, unsaved session) = `{ day, today: {A:{}, B:{}}, prep: {} }`,
where `today[day][exerciseId]` is an array of set objects. Autosaved on every edit
(600ms debounce) and on `visibilitychange`/`pagehide`/`beforeunload`. Cleared on
"Save session".

---

## 5. The PROGRAM object

`PROGRAM = { A: {title, blocks}, B: {title, blocks} }`. Each block = `{name, items}`;
each item (exercise) =
```
{ id, key, name, type, scheme, note? }
```
- `id` is unique per slot; `key` is the **history key**. Exercises that repeat across both
  days (Pallof press, suitcase carry) share a `key` (`"pallof"`, `"carry"`) so their
  last/best/PR history merges across days.
- Block order is meaningful: **Prep → Power → Strength → Accessory → Core** (warm/jump work
  before heavy lifts before isolation/core).
- To add/change an exercise, edit this object only — rendering, logging, history, and PR
  detection all derive from it.

Current program: **Day A** ankle prep, pogo hops, RDL, leg press, split squat/step-up,
leg extension, leg curl, standing + seated calf raises, Pallof, carry. **Day B** thoracic prep, med-ball,
chest press, lat pulldown, row, shoulder press, reverse fly, Pallof, carry.

---

## 6. Key functions / behaviors

- `init()` — `initFirebase()` → `loadData()` → `loadDraft()` → `render()`, then sets the
  status line and attaches the leave-the-page draft-flush listeners.
- `render()` is **wrapped** at the bottom to also call `tagCards()`; cards are looked up by
  a `__exid` property set on each DOM node (see `tagCards`/`findCard`).
- `setEditor` / `stepper` / `wireSteppers` — the per-set logging UI (thumb-friendly ± steppers
  with manual entry). `livePR` toggles the PR badge live without re-rendering/collapsing.
- `saveSession` — filters empty sets, computes whether it beat prior bests, appends the
  session, persists, clears the draft.
- `renderHistory` — past sessions (newest first), plus **Export a backup** (downloads
  `lightforge-backup-<date>.json`) and **Clear all data**.
- `statusLine()` — the header subtitle is the live source-of-truth indicator:
  "Synced to Firebase" / "Saved on this device" / "Logs won't save here".

---

## 7. Deploy + Firebase setup

Two prerequisites in the **lightforge-jj** console or it silently falls back to device-only:

1. **Authentication → Sign-in method → Anonymous → ENABLE.** (Required — without it the app
   gets `auth/admin-restricted-operation` on sign-in and silently falls back to device-only.
   Anonymous auth is *not* a login screen; it's invisible.)
2. **Firestore → create (Production) → Rules → publish:**
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /logs/{docId} {
      allow get:   if true;                  // public read — live monitor link + the shared log
      allow list:  if false;                 // but NOT enumerating the collection
      allow write: if request.auth != null;  // any signed-in (anonymous) device can write the shared doc
    }
  }
}
```
> These rules back the **shared-log** model (§3): all of this user's devices sign in anonymously
> and read/write one fixed doc `logs/{SHARED_ID}`. The write rule is `request.auth != null`
> (any signed-in device) rather than `request.auth.uid == docId`, because with a *shared* doc
> the id isn't a uid. Reads are public so the monitor link works. Tradeoff: anyone who knows the
> obscure doc id (it ships in the client JS) and signs in anonymously could write it — acceptable
> for a single-user, non-private training log. To kill all writes instantly, disable Anonymous
> auth. To make it private again: switch to real auth and use `allow write: if request.auth.uid
> == docId` with the doc keyed by uid.

### Deploy — automated (GitHub Actions)

This repo auto-deploys: **every push to `main` publishes to `https://lightforge-jj.web.app`**
via `.github/workflows/deploy.yml`. The workflow stages the single file as
`public/index.html` (per `firebase.json`) and deploys the `live` channel of project
`lightforge-jj` (pinned in `.firebaserc`, so it can't go to sideout-vb). No manual step per
change — edit `lightforge.html`, push to `main`, done.

**One-time setup** to arm it: add a repository secret named
`FIREBASE_SERVICE_ACCOUNT_LIGHTFORGE_JJ` (a Firebase service-account key JSON).
GitHub → repo **Settings → Secrets and variables → Actions → New repository secret**; the
JSON comes from Firebase console → **Project settings → Service accounts → Generate new
private key**. Until the secret exists, the workflow runs but skips the deploy (stays green).
You can also trigger a deploy by hand from the **Actions** tab (`workflow_dispatch`).

### Deploy — manual fallback (first-time project setup, or if Actions is off)
```
firebase use lightforge-jj      # confirm correct project — NOT sideout-vb
firebase init hosting           # public dir: public; SPA: No; GitHub: No
firebase deploy --only hosting  # -> https://lightforge-jj.web.app
```

Then on iPhone: open the URL in Safari → Share → Add to Home Screen (the PWA meta tags are
already in `<head>`).

---

## 8. Things worth knowing before extending

- **Single-file is intentional.** If a feature pushes toward a build step or multiple files,
  flag the tradeoff rather than silently restructuring.
- **No `localStorage` inside a Claude.ai artifact preview** (it fails there) — but this app is
  deployed standalone, so the localStorage fallback is fine in the real world. Just don't
  expect the in-chat preview to persist.
- **The kv layer is the seam.** Anything storage-related (sync, multi-device, accounts, a
  read-only JSON endpoint for external analysis) should go through or extend `kvGet/Set/Del`.
- A common future ask: a **read-only JSON endpoint** (Cloud Function) so the log can be pulled
  for analysis without manual export. Writes must stay locked to the authenticated user.
- Bodyweight/calorie tracking was deliberately left out — it's a training logger, by choice.

---

## 9. Features

### Done

- [x] **No duplicate sessions / smart save.** Saving collapses to **one workout per
      day-template per calendar day**. If a session for the current day already exists,
      a save *updates it in place* (keeps the original `id` + `date`) instead of appending a
      second one. The save button reflects this: it reads **"Update workout"** when today's
      session exists, **"Save session"** otherwise, and resets automatically on a new calendar
      day. Logic: `todaySessionIndex(day)` + `sameLocalDay()`; PR detection excludes the
      session being replaced from its baseline (`bestSet(key,type,pool)`).
- [x] **Delete a single session.** Each session in History has a "Delete this session" button
      (confirm-gated). Removes exactly one match by `id` (so it cleans up an existing
      duplicate without nuking both).
- [x] **Live read-only monitor feed (for Claude/real-time feedback).** On every edit and save
      the app writes a denormalized, self-describing snapshot to a new Firestore field
      `lightforgeLive` (`LIVE_KEY`) via the kv layer — current in-progress sets plus recent
      history, with exercise names, targets, last and best. History shows a **"Live monitor
      link"** (copy button) pointing at a masked Firestore REST URL:
      `…/documents/logs/<uid>?key=<apiKey>&mask.fieldPaths=lightforgeLive`. Paste that into a
      Claude chat and it can read the workout as it happens. **Requires the public `get` rule
      in §7** (until those rules are published + the app redeployed, the link returns
      `PERMISSION_DENIED`). Helpers: `buildLiveFeed()`, `writeLive()`, `monitorURL()`.

### Manual steps to make the monitor link live

1. Publish the updated Firestore rules in §7 (adds `allow get: if true`).
2. Redeploy hosting (§7) so the new `lightforge.html` is live.
3. Open the app → **History** → copy the **Live monitor link** → paste into a Claude chat.
   (The `lightforgeLive` field is created the first time you edit/save after deploying, so
   log at least one set first.)

### Possible next

- [ ] **Edit an older session** (not just today's): a date picker on save + loading a past
      session back into the editor. Today's workout is already editable via the update-in-place
      save; this would extend it to arbitrary past dates.

---

## 10. Suggested opening prompt for the Claude Code session

> I'm working on **LightForge**, a single-file phone-first workout tracker
> (`lightforge.html`, vanilla JS + Firebase compat SDK, anonymous auth, Firestore at
> `logs/{uid}`, deployed to `lightforge-jj.web.app`). Read `lightforge.html` and
> `LIGHTFORGE_HANDOFF.md` first to understand the architecture and data model, then help me
> implement the features in section 9. Keep it a single static file unless we explicitly
> decide otherwise, and route any storage changes through the existing `kvGet/kvSet/kvDel`
> layer.
