# GenesisFanControl

A SwiftUI macOS clone of [Macs Fan Control](https://crystalidea.com/macs-fan-control):
the original full-window app, plus everything both apps in this repo share —
`GenesisFanControlCore` (the IOKit/AppleSMC backend, helper client, settings,
`AppState`), the privileged helper daemon, the global `fans` CLI and the test
suite. FanDeck (`../FanDeck/`) is UI-only on top of this core; the repo
overview is [../README.md](../README.md).

Tested on MacBookPro21,5 (M4 Pro, macOS 15). The read path also works on
Intel Macs; the write path's Apple-Silicon `Ftst` dance is a no-op there.

**Only one app manages fans at a time.** Running GenesisFanControl and
FanDeck together is unsupported (both drive the same helper and fans —
[decision log](../docs/specs/2026-07-18-fan-compact-decisions.md), Assumptions).

| Target | Kind | Purpose |
|---|---|---|
| `GenesisFanControlCore` | library | SMC bridge, AppState, settings, logging, helper protocol — shared by everything below |
| `GenesisFanControl` | executable | SwiftUI app, menu-bar icon, settings window |
| `GenesisFanControlHelper` | executable | privileged daemon launched by launchd; owns SMC writes |
| `fans` | executable | command-line client over the same `AppleSMCService` |

---

## Quick start

```bash
cd GenesisFanControl
bun install         # nothing actually pulled — only scripts/ has TS
bun run start       # build release + install /Applications/.app + launch
```

The first run pops one macOS admin prompt to drop a symlink at
`/usr/local/bin/fans`. Subsequent `bun run start` invocations need no
password — the install layout uses symlinks so every rebuild is
instantly reflected.

To control fans, click **Install Helper** in the amber banner that
appears when you drag a fan; a second admin prompt installs a launchd
daemon at `/usr/local/sbin/genesis-fan-control-helper`. From then on,
the GUI's drag-to-set drives real fans without `sudo`.

```bash
fans list                       # globally available
fans get F0
sudo fans set F0 const 3500     # helper first; root direct write is the fallback
sudo fans set F0 auto
fans watch                      # 1Hz tail of RPMs + headline sensor
```

---

## The SMC backend

`Sources/GenesisFanControlCore/SMC/AppleSMCService.swift` is the real IOKit
bridge. It speaks the standard private-AppleSMC ioctl that every Swift /
Objective-C SMC wrapper uses (smcFanControl, SMCKit, stats, AlDente, …).

### The wire protocol

Open `IOServiceMatching("AppleSMC")` with `IOServiceOpen`, then
`IOConnectCallStructMethod(connection, selector=2, input80B, 80, output80B, &80)`,
where the 80-byte buffer is **the** `SMCParamStruct`:

```text
offset  size  field
   0     4    key (FourCC, e.g. 'F0Ac')
   4     6    vers (SMCVersion)
  10     2    (padding)
  12    16    pLimitData (SMCPLimitData)
  28    12    keyInfo (SMCKeyInfoData — dataSize, dataType, dataAttributes + 3B pad)
  40     1    result (0 = ok; SMC firmware error otherwise)
  41     1    status
  42     1    data8 (call type: 5=read, 6=write, 9=getKeyInfo)
  43     1    (padding)
  44     4    data32
  48    32    bytes (payload)
                                                    total = 80
```

Swift packs `SMCKeyInfoData` to 9 bytes; three explicit `UInt8` pad fields
round it to 12. A `precondition(MemoryLayout<SMCParamStruct>.stride == 80)`
crashes startup if anyone ever re-packs it; without it, the kernel returns
garbage values silently.

### Data type tags

`keyInfo.dataType` is itself a FourCC:

| Type tag | Bytes | Semantics |
|---|---|---|
| `"ui8 "` / `"si8 "` | 1 | unsigned / signed 8-bit |
| `"ui16"` / `"si16"` | 2 | unsigned / signed 16-bit big-endian |
| `"ui32"` | 4 | unsigned 32-bit big-endian |
| `"fpe2"` | 2 | 14-bit unsigned int, 2-bit fraction (raw / 4) — most common fan target on Intel |
| `"sp78"` | 2 | signed 8.8 fixed-point (raw / 256) — common temperature |
| `"flt "` | 4 | IEEE 754 32-bit float — **the** Apple Silicon fan/temp encoding |

Encoders only need to be exact for the keys we WRITE — fan mode (`ui8`)
and fan target (`fpe2` or `flt`, machine-dependent).

### Apple Silicon write path — the `Ftst` unlock dance

On Intel, forcing manual mode is one write: `F{i}Md = 1`. On M1 / M2 / M3 /
M4 the SMC accepts that write (`result == 0`) but `thermalmonitord`
silently undoes it within milliseconds. The fix, cribbed from
[exelban/stats SMC/smc.swift:566-602](https://github.com/exelban/stats/blob/master/SMC/smc.swift):

1. **Fast path** — write the mode key = 1, wait 20 ms, verify by readback.
   An already-unlocked fan succeeds here.
2. **Read `Ftst`**. If it doesn't exist, give up. If it is already 1,
   retry the mode write up to 20× (each 20 ms readback + 50 ms) and stop there.
3. Otherwise **write `Ftst = 1`**, retrying up to 100× at 50 ms until the
   SMC accepts it (the firmware "unlock fan control" permission).
4. **Sleep 3 s** so thermalmonitord sees the unlock and yields.
5. **Retry the mode write up to 300×** (each 20 ms readback + 100 ms); the
   first few usually fail.

The 3 s sleep dominates a typical first write; the retry budgets alone
allow ~44 s (5 s + 3 s + 36 s) plus SMC I/O before giving up. Later writes
take the fast path. The sequence runs off the main thread (*Concurrency model*).

**Release direction matters.** To return to auto, write the mode key
FIRST while `Ftst` is still unlocked, THEN drop `Ftst`:

```swift
writeUInt8("F0Md", 0)   // mode back to auto WHILE Ftst is still 1
writeUInt8("Ftst", 0)   // re-lock; firmware reclaims fan control
```

Doing it the other way around (`Ftst=0` first, `F0Md=0` second) silently
fails: the firmware re-locks the moment `Ftst` goes 1→0 and ignores the
subsequent `F0Md` write ("click Auto → UI blinks to auto → snaps back").

**`Ftst` is global, not per-fan.** Releasing one fan drops `Ftst` to 0 only
when it is the last non-auto fan; otherwise the other fans' constants
would lose their unlock (`autoReleaseDirect`).

**Re-assertion is the mechanism, not maintenance.** thermalmonitord claws
back `F0Tg` (and the physical `F0Ac`) under thermal governance — 5500 RPM
can drop to ~4100 within seconds, even while `F0Md=1`. Since helper
protocol v8 the root helper owns the hold, re-pushing every held target on
its own 1 Hz timer whether or not the GUI runs. `.sensorBased` stays
host-driven: the GUI's `primeSnapshot()` computes the ramp target each tick
and sends it to the helper; a vanished sensor drops that fan to AUTO. Any
new mode or helper RPC that sets a target without joining this drifts back.

**Cold-start safe-reset.** On `AppleSMCService.init`, every fan found in
`F0Md=1` is dropped back to auto before the first snapshot, so a crashed
run's firmware-clamped `F0Tg` is never adopted as "user intent". Only the
GUI does this (`forceSafeReset: true`, the default); the helper and the
`fans` CLI pass `false`, so a launchd respawn or a `fans list` never
silently reverts a pinned CONSTANT.

**Absolute RPM safety floor.** `writeRPM(fanIdx:rpm:)` clamps to
`max(800, rawRPM)` BEFORE encoding so no path (CLI, helper, host
re-assertion, future programmatic) can write 0 — which on most Mac
fans stalls the bearing.

### Key quirks

- **`F0Tg` is `flt`, not `fpe2`, on Apple Silicon.** Writing `fpe2` bytes
  into the 4-byte float slot yields a denormal ≈ 1e-43 RPM, which the
  firmware fail-safes to maximum. `writeRPM` reads the real type tag with
  `getKeyInfo` first and encodes per tag; every write uses that dispatch.
- **`F0Md` or `F0md`.** `modeKey(forFan:)` probes both and caches the one
  that answers; without it, lowercase-only machines silently fail every write.
- **Sensors.** `FNum` (`ui8`) is the fan count; temperatures come from the
  hand-curated `AppleSMCService.candidateSensors` (answering keys only).
  ~20 M3 / M4 keys (`Te??`, `Tf??`) are not probed yet.

---

## Privileged helper daemon

Fan writes need root; the GUI doesn't run as root, so a small launchd
daemon does. Every GUI and CLI write goes **helper-first**; the direct SMC
path runs only inside the helper or as a last resort, where it succeeds
only if that process is privileged.

### Installation flow

`HelperInstaller.install()` runs from the GUI's **Install Helper** /
**Update Helper** button as **one** `osascript` "with administrator
privileges" call that:

1. Wipes any legacy MacsFanControl-era helper (bootout + `rm -f`).
2. Copies the freshly built helper to
   `/usr/local/sbin/genesis-fan-control-helper` (root:wheel, 0755).
3. Writes `/Library/LaunchDaemons/dev.foltyn.genesis-fan-control.helper.plist`
   with `RunAtLoad=true`, `KeepAlive=true`, log paths under `/var/log/`.
4. `launchctl bootout system <plist>` (in case of upgrade), then
   `launchctl bootstrap system <plist>`, then
   `launchctl kickstart -k system/<label>` to force-start it.
5. Waits up to 3 s for the socket; `ping()` succeeds or it throws `.timeout`.

### IPC — JSON over a Unix domain socket

`HelperProtocol.swift` defines `HelperRequest` (`ping`,
`setAuto(fanID:)`, `setConstant(fanID:rpm:)`) and `HelperResponse`
(`ok`, `error`, `backendName`, `protocolVersion`). Wire format: one JSON
object per line, `\n`-delimited, one request → one response → close, so
the helper stays stateless per connection. The client sets 5 s
`SO_RCVTIMEO`/`SO_SNDTIMEO` so a hung helper cannot pin the GUI.

### Socket authorization and the helper's safety net

- Socket `/var/run/genesis-fan-control.sock` is `root:admin 0660`, and
  every accepted connection is checked with `getpeereid(2)`: only root or
  the active console user is served (`HelperPeerAuth`).
- Every non-auto fan write is tracked. SIGTERM / SIGINT revert every held
  fan to auto before unlink + exit.
- Idle watchdog: no client activity for >60 s while holding fans → revert
  them (checked every 10 s). It covers a crashed or SIGKILLed GUI.
- Both apps add: quit (`applicationWillTerminate`) reverts every non-auto
  fan; a 30-minute max-hold watchdog reverts with a notification; on wake,
  non-auto fans are re-asserted.
- **Residual risk:** no code-signing identity check, so any process running
  as the console user can drive fans.

These describe the source; they are not runtime- or security-verified claims.

### Protocol version + auto-update banner

`HelperConstants.protocolVersion` is bumped on **every** helper-side
behavior fix (even bug-fix-only). Ping responses carry the helper's
compiled version; `HelperClient.health()` returns `.healthy`, `.outdated`
or `.down`; `AppState.tick` checks it at most every 5 s, skipping ticks
while a write is in flight. On `.outdated` a
persistent banner ("Helper is out of date — installed v1, GUI expects v2")
offers **Update Helper**, which re-runs the admin install. Without it a GUI
carrying fix X silently drives a helper built before X.

### Why not `NSXPCConnection`?

It would give a codesigning-anchored identity check for free, but needs the
helper inside a signed `.app` bundle (same team ID) registered via
`SMAppService.daemon(plistName:)` — incompatible with standalone
`swift build -c release` binaries.

---

## Fan modes (`FanMode`)

1. **Automatic** — `.auto`: mode key 0 (still under unlock), then `Ftst`
   0 per the release rules above.
2. **Constant speed** — `.constant(rpm: Int)`: the full unlock dance, then
   `writeRPM`; the helper holds it.
3. **Sensor-based** — `.sensorBased(sensorId: String, points: [RampPoint])`,
   `RampPoint = { tempC, rpm }`: every tick reads the sensor and
   interpolates the sorted points piecewise-linearly, holding `first.rpm`
   below `first.tempC` and `last.rpm` above `last.tempC`; the result is
   clamped into `[fan.minRPM, fan.maxRPM]`.

Ramp configs persist per fan (`SettingsStore.sensorRampConfigs`, JSON in
UserDefaults). The sheet hydrates from this fan's config, else any
sibling's; Apply saves it REGARDLESS of the committed mode, so switching
to constant or auto and back restores the exact ramp.

Both ramp editors (`FanControlSheet`, FanDeck's `RampEditorSheet`) clamp a
dragged point per axis (0–120 °C; RPM into `[fan.minRPM, fan.maxRPM]`),
never let it cross a neighbouring point's temperature, and allow deleting
a point only while more than two remain.

---

## Concurrency model

`AppState` is `@MainActor`; SMC work can take seconds, so `tick()` (1 Hz)
and `setMode(...)` run it on one serial `smcQueue` and hop back to publish.
`setMode` updates the UI optimistically and records `pendingIntent[fanID]`,
which every snapshot merges until the write completes (no flicker back to
the old setpoint); a rejected write sets `needsElevation`.
`AppState.publish(snapshot:)` is the single funnel for fans/sensors.
Concrete `SMCService` classes are `@unchecked Sendable`: the queue guards
their cache.

---

## Build system

`bun run start` (`scripts/install.ts`) builds release and installs
`/Applications/GenesisFanControl.app/` whose `Contents/MacOS/*` are
**symlinks** into `.build/release/`, plus `/usr/local/bin/fans`, so every
`swift build` is instantly reflected. A real `.app` (not `.build/debug/`) is
what Spotlight, the Dock, ⌘-Tab activation and the AppIcon need.

### ⚠ install:app DOES NOT update the installed helper at /usr/local/sbin/

The root-owned helper is copied only by the admin-prompted
`HelperInstaller`; `install:app` handles only the **unprivileged** GUI + CLI.

**Bump-on-every-helper-change rule:** when you edit any code that the
helper picks up (anything in `AppleSMCService`, `HelperProtocol`, the
helper's `main.swift`), increment `protocolVersion`. Otherwise the
GUI won't know the installed copy is stale.

### Scripts

| Script | Effect |
|---|---|
| `bun run build` | `swift build -c release` |
| `bun run build:debug` | `swift build` |
| `bun run start` | Build release + install .app + restart |
| `bun run start:debug` | Build debug + install .app + restart |
| `bun run dev` | Watch mode; rebuilds + restarts on save |
| `bun run icon` | Regenerate `scripts/AppIcon.icns` |
| `bun run test` | `swift test` (329 cases, ~6 s) |
| `bun run clean` | `swift package clean` |
| `bun run uninstall:app` | Remove .app + CLI symlink + helper daemon |

---

## CLI

`fans` runs the same `AppleSMCService` as the GUI: reads are unprivileged,
writes go helper-first with a direct write as the fallback (which needs
`sudo`). `/usr/local/bin/fans` is a symlink to `.build/release/fans`, so
`bun run start` updates it transparently.

```
fans <command>

  list                                          fans + sensors
  get <fanID>                                   detail
  sensors                                       all sensors
  sensor <sensorID>                             detail
  set <fanID> auto                              release
  set <fanID> const <rpm>                       constant
  set <fanID> sensor <sensorID> <lo> <hi>       sensor-based
  set <fanID> ramp <sensorID> <c1:rpm1> <c2:rpm2> ...   ramp (≥ 2 points)
  watch [intervalSec]                           tail readings
```

The built-in `fans help` text still claims a MOCK backend; that is stale —
the CLI falls back to `MockSMCService` only when AppleSMC cannot be opened.

---

## Logging

Every `Log.<category>` call writes to `os.Logger` AND the in-memory
`LogStore.shared` ring buffer (1000 entries); `.public` privacy is explicit
on every interpolation so unified-log output stays legible. To tail:

```bash
log stream --predicate 'subsystem == "dev.foltyn.genesis-fan-control"' --level debug
```

---

## Testing

`swift test` runs **329 XCTest cases** in ~6 s. Each executable's decision
logic lives in a `Core` seam the executable calls (`MaxHoldDecider`,
`HelperHoldState`, `CLIParser`, …), so tests guard the real path, not a
copy; ViewInspector (test-only) covers view structure.

- Singletons are reset or avoided per test: `LogStore.shared.clear()` in
  setUp/tearDown; `SettingsStore(defaults:)` over UUID-named UserDefaults
  suites; `AppState(smc:autoStartPolling:false)` for AppState tests.
- Activation policy (`ActivationPolicyDecider`, called by both app
  delegates): never demote while the main OR Settings window is still
  visible; `ActivationPolicyTests` assert it.
- The real `AppleSMCService` is **not** unit-tested — it talks to the kernel.
- **Not covered by `swift test`:** real on-screen window-surfacing (does
  the gear bring Settings to the front, does the menu-bar click raise the
  main window) — that needs XCUITest or a manual click.

---

## Known limitations

The 3-lens review (`.claude/plans/2026-06-30-GenesisFanControlReview.md`,
41 findings) is not in this repository. Its open items were re-checked
against source on 2026-10-22; "implemented" means the code exists, not
that it was runtime- or security-tested.

Implemented in source: the socket peer check (above), helper SIGTERM and
idle-watchdog revert, both apps' max-hold, quit and wake handling, `Ftst`
released only with the last non-auto fan, the helper-held constant (so
`fans set … const` survives the CLI exiting), `FanControlSheet` reading the
fan live by `fanID`, `openAtLogin` wired to `SMAppService.mainApp` and the
menu-bar style/fan/sensor settings re-rendering the menu bar, the dead
update-check and language controls removed (`GeneralSettingsTab`), 5 s socket
timeouts, `shellEscape`d installer paths, drag committing on `onEnded`, and
a vanished ramp sensor dropping its fan to AUTO.

Still open:

- No code-signing identity check on the socket (*Future work*).
- CLI-set sensor/ramp modes are not maintained once the CLI exits
  (*Troubleshooting*).
- Not re-checked: multi-click `setMode` race wipes
  `pendingIntent` (per-fan version tokens); per-tick `helperClient.setMode`
  costs a fresh socket + 3 SMC round-trips; `MenuBarIconTab` Picker rows
  use `HStack{Image,Text}` (fix with `Label(...).tag(...)`);
  activation-policy demote 300 ms `asyncAfter` cancellation; missing M3/M4
  sensor keys; silent mock fallback (a banner would be kinder); in-app log
  viewer.

---

## Future work

- **Helper via SMAppService + code-signature peer identity** — an `.app`
  bundle layout and a paid developer ID, then an `NSXPCConnection`
  machService validating `auditToken` against the signing identity.
- **Sparkle for OTA updates.**
- **Localization** — English-only today; move strings into a String Catalog.
- **Hardware-specific sensor catalog** (`sysctl machdep.cpu.brand_string`).
- **Per-fan profile presets** ("Silent / Balanced / Performance").
- **Headless CLI mode** — `fans daemon` applying a curve from a YAML config.
- **Sensor history graph** — a 10-minute sparkline per sensor.
- **CLI smoke test** — `bun run test:smoke`: `fans list`, RPM count > 0.

---

## Troubleshooting & dev gotchas

Each of these has been mis-diagnosed at least once.

**Gear button doesn't open Settings.** Hit-testing, not action dispatch:
ZStack hit-tests last-declared-first and an HStack with a trailing
`Spacer()` claims its whole width. Declare the interactive HStack AFTER
decorative overlays (`topStatusBar`), AND put `.allowsHitTesting(false)` on
the decorative side. Don't reach for SettingsLink / NSApp.sendAction /
NSViewRepresentable wrappers until the click is confirmed to reach the
Button. Open Settings with the `openSettings()` environment action: the
`showSettingsWindow:` selector was removed in macOS 14.

**App crashes with EXC_CRASH / SIGABRT during layout.** Look for
`NSHostingView.SizeConstraints.update(from:)` in the backtrace: an
`NSHostingView` with `translatesAutoresizingMaskIntoConstraints = false` +
explicit edge constraints. Drop the constraints and let the hosting view
size itself, OR avoid the bridge (a SwiftUI Button already handles
`mouseDownCanMoveWindow = false`).

**Picker / menu rows show only the label, not the live value.** NSMenuItem
takes only the first Text of a Button label: embed the value in the title
string (`Menu { Button { Label(..., systemImage:) } }`).

**Fan setpoint reverts to the firmware-clamped value within ~5 s.** The
target is not in the per-tick re-assertion (*The SMC backend*).

**Click "Auto" → UI blinks to auto → snaps back to constant.**
`setMode(.auto)` is writing `Ftst=0` BEFORE `F0Md=0`. Reverse the order.

**Helper write failed, banner didn't pop.** The installed helper at
`/usr/local/sbin/` is older than the protocol version. Bump
`HelperConstants.protocolVersion` to force the GUI's `health()` check
to flag it `.outdated` → banner pops with "Update Helper".

**`fans set F0 sensor ...` (or `ramp`) from the CLI stops following
temperature.** The curve lives only in that CLI process, which exits; the
helper receives just a constant seed (the fan's current target, else its
minimum), and no
other process (a running GUI, a later `fans watch`) learns the curve. The
helper holds that seed as a fixed RPM and reverts the fan to auto after
60 s without client activity. Configure a temperature ramp in the GUI that
stays running; persistent CLI curve control is not implemented.

**Tests `testTickAdvancesLastUpdated` / `testTickRefreshesSensors`
fail.** They assert right after `state.tick()`, but `tick()` dispatches onto
`smcQueue` and publishes back on `@MainActor`. Use `drainTicks()` (in
`AppStateTests.swift`) between `tick()` and the assertion.

**App relaunches but the new binary doesn't seem to load.** Confirm via
`ls -la /Applications/GenesisFanControl.app/Contents/MacOS/GenesisFanControl`
that it's a symlink into `.build/release/`. If it points somewhere old,
`bun run uninstall:app` + `bun run install:app` to rebuild from scratch.

---

## History

The review-era changelog (monorepo SHAs that do not resolve here) and the
"shipped since the review" list are in `git show 24a668d:GenesisFanControl/README.md`.
Helper changes v3–v8 (peer check, SIGTERM revert and idle watchdog,
canonical AUTO release, helper-owned hold) are recorded beside
`HelperConstants.protocolVersion` in `HelperProtocol.swift`.

---

## Sources & research

- [exelban/stats](https://github.com/exelban/stats) — canonical Swift SMC
  bridge with Apple Silicon support; source of the `unlockFanControl` sequence.
- [beltex/SMCKit](https://github.com/beltex/SMCKit) — the original 2014
  Swift SMC wrapper and struct layout.
- [hholtmann/smcFanControl](https://github.com/hholtmann/smcFanControl)
  — Objective-C predecessor; canonical install/uninstall pattern.
- [AlDente](https://github.com/davidwernhart/AlDente),
  [BatFi](https://github.com/rurza/BatFi),
  [bclm](https://github.com/zackelia/bclm) — cross-checks on the type-tag set.

The full research write-up (line numbers, commit SHAs, the unprobed M3/M4
keys) is in the author's private notes
(`~/Tresors/Projects/GenesisBrain/GenesisTools/Fans/Research-2026-06-29.md`),
not in this repository.
