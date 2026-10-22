# FanDeck

macOS fan control for Apple Silicon. A menu-bar app that reads the SMC directly
over IOKit, and — via a small root helper daemon — actually *writes* fan mode and
target RPM, which on M-series Macs is harder than it sounds (see
[the SMC write path](GenesisFanControl/README.md#apple-silicon-write-path--the-ftst-unlock-dance)).

The repo holds **two apps sharing one core**:

| | |
|---|---|
| **FanDeck** | the current app — menu-bar-first (`LSUIElement`), a rich `MenuBarExtra` panel plus a compact drag-to-set gauge window |
| **GenesisFanControl** | the original full-window app, kept building. It also owns `GenesisFanControlCore` (the SMC bridge, helper client, settings, app state), the privileged helper, the `fans` CLI, and the whole 329-case test suite |

FanDeck is UI-only: every fan write in both apps goes through the same
`GenesisFanControlCore`. It is a standalone utility — it shares no code with the
FixIt product, and is published here because it is generally useful, not because
anything else depends on it.

**Status: works, feature-complete for v1, dormant.** Last product change 2026-07-24;
`release.yaml` declares version 1.0.0. Developed on MacBookPro21,5 (M4 Pro).
The read path also works on Intel; the Apple-Silicon-specific `Ftst` unlock is a
no-op there. Not notarized — builds are ad-hoc signed (`CODE_SIGN_IDENTITY: -`),
so a downloaded build needs a Gatekeeper override.

## Quickstart

Prerequisites: macOS 14+ and a Swift 6 toolchain — Xcode Command Line Tools are
enough for the core/CLI/tests below; Xcode.app itself (plus
[Tuist](https://tuist.dev), `brew install tuist`) is needed only to build
**FanDeck.app**. [Bun](https://bun.sh) runs the GenesisFanControl scripts. No env
vars, no `.env`, no secrets — nothing to configure.

Build and run the **core, CLI and tests** without Xcode at all:

```bash
cd GenesisFanControl
swift build          # → .build/debug/{GenesisFanControl,fans,genesis-fan-control-helper}
swift test           # 329 XCTest cases, ~6 s — verified green on a clean clone
```

Build **FanDeck.app** (Tuist generates the Xcode workspace; it is not committed):

```bash
tuist install                  # resolve ViewInspector
tuist generate --no-open       # → GenesisFans.xcworkspace
./scripts/dist.sh              # Release build + embedded helper + ad-hoc re-sign
                               # → dist/FanDeck.zip
```

Unzip `dist/FanDeck.zip` into `/Applications` and launch. To control fans (rather
than just watch them) click **Install Helper** in the app: one admin prompt installs
a launchd daemon at `/usr/local/sbin/genesis-fan-control-helper`. After that, writes
need no `sudo`.

For the legacy app + the global `fans` CLI, the Bun scripts do the install dance
(symlinked `.app` in `/Applications`, `fans` in `/usr/local/bin`):

```bash
cd GenesisFanControl
bun run start        # release build → install → relaunch
bun run dev          # watch Sources/, rebuild + relaunch on save
fans list            # once installed: fans + sensors
sudo fans set F0 const 3500
```

## Repository map

| Path | What |
|---|---|
| `FanDeck/` | the menu-bar app — Tuist project, SwiftUI only (`MenuPanel`, `MainWindow` gauges, `RampEditor`, theme) |
| `GenesisFanControl/` | SPM package *and* Tuist project: `GenesisFanControlCore` (SMC/IOKit, helper client, `AppState`, settings), the original app, the root helper, the `fans` CLI, `Tests/` |
| `GenesisFanControl/scripts/` | Bun-driven install / uninstall / watch-mode / icon generation |
| `Workspace.swift`, `Tuist/` | Tuist workspace tying both projects together; `Tuist/Package.swift` holds external deps (ViewInspector, test-only) |
| `scripts/dist.sh` | the release build — the only supported way to produce a shippable `FanDeck.app` |
| `docs/specs/` | decision log for the FanDeck design (D1–D7) |
| `release.yaml` | `pultik ship` manifest (app id, icon, artifact path) |

**The deep documentation is [`GenesisFanControl/README.md`](GenesisFanControl/README.md)** —
it covers the 80-byte `SMCParamStruct` wire protocol, the `Ftst` unlock
dance and why release order matters, per-tick re-assertion against
`thermalmonitord`, the helper's socket protocol and hardening, the concurrency
model, testing strategy, known limitations and troubleshooting. Read it before
touching anything under `SMC/` or `Privileged/`.

## Development

Everything below is verified to run from a clean clone:

| Command | Where | Effect |
|---|---|---|
| `swift build` | `GenesisFanControl/` | build core + 3 executables |
| `swift test` / `bun run test` | `GenesisFanControl/` | 329 XCTest cases |
| `bun run start` / `start:debug` | `GenesisFanControl/` | build + install the legacy `.app` + `fans` CLI, relaunch |
| `bun run dev` | `GenesisFanControl/` | fs.watch rebuild loop (200 ms debounce) |
| `bun run uninstall:app` | `GenesisFanControl/` | remove `.app`, CLI symlink, helper daemon |
| `tuist install` + `tuist generate` | repo root | produce `GenesisFans.xcworkspace` |
| `./scripts/dist.sh` | repo root | Release `FanDeck.app` → `dist/FanDeck.zip` |

There is no CI in this repo: `swift test` locally is the gate.

Two rules that are easy to get wrong and expensive to debug:

- **Bump `HelperConstants.protocolVersion` whenever you change code the helper
  compiles in** (`AppleSMCService`, `HelperProtocol`, the helper's `main.swift`).
  `bun run start` cannot overwrite the root-owned daemon, so without the bump the
  GUI silently talks to a stale helper. With it, the app shows an "Update Helper"
  banner.
- **Fan writes are real hardware.** `writeRPM` clamps to a hard 800 RPM floor and
  the helper reverts locked fans on SIGTERM and on a 60 s idle watchdog — keep those
  invariants intact when refactoring, and prefer `MockSMCService` for anything
  exercised in tests.

Releases go out through `pultik ship` from the repo root, which runs
`scripts/dist.sh` and publishes `dist/FanDeck.zip` per `release.yaml`. There is no
auto-update channel in the app (Sparkle is listed as future work).

## Repository etiquette

- Work on a branch named `work/<slug>` and open a PR against `master`.
- **PRs to `master` require the owner's (@LEFTEQ) review** — an org ruleset enforces
  it, and the owner merging is the review gate. Do not self-merge or enable
  auto-merge.
- Conventional Commits (`fix(genesis-fan-control): …`, `feat(fandeck): …`) —
  the whole history follows this, scope by app.
- `GenesisFans.xcworkspace/`, `Derived/`, `.build/` and `dist/` are generated.
  Never commit them.
