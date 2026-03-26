# RuView ESP32 Mesh Bring-Up Breakdown (March 2026)

This document is a full postmortem-style breakdown of the ESP32-to-Rust-server bring-up session.

## Goal

Establish reliable end-to-end sensing pipeline:

`ESP32-S3 CSI nodes -> UDP -> Rust sensing-server -> API/UI`

Target final state:

- 3 provisioned nodes streaming concurrently
- server accepts and parses ADR-018 frames correctly
- `/api/v1/sensing/latest` returns fused multi-node updates (`node_count > 1`)

## Starting Conditions

- Server on Ubuntu (`192.168.0.54`) listening on UDP `5005`
- ESP32 firmware flashed/provisioned and WiFi connected
- Heartbeat path eventually working, CSI path intermittent

## High-Level Root Causes We Found

1. **Toolchain/environment mismatch on macOS shell**
   - `idf.py` failed with `No module named 'esp_idf_monitor'` while project `.venv` was active.
2. **Firmware runtime contention and edge tier pressure**
   - `edge_dsp` watchdog stalls when running non-zero edge tiers under current runtime load.
3. **Server parser mismatch with ADR-018 header layout**
   - frame header fields decoded with wrong sizes/offsets in sensing-server.
4. **No fusion window in server output path**
   - API exposed latest single-node frame only (`node_count: 1`) even with multi-node ingest.

## Troubleshooting Timeline

1. Confirmed server UDP port open and manual UDP test OK.
2. `idf.py build` failed on firmware due to SHA include path (`mbedtls/sha256.h`).
3. Added send-path observability in firmware (`CSI UDP send ok/fail`, aggregate stats).
4. Added tiny UDP heartbeat payload to separate transport health from CSI health.
5. Verified heartbeat reached server (`UDP length 8`) while CSI was inconsistent.
6. Disabled WiFi power-save (`WIFI_PS_NONE`) to improve CSI callback behavior.
7. Observed intermittent CSI payload packets (`148`, `404`, `632`) proving network path was healthy.
8. Observed `edge_dsp` watchdog trigger loops on some nodes.
9. Switched operational provisioning to `--edge-tier 0` for stability.
10. Fixed ESP-IDF shell usage (`source ~/esp/esp-idf/export.sh`, not project `.venv`).
11. Confirmed 3-node concurrent UDP ingest by source IP in `tcpdump`.
12. Fixed Rust server frame parser to ADR-018 header field sizes.
13. Implemented 100 ms fusion window in UDP receiver path.
14. Deployed updated server to Ubuntu and verified fused output (`node_count: 3`, node IDs `[3,1,2]`).

## Firmware Code Changes (What + Why)

Source base: `RuView/firmware/esp32-csi-node/main/`

### 1) `rvf_parser.c` - SHA implementation migration

- Replaced direct `mbedtls/sha256.h` usage with PSA crypto hashing.
- Added helper hash function and PSA-based incremental hash path for signature verification.

Why:

- Avoid build break caused by unavailable/relocated public MbedTLS SHA header in current ESP-IDF environment.
- Keep hash behavior explicit and toolchain-compatible.

### 2) `csi_collector.c` - CSI/UDP observability and liveness

- Added detailed send logging around `stream_sender_send(...)`.
- Added periodic aggregate counters (`cb`, `ok`, `fail`, `rate_skip`, `edge_drop`, `tier`).
- Added promiscuous callback counters/logging (`Promisc rx #...`).
- Enabled `dump_ack_en` in CSI config.
- Added callback timestamp tracking and public getter accessors.

Why:

- Distinguish among: no callback, callback no send, send failures, rate limiting, ring pressure.
- Rapidly identify where the pipeline stalls in field runs.

### 3) `csi_collector.h` - exported liveness accessors

- Added function declarations for callback count and last callback timestamp.

Why:

- Let `main.c` perform health checks without exposing internals.

### 4) `edge_processing.c` - watchdog mitigation

- Increased loop yield delay and lowered `edge_dsp` task priority.

Why:

- Reduce starvation of `IDLE1` and mitigate task watchdog resets under heavy processing.

### 5) `main.c` - heartbeat, power-save policy, and status signals

- Added 8-byte UDP heartbeat at fixed interval.
- Disabled WiFi modem power save after STA connect (`WIFI_PS_NONE`).
- Added CSI liveness warning logs (`No CSI callbacks for ...`).
- Added optional RGB status LED framework (disabled unless pins configured).

Why:

- Heartbeat gives always-on transport sanity signal.
- Power save can suppress callback cadence in sensing scenarios.
- LED + logs make field diagnostics possible without deep tooling.

## Server Code Changes (What + Why)

Source base: `RuView/rust-port/wifi-densepose-rs/crates/wifi-densepose-sensing-server/src/main.rs`

### 1) ADR-018 parser alignment

- Changed `Esp32Frame.n_subcarriers` to `u16`.
- Changed `Esp32Frame.freq_mhz` to `u32`.
- Updated parse offsets for sequence/rssi/noise accordingly.

Why:

- Firmware uses ADR-018 layout (`u16`/`u32` fields), parser previously decoded with narrower types.
- Misdecode blocked consistent frame acceptance and downstream updates.

### 2) Multi-node fusion window

- Added fusion window constant (`100 ms`).
- Added per-node pending frame map in UDP receiver.
- Added batch processor to emit one merged `SensingUpdate` containing all nodes seen in the window.
- Added node position mapping helper.

Why:

- Without windowing, API only shows latest single node (`node_count: 1`).
- Mesh behavior requires temporal coalescing before classification/publication.

## Operational Commands Used (Canonical)

### ESP-IDF shell fix (macOS)

```bash
deactivate 2>/dev/null || true
source "$HOME/esp/esp-idf/export.sh"
idf.py --version
```

### Firmware build/flash

```bash
idf.py -D CONFIG_DISPLAY_ENABLE=n build
idf.py -p /dev/cu.usbserial-XXXX flash
```

### Provision example (node 3)

Note: TDM slot is zero-based for total=3 (`0,1,2`).

```bash
python provision.py \
  --port /dev/cu.usbserial-XXXX \
  --ssid "<SSID>" \
  --password "<PASSWORD>" \
  --target-ip 192.168.0.54 \
  --target-port 5005 \
  --node-id 3 \
  --tdm-slot 2 \
  --tdm-total 3 \
  --edge-tier 0
```

### Server deployment (Ubuntu)

```bash
cd ~/repos/RuView/rust-port/wifi-densepose-rs
cargo build -p wifi-densepose-sensing-server --release --no-default-features
./target/release/sensing-server --source esp32 --udp-port 5005 --http-port 3000 --ws-port 3001 --bind-addr 0.0.0.0
```

## Validation Evidence

### Network-level (UDP)

- Simultaneous traffic from three source IPs to `192.168.0.54:5005`.
- Packet lengths included heartbeat and CSI/vitals frame sizes (`8`, `32`, `148`, `404`, `632`).

### API-level (fusion)

- Before fusion-window patch: `node_count` always `1` with rotating node IDs.
- After patch + deploy: fused `latest` response showed `node_count: 3`, `node_ids: [3,1,2]`.

## Final State

- 3 nodes provisioned with distinct IDs and TDM slots.
- Stable ingest achieved with `edge-tier 0`.
- Server parser corrected for ADR-018 header.
- Multi-node fusion active in API output.

## Known Follow-Ups

1. Make display-disable behavior deterministic in build config (not just CLI override).
2. Add automatic CSI recovery path on prolonged callback stalls.
3. Move node positions to runtime config (instead of hardcoded mapping).
4. Add metrics endpoint for fusion window stats (frames per node per batch, drop/late counts).
