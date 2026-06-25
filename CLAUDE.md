# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A cloud-free client + reverse-engineering bench for the Oura ring BLE protocol. It reads data **directly from the ring** over BLE — no Oura account. Two surfaces share one protocol understanding:

- **`crates/`** — the production Rust client (`oura` binary).
- **`tools/`** — Python research bench for protocol exploration (`oura_protocol.py` is the live command matrix; the Rust decoders are ports of what's discovered here).

Designed for Ring 3/4/5, which share GATT layout, framing, and auth. The code branches on reported **capabilities**, not model numbers.

## Build / test / run

```bash
cargo build --release            # binary at target/release/oura
cargo test --workspace           # protocol/auth/parser/store tests
cargo test -p oura-protocol      # one crate
cargo test events::tests::spo2   # one test by path substring
cargo audit                      # dependency vuln scan (0 expected)
```

Rust toolchain lives at `~/.cargo/bin` (rustup) — `source ~/.cargo/env` if `cargo` isn't on PATH. SQLite is vendored (`rusqlite` `bundled`), so there's no system DB dependency.

Python bench (needs **Python 3.13** — older interpreters pull an ancient, vulnerable mitmproxy):

```bash
python3.13 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/python tools/oura_protocol.py --list
.venv/bin/python tools/test_firmware_download.py   # the one Python test
```

On macOS, grant the terminal Bluetooth permission before any BLE command.

## Architecture: fetch → interpret → apply

The workspace is split by concern; each crate depends only "downward". Full diagram in [docs/architecture.md](docs/architecture.md).

| Crate | Layer | Owns | I/O |
| --- | --- | --- | --- |
| `oura-protocol` | interpret (decode) | packet framing, request builders, app-auth AES, device parsers, **event-body decoders** (bytes → typed samples) | none (pure) |
| `oura-link` | fetch | `Transport` trait + `btleplug` BLE, `OuraClient` (connect/auth, sync drain, live HR/ACM, features, RData) | BLE, async |
| `oura-analysis` | interpret (compute) | ecore-ported metric algorithms (HRV, SpO2, baselines, sleep, scores) | none (pure) |
| `oura-store` | apply | SQLite: raw events (lossless), readings, daily metrics, per-device sync cursor | SQLite |
| `oura-cli` | wiring | command dispatch + the `viz`/`game` web UIs (`motion_server`) | everything |

**Where to add things:**
- New **event-body decoder** → `oura-protocol::events::decode_body` + a test with captured bytes. Never needs a re-sync — `oura redecode` backfills stored raw bodies.
- New **BLE command/capability** → request builder in `oura-protocol::protocol` + a method on `OuraClient` in `oura-link`.
- New **metric/algorithm** → a module in `oura-analysis` + a one-file doc under `docs/algorithms/` (record source ecore fn/addr, inputs, formula, impl location, validation status; update `docs/algorithms/README.md`).
- New **table/query** → `oura-store::storage`. New **command/UI** → `oura-cli`.

**Errors:** each layer has its own error type (no shared enum). Protocol decoders are infallible (`Option`). `oura-cli` collapses everything to `anyhow`.

## Domain rules that shape the code

- **Event bodies are always stored raw and lossless**, even when decoded — so an unknown format is never lost and new decoders can backfill via `oura redecode`. Don't drop raw bytes.
- **Decoders are ports of the firmware's own logic**, recovered from the native `libringeventparser.so` with Ghidra (see [docs/native-decoder.md](docs/native-decoder.md)) — not guesses. New decoders should cite the native parser fn and be tested against real captured bytes. Status (verified / best-effort / deferred) is tracked in [crates/README.md](crates/README.md).
- **Auth-gated ops** (battery, history events, live HR, features) need the ring's 16-byte app-auth key (hex, one line), re-sent every connection. Pass via `--key-file`. In the Python bench the equivalent is `--auth-key`/`--auth-key-file`.
- **Safety gates:** state-changing and destructive ops (reset / DFU / factory-reset / flight-mode) are hidden behind explicit flags — `--include-state` / `--include-danger` (Python) and gated CLI flags (Rust). Prefer passive read-only requests; never send destructive frames during normal use. Raw `hex:` frames in the Python bench are treated as unclassified and require `--include-danger`.
- **Live channels are power-hungry and teardown is part of the operation.** ACM realtime (`0x06`) is time-boxed and explicitly turned off on exit; RData (`0x03`) is a persistent flash session whose lifecycle is `configure → get_page → stop → clear` (only read/teardown actions are exposed, never "start collection").
- What the ring **cannot** give: the 0–100 Readiness/Sleep/Activity/Stress scores and workout classification are cloud-computed, out of scope by design (see [docs/data-recovery-map.md](docs/data-recovery-map.md)).

## Secrets / gitignored

`captures/`, `reverse/`, `*.key`, `*.db`, `tools/oura_tokens.txt`, and `docs/security-audit/` are gitignored — they may contain serials, MACs, auth keys, health data, or account tokens. Never commit them.

## CLI commands

`scan`, `pair`, `info`, `sync`, `latest`, `live-hr`, `accel`, `viz` (3D motion UI, :8088), `game` (tilt asteroid game, :8089), `features`, `rdata`, `events`, `redecode`, `sleep-analyze`, `sessions`, `subscribe`. Global flags: `--name` (default `Oura`), `--address`, `--scan-timeout`, `--db`, `--key-file`. Full usage + auth-key details in [crates/README.md](crates/README.md).

## Docs index

Protocol reference and reverse-engineering notes live in `docs/` — start with [docs/architecture.md](docs/architecture.md), the [Ring 3 cheatsheet](docs/horizon-ring3-protocol-cheatsheet.md), [docs/android-app-reversing.md](docs/android-app-reversing.md), and [docs/sync-orchestration.md](docs/sync-orchestration.md).
