# FanDeck — agent notes

macOS fan control for Apple Silicon: two apps over one shared core that
reads the SMC over IOKit and writes fans through a root launchd helper
(`/usr/local/sbin/genesis-fan-control-helper`, installed by the app's
**Install Helper**: one admin prompt, then writes need no `sudo`). A
standalone utility — it shares no code with the FixIt product; no env
vars, no secrets. Overview, quick start and full repo map:
[README.md](README.md). SMC/helper reference:
[GenesisFanControl/README.md](GenesisFanControl/README.md).

## Compass

- FanDeck is UI-only: every fan write in both apps goes through
  `GenesisFanControlCore`.
- Only one app manages fans at a time; running FanDeck and
  GenesisFanControl together is unsupported
  ([decision log](docs/specs/2026-07-18-fan-compact-decisions.md)).
- Builds are ad-hoc signed (`CODE_SIGN_IDENTITY: -`), not notarized: a
  downloaded build needs a Gatekeeper override.
- `scripts/dist.sh` is the only supported way to produce a shippable
  `FanDeck.app`; releases go out through `pultik ship` from the repo root
  (`release.yaml`). The app has no auto-update channel.

## Map

`FanDeck/` menu-bar app (Tuist, SwiftUI only) · `GenesisFanControl/` SPM
package + Tuist project: `GenesisFanControlCore`, the original app, the
root helper, the `fans` CLI, `Tests/` · `Workspace.swift` + `Tuist/` the
workspace · `docs/specs/` decision log (history).

## Commands

```bash
cd GenesisFanControl && swift build && swift test   # core + CLI + helper, 329 cases
cd GenesisFanControl && bun run start               # legacy app + fans CLI, relaunch
tuist install && tuist generate --no-open           # repo root → GenesisFans.xcworkspace
./scripts/dist.sh                                   # repo root → dist/FanDeck.zip
```

Also in `GenesisFanControl/`: `bun run test`, `bun run start:debug`, `bun run dev`
(watch), `bun run uninstall:app`; once installed, `fans list`,
`sudo fans set F0 const 3500` — table in README §Development.

## Non-obvious

- Two manifests describe the same targets: `GenesisFanControl/Package.swift` (SPM)
  and `GenesisFanControl/Project.swift` (Tuist). They are twins — change one, change
  the other, or `swift build` and the Xcode workspace drift apart.
- The Xcode workspace is generated. `tuist install && tuist generate --no-open`
  before anything Xcode-shaped; never commit `GenesisFans.xcworkspace/`, `Derived/`,
  `.build/` or `dist/`.
- `bun run start` cannot replace the root-owned helper at `/usr/local/sbin/`. Bump
  `HelperConstants.protocolVersion` whenever you touch code the helper compiles in
  (`AppleSMCService`, `HelperProtocol`, `GenesisFanControlHelper/main.swift`),
  otherwise the GUI silently drives a stale daemon instead of showing the
  "Update Helper" banner.
- Bun only — the `package.json` scripts are Bun scripts. No npm/pnpm/yarn.
- The `fans help` text still claims a MOCK backend. It is stale; the CLI has used
  `AppleSMCService` since `fans/main.swift:33`. Don't propagate the claim.

## Fan writes are real hardware

`GenesisFanControl/Sources/GenesisFanControlCore/SMC/` and `.../Privileged/` drive a physical machine.
Read `GenesisFanControl/README.md` (SMC backend + helper sections) before editing
there — the `Ftst` unlock sequence, its release *order*, and the per-tick
re-assertion against `thermalmonitord` are load-bearing and counter-intuitive.
Preserve the 800 RPM floor in `writeRPM`, the cold-start safe reset, the
SIGTERM/idle-watchdog revert of locked fans, and the `getpeereid` peer check.
The same read-first and preserve rules cover the helper
(`GenesisFanControl/Sources/GenesisFanControlHelper/`, whose idle watchdog reverts
after 60 s), the `fans` CLI and both apps' delegates (quit revert, 30-minute
max-hold revert).

## Testing

`swift test` from `GenesisFanControl/` — 329 XCTest cases, ~6 s, and there is no CI, so
it is the only gate. Exercise logic through `MockSMCService`; never pin a real fan in a test.

## Etiquette

Branch `work/<slug>`, Conventional Commits scoped by app (`feat(fandeck):`,
`fix(genesis-fan-control):`), PR to `master`. The owner (@LEFTEQ) reviews and merges —
never self-merge, never enable auto-merge. An org ruleset enforces the owner's review.
