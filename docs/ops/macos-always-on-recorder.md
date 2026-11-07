# Always-On macOS Recorder — Deployment Runbook

> **Status**: Reference runbook, verified on 2026-11-07
> **Reference host**: `FatBoi.local` (tailnet `fatboi`), screenpipe 0.4.52
> **Companion**: [`workboi-bringup.md`](./workboi-bringup.md) — replicating this on a second machine

This documents a real, running deployment: what is configured, why each choice was made, what has
actually broken, and how to verify health. It intentionally records the failure modes, because they
account for most of the "screenpipe isn't recording my meetings" reports.

## 1. What runs

The recorder is **not** started by hand and not the inbox app. It is a LaunchAgent running the
packaged CLI:

```
/Users/chris/Applications/ScreenpipeRecorder.app/Contents/MacOS/ScreenpipeRecorder record \
  --data-dir ~/.screenpipe-recorder --port 3030 --listen-on-lan --api-auth true \
  --retention-days 14 --retention-mode media --disable-telemetry \
  --use-system-default-audio false \
  --audio-device "Logitech BRIO (input)" --audio-device "System Audio (output)" \
  --experimental-coreaudio-system-audio \
  --experimental-meeting-piggyback false
```

> As of 2026-11-07 the EMEET PIXY and fifine microphones were removed from this host — they move to
> WorkBoi's dock (§4). Two devices is the intended set here: one room mic plus system audio.

The launcher resolves to `@screenpipe/cli-darwin-arm64` (version `0.4.52`). **This is a packaged
build, not a build of this repo** — its supported flags are the authority, not the source tree:

```bash
/Users/chris/Applications/ScreenpipeRecorder.app/Contents/MacOS/ScreenpipeRecorder record --help
```

### Job table

| LaunchAgent | Schedule | Purpose |
| --- | --- | --- |
| `ai.screenpipe.recorder` | `KeepAlive` | capture (audio + vision), API on `:3030` |
| `ai.screenpipe.budget` | daily 04:15 | media age prune + 24 GB media cap |
| `ai.screenpipe.visionretention` | daily 04:45 | prune vision rows older than 14 days |
| `ai.screenpipe.storagealert` | hourly, `RunAtLoad` | warn at 20 GB free, escalate at 10 GB |

`launchctl list | grep screenpipe` should show the recorder with a **PID**. The three maintenance jobs
show `-` in the PID column and `0` as last exit code — that is correct, they are periodic.

**Editing a plist requires a reload.** `launchctl kickstart` does not pick up argument changes:

```bash
launchctl bootout   gui/$(id -u)/ai.screenpipe.recorder
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.screenpipe.recorder.plist
```

Use `python3`/`plistlib` for array edits. `PlistBuddy` prints array entries without an index column,
so index arithmetic against its output is unsafe and will silently corrupt `ProgramArguments`.

## 2. Storage model

Retention is split deliberately, because the built-in knobs do not do what their names suggest.

- **`--retention-mode media`** prunes mp4 media only. **Do not switch to `all`.** `all` routes through
  `delete_time_range_batch`, which also runs `DELETE FROM audio_transcriptions` — it would erase
  meeting transcripts, which are the entire point of the deployment.
- **Vision rows** (biggest space consumer: `elements` was 9.86M rows for ~2 weeks) are pruned by
  `~/.local/bin/screenpipe-vision-retention.sh`. It keeps anchor frames still referenced by
  `elements_ref_frame_id`, runs FTS integrity checks, and leaves audio tables untouched.
- **Media cap** (24 GB) is enforced by `~/.local/bin/screenpipe-storage-budget.sh`.

### Compaction

The database uses SQLite `auto_vacuum=OFF`, so deleted rows leave free pages that are reused but never
returned to the filesystem. After the first vision purge the file stayed at 9.2 GB with only ~89 MB of
live data.

**Compaction procedure** (measured: 9,248 MB → 70 MB in ~3 minutes):

1. Ask the engine for `GET /data/backup?path=…` to produce a compacted copy. This is `VACUUM INTO`.
2. Confirm the copy's size and row counts, then swap it in.

Two constraints make this awkward and both are load-bearing:

- **`sqlite3` CLI cannot vacuum this database.** It fails with `no such function: vec_length`, because
  `speaker_embeddings` has a `vec_length` CHECK constraint and sqlite-vec is statically linked into the
  Rust binary. Only the engine can vacuum.
- **The macOS engine opens the DB with the `unix-excl` VFS** — the engine log says
  `macOS capture database using unix-excl VFS with a process-exclusive database lock`. While the
  recorder runs, *any* external `sqlite3` access fails with `database is locked (5)`, including
  `-readonly`. Destructive DB work therefore requires stopping the recorder, with a trap guaranteeing
  restart.

A `VACUUM` only reclaims **free pages**. A file that was produced by `VACUUM INTO` is already fully
packed, so vacuuming it again returns roughly the same size. Deleting a backup is the only way to
reclaim a backup's space.

## 3. Failure modes actually observed

| Symptom | Log evidence | Cause / fix |
| --- | --- | --- |
| Audio gaps, no transcripts for a window | `audio persistence write failed … cause=No space left on device (os error 28)` | Boot volume hit 0 bytes free. Capture keeps running but every 30 s write is lost. Freeing space is the fix; the alert job exists to catch this *before* it. |
| All inputs stop at once, then resume | `recording for device … exited with error: device disconnected` + `[DEVICE_RECOVERY] detected stale recording handle` | USB/aggregate-device churn (sleep, dock, bus reset). The device monitor self-heals within seconds; no intervention needed. |
| Pruner claims "nothing to prune" while rows exist | `sqlite3` returning `database is locked` and being treated as an empty result | Fixed: the script now distinguishes lock/error from a legitimate empty result and refuses to report success. |
| Cutoff off by the UTC offset | `frames`/`ui_events` store UTC, the script compared against local time | Fixed: compare `substr(timestamp,1,19)` against `date -u -v-14d`. |
| Job exits 78 (`EX_CONFIG`) | plist `ProgramArguments` split a command into an executable named `record` | Fixed: one executable + discrete argument entries. |

Alerting was also verified end-to-end (a real notification was posted at the 10 GB floor), including
debounce: one alert per condition, re-alert after 24 h, state cleared when free space recovers.
Note the cosmetic defect that free space under 1 GB prints as `0GB` instead of MB.

## 4. Audio lessons

Audio is the part that matters and the part that is easiest to get subtly wrong.

- **`--use-system-default-audio false` is required** on this host. With it enabled, capture followed
  whatever the OS default happened to be, which changed when headsets connected and produced silent
  or duplicated streams. Explicit `--audio-device` entries are stable.
- **Redundant microphones duplicate every sentence.** Until 2026-11-07 this host captured three mics
  aimed at the same room, and identical speech produced two transcripts with different wording
  (`wake on land` / `wake online`, `Chris Roddy` / `Chris Running`). BRIO + System Audio is the correct
  set here. Counts from a day with all three, which show the duplication:

  | Device | Transcriptions | Notes |
  | --- | --- | --- |
  | Logitech BRIO | 33,833 | room mic, dominant |
  | System Audio | 16,151 | only transcribes when something is playing |
  | fifine Microphone | 244 | near-silent, low gain |

  Same speech is therefore transcribed 2–3× with different wording. For meeting minutes this is noise;
  the fix is to keep one room mic plus System Audio, and drop the redundant inputs.
- **`System Audio (output)` going quiet is normal.** Health reports it `inactive (last activity: Ns ago)`
  when nothing is playing; it is not an error.
- **VAD rejects most chunks by design** (`vad_rejected` ≈ 70 % on a quiet room). A low
  `vad_passthrough_rate` is not a fault.
- `krisp microphone (input)` appears in `/audio/list` but is not configured — that is the virtual
  device from Krisp, not an input that is being recorded.

## 5. Meeting detection — current status: **not firing**

`/meetings/status` returns `{"active":false,"detectionSource":null}`. The `meetings` table contains
**exactly one row in its entire history**:

```
id=1  Zoom  2026-10-30T16:33:37 → 16:56:12  detection_source=audio_process
```

That is the key diagnostic. Detection is not "sometimes flaky" — it worked once, via the
`audio_process` source, and has produced nothing since `--experimental-meeting-piggyback false` was
added to the recorder arguments.

Details that matter when diagnosing:

- **Log silence is expected, not proof of health.** The v2 detector logs per-scan results at `debug!`
  (`meeting scanner: pid=… signals=… in_call=…`). At the default level you only see the one-shot
  startup lines (`meeting detector enabled`, `loop started (profiles=13)`).
- The v2 detector **scans the accessibility tree of running meeting apps** for leave/hangup controls.
  Mute buttons alone do not count, because they also appear in pre-join lobbies and chat apps.
- It can only fire if a **meeting app is actually running with a call window**. On the check that
  surfaced this, only `TeamsWidgetExtension` and `com.microsoft.teams2.agent` were alive — not the
  Teams main app — and no Zoom.
- `--experimental-meeting-piggyback false` disables meeting-driven per-process audio capture
  ("smart recording"). The flag's own help says *"Pass false to use the configured capture devices
  throughout meetings"*, so capture stays correct — but the `audio_process` detection source that
  produced the only successful detection is tied to that path.

**Working hypothesis**: disabling piggyback removed the detection source that actually worked on this
host, leaving only AX scanning, which never matches because the apps used for calls do not expose
detectable controls (or do not stay running). Next step is to re-enable piggyback
(`--experimental-meeting-piggyback true` or omit the flag, which defaults to enabled), keep the explicit
`--audio-device` list, and confirm during a real call that `meeting_detected` flips to `true` and a row
appears in `meetings`. `--ignored-meeting-apps` can exempt specific services if detection over-triggers.

**Status (2026-11-07 16:30)**: the flag was set back to `true` and the recorder reloaded (healthy,
2 audio devices, detector loop started, `profiles=13`). Detection has **not** yet been confirmed: no
meeting app was running at the time, and the only evidence that counts is a new `meetings` row with a
non-null `detection_source` during an actual call. `/meetings/status` still reports
`detectionSource: null` immediately after the reload, which is expected while idle.

### Calendar-assisted detection is not available headless

A reasonable assumption is that meetings can be picked up from the Mac calendar. On this deployment
that path is closed, and it is worth knowing before spending time on calendar settings:

- The detector consumes a `calendar_events` broadcast (`subscribe_to_event::<Vec<CalendarEventSignal>>`
in `crates/screenpipe-engine/src/meeting_detector.rs`) and only enriches a meeting it has *already*
detected from app signals. It is not a standalone trigger.
- The running CLI binary does **not** serve the calendar route. `GET /calendar/events` returns **404**
for every parameter combination, even though `crates/screenpipe-engine/src/connections_api.rs` defines
that route. The route exists in this repo's source, not in packaged build 0.4.52.
- `/connections` lists connectable services (telegram, …) but no calendar provider.

So calendar enrichment requires the desktop app to be running and publishing those events; a headless
LaunchAgent recorder will not receive them. The calendar data itself is present — the Mac has a Google
account for `chris@chrisrodney.com` syncing calendars, and Calendar.app is readable via AppleScript —
but no calendar named `KCT` exists on this machine, so the expected KCT calendar is either not synced
here or named differently (the list includes several `Client - …` calendars).

## 6. Health verification

```bash
KEY=$(/usr/libexec/PlistBuddy -c "Print :EnvironmentVariables:SCREENPIPE_API_KEY" \
  ~/Library/LaunchAgents/ai.screenpipe.recorder.plist)
curl -s -H "Authorization: Bearer $KEY" localhost:3030/health | python3 -m json.tool | \
  grep -E '"status"|audio_status|frame_status|last_audio_timestamp|meeting_detected|device_status_details'
curl -s -H "Authorization: Bearer $KEY" localhost:3030/audio/list
curl -s -H "Authorization: Bearer $KEY" localhost:3030/meetings/status
curl -s -H "Authorization: Bearer $KEY" localhost:3030/retention/status
df -h / && du -sh ~/.screenpipe-recorder
```

The API requires the bearer key and is bound to `--listen-on-lan`, so it is reachable from another
machine on the LAN (`/raw_sql` accepts `SELECT` only, and requires an explicit `LIMIT`). Treat that key
as a credential: it grants read access to everything captured.

The cheapest end-to-end audio check is to confirm a transcription timestamp is recent rather than to
trust `audio_status`:

```sql
SELECT substr(timestamp,1,19) AS ts, device, substr(transcription,1,60) AS t
FROM audio_transcriptions ORDER BY id DESC LIMIT 10
```

## 7. GUI app vs. headless recorder

They are **peers, not client and server**. The Tauri app bundles its own engine and talks to it over
HTTP on `localhost:3030` (`apps/screenpipe-app-tauri/lib/api.ts`, `src-tauri/src/recording.rs`); the
LaunchAgent runs that same engine without any UI. Consequences worth knowing:

- The app always **spawns** its engine (`spawn_screenpipe`) and binds the configured port
  (`server_core.rs` → `bind_listener`). It does not attach to an engine that is already running.
- Both need the database, and the macOS engine takes an **exclusive** `unix-excl` lock. A second
  process against the same data dir fails with `database is locked (5)`.
- So for a given data directory, run **one** engine: either the LaunchAgent (always-on capture, no UI)
  or the desktop app (UI, which owns capture while it runs). Launching the app while the agent holds
  the DB does not "connect" to it — it conflicts with it.

This is why nothing appeared in the app while capture was clearly working: the agent was the engine,
and the app was not running.

### Managing the headless recorder

`~/.local/bin/screenpipe-ctl.sh` wraps the operations that matter, without needing the GUI:

```
screenpipe-ctl.sh status     # pid, uptime, port, db size, data dir, free space
screenpipe-ctl.sh health     # status, audio/vision, devices, last frame/audio, transcript counts
screenpipe-ctl.sh devices    # audio devices the engine can see
screenpipe-ctl.sh meetings   # detection state + the last rows in `meetings`
screenpipe-ctl.sh logs 60    # tail today's engine log
screenpipe-ctl.sh start|stop|restart
screenpipe-ctl.sh api        # prints the base URL
```

The API key stays in the plist and is read by the script; it is never printed. Anything the app can do
is an HTTP call to that API, which is also reachable from another machine on the LAN because the
recorder runs with `--listen-on-lan` (treat the key as a credential).

## 8. Calendar reminders and browser-call capture

The engine cannot do either of these on this build, so both are handled by small sidecar jobs that
drive its API. The engine's own limitations, verified on 0.4.52:

- `GET /calendar/events` is **404**; the detector only *enriches* meetings it already found, so there
  is no calendar-driven trigger at all.
- `ui_events.browser_url` has **never been populated** (`SELECT count(*) … WHERE browser_url IS NOT NULL`
  → 0), and frames store only `app_name`/`window_name`. Browser meetings (Meet, Teams-web, Zoom-web,
  Slack huddles) therefore cannot be discovered by the built-in detector on this host.

### The manual meeting API does work

Discovered by probing (note: `OPTIONS` returns 200 for *any* path here — always send a control request
to a bogus path before trusting a probe):

```
POST /meetings/start  {"app":"Google Meet","title":"…","attendees":"…"}
  → {"id":2,…,"meeting_app":"manual","detection_source":"manual"}
POST /meetings/stop   {"id":2}   → closes the row
```

`StartMeetingRequest` is `{app?, title?, attendees?}` and hardcodes `detection_source="manual"`
(`crates/screenpipe-engine/src/routes/meetings.rs`). This is the hook both sidecars use.

### Calendar reminders — `screenpipe-calendar-watch.sh`

Notifies N minutes (default 10) before an event, and flags events carrying a joinable link.

- AppleScript/JXA over Calendar.app is **O(all events)**: measured 91–107 s for a 12-hour window. It is
  therefore cached for 30 minutes rather than polled.
- A Swift/EventKit helper (`sp-calendar.swift` → `sp-calendar`) does the same query in milliseconds, but
  Calendars TCC access was **denied** for the CLI binary (exit 2, `CALENDAR_ACCESS_DENIED`), even
  ad-hoc signed. Grant it in System Settings → Privacy & Security → Calendars to switch the fast path on;
  until then the script logs `cache refreshed via AppleScript (slow path)`.
- LaunchAgent `ai.screenpipe.calendarwatch`, every 5 min, `RunAtLoad`.

### Browser calls — `screenpipe-browser-meeting.sh`

Reads the front Safari tab URL via AppleScript (verified working) and, on
`meet.google.com` / `teams.microsoft.com` / `zoom.us/j` / `webex.com`, calls `POST /meetings/start`
with the tab title, then stops the meeting after two consecutive polls (~40 s) with no meeting URL.
It never touches a meeting it did not start: an already-active engine meeting is recorded as
`external` and left alone. LaunchAgent `ai.screenpipe.browsermeeting`, every 20 s.

Chrome/Arc are not polled because neither is installed here; add their AppleScript blocks if that
changes.
