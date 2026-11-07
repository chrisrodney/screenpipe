# screenpipe 0.4.52 — Verified Capabilities

> **Source of truth**: the installed binary, not the fork's source tree. Everything here was checked on
> 2026-11-07 against `@screenpipe/cli-darwin-arm64` **v0.4.52** running headless on `FatBoi`.
> The desktop app is a *different product line* (v2.7.145), so app docs do not describe the CLI.

## 1. Two products, not two halves of one

| | CLI | Desktop app |
| --- | --- | --- |
| Version installed / latest | **0.4.52** / 0.4.58 (Nov 6) | — / **2.7.145** |
| Role | scriptable engine + HTTP API | engine + timeline UI, chat, settings, connections |
| UI | none | full app |

Docs put it plainly: the CLI "does not give you the desktop timeline UI by itself", while the app
"manages recording, settings, search, scheduled tasks, and AI connections — no terminal needed".

**They do not cooperate on one recording.** The app spawns its own engine and binds the same port; the
macOS engine holds an exclusive `unix-excl` DB lock. Same data dir = conflict, not sharing. Running both
on one machine means two independent recorders on different ports and data dirs.

## 2. CLI surface (from `screenpipe --help`)

```
record  status  profile  search  team  pipe  view  audio  vision  sync  mcp  agent
setup   service connection vault install login logout whoami survey doctor diagnose
auth    db      storage  backup export  help
```

Worth knowing these exist, because they replace hand-rolled scripts:

- `service install|status|uninstall` — built-in launchd service (`com.screenpipe.agent`). Currently
  **not** installed here; the recorder runs from a hand-written LaunchAgent instead.
- `doctor` — permissions + ffmpeg + port readiness.
- `diagnose` — bundles logs and can open a support ticket (`--dry-run` saves locally).
- `db` — corruption repair and free-disk cleanup. `backup` — WAL checkpoint + snapshot.
- `export --meeting-id` — export a meeting (or any time range) to a single MP4.
- `storage` — offline frame lifecycle. `vault` — encrypt data at rest.
- `pipe` — scheduled AI agents over the data. `mcp`/`agent` — expose it to Claude/Codex/OpenClaw/Hermes.
- `connection` — 71 integrations (see §4).

Recording flags relevant to a meeting box: `--keep-computer-awake`, `--transcription-mode`,
`--deepgram-api-key`, the PII redaction family, `--ignored-urls`/`--url-block`, `--ignored-windows`,
`--ignored-meeting-apps`, `--retention-days`, `--listen-on-lan`, `--api-auth`.

## 3. Permissions: what `doctor` checks, and what it doesn't

```
permissions:
  screen recording: ok        microphone: ok
  accessibility: ok           system audio capture: ok
```

All four pass. Note what is **absent**: `doctor` never mentions the calendar. Calendar is not a CLI
permission concern — it is a *connection* (or an app-level entitlement).

## 4. Calendar: connections exist, but the CLI only wires one of them

The engine's `/connections` catalog (71 entries) includes:

```
google-calendar   Apple Calendar (apple-calendar)   ICS Calendar (ics-calendar)
calcom            calendly                          zoom   teams   microsoft365
```

All currently `connected: false`. But the CLI can only set some of them:

- `connection set google-calendar key=value …` — **recognised** ("google-calendar is not connected",
  with a hint).
- `connection get apple-calendar` / `ics-calendar` — **`Error: unknown integration`**. They appear in
  the API's advertised catalog but are not wired into the CLI's connection setter.

So the headless path to calendar context is a **Google Calendar connection**, not the Mac's local
calendars.

### What calendar actually contributes

Per the meeting-intelligence docs, calendar adds **attendee names, meeting title, and the time window**
— used mainly for speaker labelling ("calendar context is especially strong for 1:1 meetings"). The
setup checklist is: choose audio devices → *optionally* connect Google Calendar for attendee-aware
speaker naming → **open the meeting app normally** → let it record.

It is not documented as a trigger. Detection comes from the meeting surface being present (app window /
browser tab) plus audio.

### The known packaging bug that explains a silent denial

From the official changelog: *"Apple Calendar connection works again on signed builds — the production
bundle was missing the macOS calendar entitlement, so EventKit was denied before the system prompt could
fire and screenpipe never appeared under Privacy & Security → Calendars."*

That is exactly the failure observed here — instant `CALENDAR_ACCESS_DENIED`, no prompt, no entry in
Privacy & Security — for both the app path and a locally compiled EventKit helper. It is an entitlement
problem, not a permission the user forgot to grant.

## 5. Data-directory trap

```
$ screenpipe status
database   /Users/chris/.screenpipe/db.sqlite
storage    0 B total · 0 B media · 0 B database
```

That is the **default** dir. This recorder runs `--data-dir ~/.screenpipe-recorder` (**7.4 GB**, db
516 MB), so `status`/`doctor` without `--data-dir` report an empty installation. Always pass it:

```
screenpipe status --data-dir ~/.screenpipe-recorder --port 3030
```

## 6. Meeting APIs (documented and present)

```
GET  /meetings?limit=20&q=…            list / search title, attendees, notes
GET  /meetings/status                  active meeting, detection source
POST /meetings/start  {"app":…,"title":…}   → detection_source="manual"
POST /meetings/stop   {"id":…}
POST /meetings/merge  {"meeting_ids":[…]}   (docs) — binary uses a different body; verify first
```

The docs show `POST /meetings/start` with no body; **0.4.52 requires a JSON body** (`{}` is enough,
`400 EOF` otherwise). Trust the binary over the docs on details like this.

## 7. What this means for our setup

- Calendar-aware reminders and browser-call capture are **not** available from the engine on this build
  (`/calendar/events` → 404; `browser_url` never populated), which is why they are implemented as
  sidecars driving the manual meeting API.
- To get real calendar context headlessly, the move is a **Google Calendar connection**, not fiddling
  with macOS Calendar permissions.
- To get the built-in UI — timeline, meeting list, notifications — the **desktop app** is required, and
  the headless `ai.screenpipe.recorder` agent must be stopped for that data dir to avoid the DB lock.
- Upgrading the CLI (0.4.52 → 0.4.58) is available and may change detection behaviour; re-verify with
  `doctor` + `meetings/status` after any upgrade.
