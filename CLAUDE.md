# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Microduck is the firmware brain of a small biped robot (RK3566 board, aarch64 Linux): a 50 Hz control loop driving 15 servos from RL policies (ONNX, trained in the separate `microduck_rl` repo), plus Bluetooth, camera, and a signed/health-gated/rollback-capable update system. One Rust workspace, no framework, one crate per service.

## Commands

```bash
cargo test --workspace        # full test suite — no hardware, no network, no Docker
cargo test -p <crate>         # one crate
cargo fmt --all
```

- Needs Rust **1.89+**. On Linux, first: `sudo apt-get install -y libudev-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev`.
- On macOS the workspace builds fully except two ToF driver tests (the vendor C driver is `cfg(target_os = "linux")` only).
- `configd` (NetworkManager client) and `btd` (BlueZ client) are **Linux-only code that compiles on neither macOS nor a plain Linux host build the same way** — lint them against the board's target or the breakage ships:

```bash
RUSTFLAGS="-D warnings" cargo clippy -p configd --all-targets --target aarch64-unknown-linux-gnu
```

- Cross-compile for the board: `cargo board --bins` (alias → `cargo zigbuild --release --target aarch64-unknown-linux-gnu.2.31`).
- Run a change on a real robot without publishing: `scripts/dev-push.sh <user@board>` — builds here, installs there as an ordinary gated update.
- `scripts/board-test.sh` runs the cross-compiled integration assertions in CI (rollback, tamper refusal, boot-counter recovery, socket auth).

## Architecture

`docs/design/architecture.md` is the authoritative document; read it before changing the service split or IPC. The invariants that constrain any change:

- **Seven daemons over unix sockets** (`/run/<service>.sock`), speaking **JSON-RPC 2.0, one object per line (NDJSON)**. The wire contract lives in `duck-ipc-proto`, which may depend only on `serde`, `serde_json`, and `semver` — never add http/tar/crypto/async-runtime deps there. Protocols version-bump rules are documented on `PROTOCOL_VERSION` in `duck-ipc-proto/src/lib.rs`.
- **`robotd` is the only process that touches motors.** Clients send *intents* ("go this fast", "stand up"); the safety layer inside `robotd` decides what is executable. No client can bypass fall detection or joint/thermal limits.
- **`configd`, `updaterd`, and `btd` must survive a dead `robotd`** — they are the recovery path, so no systemd dependency on it, no ML runtime, no media stack in those crates.
- **`robotd`'s control loop never blocks on another service** — cross-service reads are last-value-wins caches, never synchronous RPC.
- **Single writer per piece of state**; everyone else reads or subscribes. Config lives in `configd` (not `robotd`) precisely so it is reachable when the control daemon is broken.
- Video/audio frames never cross the control-plane socket — perception publishes *features* next to the sensor (`mediad`), not frames.

Crate roles (full layout in CONTRIBUTING.md):

- **Daemons**: `robotd` (control loop, voice), `updater` (engine + `updaterd`), `configd` (wifi/identity/pairing), `btd` (BLE front door, owns nothing), `padd` (gamepad → intents, unprivileged client), `mediad` (camera/WebRTC/remote gateway), `tof` (`tofd`, publishes 8×8 depth, reads nothing).
- **Libraries** (no sockets, nothing starts them): `duck-ipc-proto` (wire contract), `duck-control` (model/bus/IMU/observations/policy/safety — the control core), `kinematics`, `odometry`, `sounds`, `pet-detect`, `robotd-params`.
- **Tools**: `robotctl` (on-robot CLI), `duckctl` (laptop-side, never shipped — excluded from `default-members` for that reason), `xtask` (package/sign/promote), `test-support`.

**Update system**: releases are swapped, not patched — a build lands whole under `/opt/robot/daemon/releases/<version>/`, `updaterd` verifies the signature, moves the `current` symlink, restarts units, then health-gates via `robotd`; failure auto-rolls-back. `updater/tests/apply.rs` drives the real engine with injected faults (bad signature, unhealthy release, power loss mid-swap) and is the honest spec of what the update system guarantees — treat it as the source of truth, not a mock suite.

**One-source-of-truth pins**: `[workspace.metadata.*]` in the root `Cargo.toml` pins ONNX Runtime, RKNN NPU runtime, policy set, and GStreamer plugin versions. Standalone scripts fetched with curl (`scripts/setup-*.sh`, `seed-policies.sh`) carry literal copies of these values and cannot read `Cargo.toml`; tests assert the two agree. **When bumping any pinned version, update both places or the test fails** (this is how a board once got an ONNX Runtime its `ort` crate panicked on).

## Conventions

- **Comments say why, not what** — the reasoning outlives the code.
- **Every non-obvious decision gets a test**, and the test's comment says which failure it exists to prevent. Rollback paths especially: they only run when something else already went wrong.
- Reach for an existing crate before writing it yourself; dependency count is not the optimisation target.
- Commit trailers use `Assisted-by:` (not `Co-Authored-By:`) for AI assistance.
- Releases are signed **in CI only**, never locally. Tags: `daemon-staging-vX.Y.Z` (builds, signs, publishes to staging) and `daemon-vX.Y.Z` (promotes staging or builds directly). Bump the workspace version in `Cargo.toml` first — `xtask package` refuses a tag that disagrees.

## Where things are written down

- `docs/design/` — architecture and per-system design docs (the "why")
- `docs/project/` — roadmap, CI setup, records of what has gone wrong
- `docs/robot/` — operating a robot, dev workflow (`cheatsheet-dev.md`, `dev-push.md`, `install-dev.md`)
- `CONTRIBUTING.md` — build/test details, crate layout, releasing
