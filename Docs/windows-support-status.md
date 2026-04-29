# Windows Support Execution Status

This file tracks implementation progress for `Docs/windows-support-plan.md`.

## Status Legend

- `pending`: not started
- `in_progress`: actively being worked on
- `blocked`: waiting on a decision or external dependency
- `done`: implemented and verified
- `deferred`: intentionally postponed

## Current Focus

| Field | Value |
|---|---|
| Current milestone | Milestone 1 — Windows Foreground MVP |
| Current status | pending |
| Last updated | 2026-04-29 |
| Notes | Start with foreground bridge support before background service or desktop refresh parity. |

## Milestones

| Milestone | Goal | Status | Verification |
|---|---|---|---|
| 1 | Windows foreground bridge MVP | pending | `phodex-bridge` tests pass; Windows smoke test reaches QR pairing path |
| 2 | Windows state, git, and path compatibility | pending | State reset and git/path tests pass with Windows mocks/runner |
| 3 | Windows desktop integration | pending | Desktop open/resume behavior works or returns clear unsupported result |
| 4 | Windows background service | pending | `start/status/stop/restart` use Windows daemon backend through mocked tests |
| 5 | Docs, scripts, CI, and release readiness | pending | Windows docs and CI matrix are complete |

## Milestone 1 — Windows Foreground MVP

### Implementation

| ID | Task | Status | Files |
|---|---|---|---|
| M1-1 | Add shared platform helper | pending | `phodex-bridge/src/platform.js` |
| M1-2 | Add command runner abstraction | pending | `phodex-bridge/src/command-runner.js` |
| M1-3 | Add URL opener abstraction | pending | `phodex-bridge/src/open-url.js` |
| M1-4 | Add wake-lock abstraction with Windows no-op | pending | `phodex-bridge/src/wake-lock.js`, `phodex-bridge/src/bridge.js` |
| M1-5 | Generalize auth login opener | pending | `phodex-bridge/src/bridge.js` |
| M1-6 | Verify `remodex up/run` foreground routing on Windows | pending | `phodex-bridge/bin/remodex.js` |
| M1-7 | Add Windows foreground CLI tests | pending | `phodex-bridge/test/remodex-cli.test.js` |

### Verification

| Check | Status | Command / Method | Notes |
|---|---|---|---|
| Bridge unit tests | pending | `cd phodex-bridge && npm test` | Run after M1 implementation |
| Relay unit tests | pending | `cd relay && npm test` | Should remain unaffected |
| Windows foreground smoke | pending | `remodex up` on Windows | Requires Windows machine/VM |
| macOS regression | pending | `remodex up` uses launchd path | Manual or test-backed |

## Milestone 2 — Windows State, Git, and Paths

| ID | Task | Status | Files |
|---|---|---|---|
| M2-1 | Add credential-store abstraction | pending | `phodex-bridge/src/credential-store.js` |
| M2-2 | Preserve macOS Keychain backend | pending | `phodex-bridge/src/secure-device-state.js` |
| M2-3 | Add Windows credential backend | pending | `phodex-bridge/src/credential-store.js` |
| M2-4 | Add platform permission helper | pending | `phodex-bridge/src/daemon-state.js`, helper TBD |
| M2-5 | Add null-device helper and replace `/dev/null` | pending | `phodex-bridge/src/null-device.js`, `phodex-bridge/src/git-handler.js` |
| M2-6 | Add Windows quick project locations | pending | `phodex-bridge/src/project-handler.js` |
| M2-7 | Add Windows state/git/path tests | pending | `phodex-bridge/test/*.test.js` |

## Milestone 3 — Windows Desktop Integration

| ID | Task | Status | Files |
|---|---|---|---|
| M3-1 | Split desktop handler into adapters | pending | `phodex-bridge/src/desktop-handler.js`, `phodex-bridge/src/desktop/*` |
| M3-2 | Add Windows Codex Desktop discovery/opening | pending | `phodex-bridge/src/desktop/windows.js` |
| M3-3 | Make session resume platform-aware | pending | `phodex-bridge/src/session-state.js` |
| M3-4 | Split desktop refresher by platform | pending | `phodex-bridge/src/codex-desktop-refresher.js` |
| M3-5 | Add Windows desktop tests | pending | `phodex-bridge/test/*.test.js` |

## Milestone 4 — Windows Background Service

| ID | Task | Status | Files |
|---|---|---|---|
| M4-1 | Add daemon controller abstraction | pending | `phodex-bridge/src/daemon-controller.js` |
| M4-2 | Wrap macOS launch agent as daemon backend | pending | `phodex-bridge/src/macos-launch-agent.js` |
| M4-3 | Add Windows Task Scheduler backend | pending | `phodex-bridge/src/windows-service.js` |
| M4-4 | Route service CLI commands by platform | pending | `phodex-bridge/bin/remodex.js`, `phodex-bridge/src/index.js` |
| M4-5 | Add Windows service mock tests | pending | `phodex-bridge/test/*.test.js` |

## Milestone 5 — Docs, Scripts, CI, and Release Readiness

| ID | Task | Status | Files |
|---|---|---|---|
| M5-1 | Add Windows CI matrix | pending | `.github/workflows/bridge-check.yml` |
| M5-2 | Add Windows local run documentation | pending | `README.md`, `Docs/self-hosting.md` |
| M5-3 | Add PowerShell local launcher if needed | pending | `run-local-remodex.ps1` |
| M5-4 | Add Windows troubleshooting docs | pending | `README.md`, `Docs/self-hosting.md` |
| M5-5 | Run final Windows smoke test | pending | Manual |

## Decision Log

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-29 | Start with foreground Windows bridge support | Fastest useful slice and lowest risk; background service and desktop refresh can follow later. |
| 2026-04-29 | Prefer user-level Task Scheduler for Windows background mode | It matches macOS LaunchAgent more closely than a machine-level Windows Service. |

## Open Questions

| ID | Question | Status | Resolution |
|---|---|---|---|
| Q1 | Which Windows credential backend should be used? | open | Evaluate native dependency vs command-based Credential Manager integration during Milestone 2. |
| Q2 | Is Windows Codex Desktop discoverable by protocol, executable path, or both? | open | Validate during Milestone 3. |
| Q3 | Is a PowerShell launcher necessary, or are npm scripts enough? | open | Decide during Milestone 5 after foreground flow is implemented. |

## Completion Criteria

- Windows foreground bridge can pair, send prompts, and receive streaming Codex output.
- Windows pairing/device state is secure and resettable.
- Windows git/status/diff/path operations pass tests.
- Windows docs explain setup and troubleshooting.
- CI covers Linux, macOS, and Windows Node tests.
- macOS launchd, Keychain, and desktop refresh behavior do not regress.
