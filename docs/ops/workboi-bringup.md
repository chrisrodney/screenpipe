# WorkBoi Bring-Up — Second Recorder Host

> **Status**: Blocked on remote access (see §1)
> **Target**: `workboi.local` / `192.168.0.216`, user `chris`
> **Reference**: [`macos-always-on-recorder.md`](./macos-always-on-recorder.md) — the proven config on FatBoi

WorkBoi is the intended control surface for this deployment. This is the checklist for making it a
first-class recorder host using the configuration and lessons from the reference machine, rather than
re-deriving them.

## 1. Access — currently blocked

From FatBoi, the machine answers at the network layer but not at the SSH layer:

| Check | Result |
| --- | --- |
| `ping 192.168.0.216` | 2/2 packets, 0 % loss (~700 ms, high for LAN — Wi-Fi power save) |
| `arp -a` | MAC `12:52:ca:21:b7:22` on `en0`/`en1` |
| mDNS `workboi.local` | resolves (IPv6 link-local + ULA) |
| SSH config | `Host workboi` → `HostName 192.168.0.216`, `User chris`, `IdentityFile ~/.ssh/id_ed25519_workboi`, `IdentitiesOnly yes` |
| port 22 | **closed / timed out** |
| ports 5900, 3283 | closed |

Ping and ARP proving reachable while 22 times out means **Remote Login is off** on WorkBoi (the key
and `authorized_keys` cannot be the problem yet — the TCP handshake never happens). On WorkBoi:

1. System Settings → General → Sharing → enable **Remote Login**, allow user `chris`.
2. Confirm from FatBoi: `ssh workboi 'hostname; sw_vers -productVersion; uname -m'`.
3. If it still times out, confirm both machines are on the same subnet/VLAN — the ~700 ms RTT hints at
   Wi-Fi with aggressive power save, or an AP-isolation policy between clients.

Screen sharing is already in use on this account (a `WorkBoi1 (chris-kct).vncloc` file was in the
single recorded meeting's notes), so enabling Remote Login is the only missing piece.

### Telling whether Remote Login is on, without SSH

Verified 2026-11-07. A sleeping Mac is easy to mistake for a broken key, and this cost real time:

- **Ping is not proof it is awake.** A sleeping Mac behind a Bonjour Sleep Proxy still answers ARP, and
  often ping, while everything else is dropped rather than refused. 66 % packet loss at ~900 ms RTT is
  the signature of a machine asleep or in deep Wi-Fi power save.
- **Dropped is not the same as closed.** `nc -z` timing out *and* no SSH banner
  (`exec 3<>/dev/tcp/HOST/22; head -c 40 <&3`) means packets are being dropped. A refused connection
  means something is listening and rejecting — a different problem with a different fix.
- **Ask Bonjour instead of guessing.** With Remote Login enabled the host advertises `_ssh._tcp`:

  ```bash
  dns-sd -B _ssh._tcp local     # expect an entry named after the host, e.g. WorkBoi
  dns-sd -B _rfb._tcp local     # screen sharing, same idea
  ```

  On 2026-11-07 this listed `FatBoi` for `_ssh._tcp` and `LilBoi`/`FatBoi` for `_rfb._tcp`, with **no
  WorkBoi entry at all** — Remote Login was off there. No key or `authorized_keys` change can fix that.
- **Wake-on-LAN will not rescue a randomized Wi-Fi MAC.** The target resolves to
  `12:52:ca:21:b7:22`; the first octet `0x12` has the locally-administered bit set, i.e. a privacy MAC.
  Magic packets can only reach a MAC that exists while the machine is associated. For reliable wake use
  Ethernet, or disable MAC randomization for that network, and enable "Wake for network access".

`~/.local/bin/workboi-bringup-wait.sh <seconds>` polls for port 22 and, the moment SSH answers, runs
the inventory below and the §5 checks into `~/.screenpipe-recorder/workboi-bringup-<date>.log`.

## 2. What to install

Match the reference host exactly before adding anything new:

1. Install the packaged app (`ScreenpipeRecorder.app`) and confirm the CLI works:
   `ScreenpipeRecorder record --help` — its flags are authoritative, it is not built from this repo.
2. **Grant TCC permissions to the app binary**, all four required:
   - **Microphone** — without it, capture starts and silently produces nothing.
   - **Screen Recording** — vision frames and AX walking.
   - **Accessibility** — meeting detector's AX tree scan *and* UI event capture.
   - **System Audio** — uses the CoreAudio process tap, hence `--experimental-coreaudio-system-audio`.
3. Resolve device names on the new machine with `curl -H "Authorization: Bearer $KEY" localhost:3030/audio/list`
   and put those exact strings into `--audio-device`. Names differ per host; copying FatBoi's strings
   verbatim will start zero devices.

## 3. What to copy, and how to adapt it

Copy these, then change the paths/devices/thresholds per host:

| From FatBoi | Change on WorkBoi |
| --- | --- |
| `~/Library/LaunchAgents/ai.screenpipe.recorder.plist` | `--audio-device` list; regenerate `SCREENPIPE_API_KEY` (do not reuse, do not commit) |
| `ai.screenpipe.budget.plist` | schedule only if it would collide with the recorder's own 04:15 work |
| `ai.screenpipe.visionretention.plist` | time; same `--days 14` |
| `ai.screenpipe.storagealert.plist` | thresholds sized to WorkBoi's disk (20 GB / 10 GB are FatBoi's) |
| `~/.local/bin/screenpipe-{vision-retention,storage-budget,storage-alert}.sh` | nothing structural — all paths come from arguments/env |

Do **not** copy `~/.screenpipe-recorder/db.sqlite` or the `store.bin` settings blob. Start with an empty
data directory so vision/audio settings are bound to the new host's displays and devices.

Install, then reload (plist argument changes are not picked up by `kickstart`):

```bash
for j in recorder budget visionretention storagealert; do
  launchctl bootout   gui/$(id -u)/ai.screenpipe.$j 2>/dev/null
  launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.screenpipe.$j.plist
done
launchctl list | grep screenpipe
```

## 4. Settings that must match, and one to reconsider

Carry these over because they encode hard-won lessons:

- `--retention-mode media` — **never `all`**; `all` also deletes `audio_transcriptions`.
- `--use-system-default-audio false` + explicit `--audio-device` list.
- `--disable-vision` absent, **and** `settings.disableVision=false` in `store.bin` — either one alone
  leaves vision off (`health` reports `disabled_by_setting`).
- One room mic + `System Audio (output)`. Do not add a second mic pointing at the same room: on FatBoi
  that produced duplicate transcripts of the same speech (33,833 vs 244 on a day's data).
- **Arriving hardware**: the EMEET PIXY and fifine microphones move from FatBoi to a Ugreen dock
  attached to WorkBoi, leaving BRIO as FatBoi's only mic. Configure them on WorkBoi as *its* room-mic
  candidates, not alongside BRIO — BRIO and a dock mic in the same room is the duplication case above.
- Verify audio by *transcription recency*, not by `audio_status: ok`.

Reconsider on WorkBoi:

- **`--experimental-meeting-piggyback`** was set to `false` on FatBoi, and meeting detection has not
  fired since (the only `meetings` row is from 2026-10-30, `detection_source=audio_process`). Start
  WorkBoi with the default (enabled) and keep the explicit device list, so detection has a working
  source. Document what actually happens during a real call.
- **`SCREENPIPE_API_KEY`** — generate a distinct key per host so one machine's credential can be
  revoked without touching the other, and keep it only in the plist (mode `600`).

## 5. Acceptance checks

Run these on WorkBoi before calling it done:

```bash
KEY=$(/usr/libexec/PlistBuddy -c "Print :EnvironmentVariables:SCREENPIPE_API_KEY" \
  ~/Library/LaunchAgents/ai.screenpipe.recorder.plist)

# 1. healthy, with the expected dependency tree
curl -s -H "Authorization: Bearer $KEY" localhost:3030/health | python3 -m json.tool | head -40

# 2. every configured device is present and active
curl -s -H "Authorization: Bearer $KEY" localhost:3030/audio/list

# 3. audio is actually becoming text (speak, then re-run)
curl -s -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -X POST localhost:3030/raw_sql \
  -d '{"query":"SELECT substr(timestamp,1,19) ts, device, substr(transcription,1,50) t FROM audio_transcriptions ORDER BY id DESC LIMIT 5","limit":5}'

# 4. vision is producing frames, not just "enabled"
#    expect frame_status=ok and a growing frames/elements count in /health

# 5. jobs are scheduled and have exit code 0
launchctl list | grep screenpipe
```

Then verify the meeting path with a real call, since it is the one thing that cannot be tested
synthetically: start a call, then watch for `meeting_detected: true` in `/health`, a non-null
`detectionSource` from `/meetings/status`, and a new row in `meetings`. If it does not fire, check that
the meeting app's main process (not just its widget/agent helpers) is running — the detector scans
running apps' accessibility trees and only counts leave/hangup controls.

## 6. Scheduling note

If WorkBoi becomes the control surface that reads both hosts, point it at FatBoi's API over the LAN
(`--listen-on-lan` is already enabled there) rather than syncing databases between machines. The
`unix-excl` lock means a second process cannot attach to FatBoi's DB directly, and copying a live
SQLite file without its `-wal` risks a torn read.
