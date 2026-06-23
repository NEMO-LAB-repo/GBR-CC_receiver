# GBR-CC_receiver

This repository is part of the implementation for the SIGCOMM 2026 paper
**"Synchronizing with the Scheduler: Dual-Loop Congestion Control for 5G Uplink
on Commodity Devices."**

This repository contains the cloud-side MsQuic/secnetperf receiver used with the
GBR-CC sender repository:

```text
https://github.com/NEMO-LAB-repo/GBR-CC_sender
```

## Project Architecture

```text
GBR-CC_sender
    -> QUIC uplink upload
    -> GBR-CC_receiver cloud server
    -> secnetperf receiver
    -> receiver-side logs and analysis
```

```text
GBR-CC_receiver
+-- src/perf/
|   +-- secnetperf receiver entry point and experiment logging
+-- src/core/
|   +-- MsQuic congestion-control code used by the receiver build
+-- scripts/
|   +-- Build and run helpers
+-- analysis/
|   +-- Receiver-side log analysis utilities
+-- cmake/
|   +-- MsQuic build configuration
+-- submodules/
    +-- Third-party dependencies used by MsQuic
```

Use `GBR-CC_receiver` on the remote cloud machine to build and run the
`secnetperf` server. Use `GBR-CC_sender` on the phone/host sender side for
CellNinjia, DIAG parsing, GBR ratio calculation, and the GBR-CC-enabled
`secnetperf` client.

## Repository Roles

```text
Android phone + host sender                    Cloud server
--------------------------------               ------------------------------
GBR-CC_sender                                 GBR-CC_receiver

cellninjia_mobile reads modem DIAG     --->    secnetperf receiver listens
host parser computes GBR ratio                 on UDP/QUIC port 4433
GBR-CC sender runs secnetperf client
```

The cloud server does not run CellNinjia and does not compute GBR. It only
receives the QUIC upload traffic and provides the remote `secnetperf` endpoint
for experiments.

## Quick Start

### 1. Clone and initialize submodules

```bash
git clone https://github.com/NEMO-LAB-repo/GBR-CC_receiver.git
cd GBR-CC_receiver
git submodule update --init --recursive
```

Existing checkout:

```bash
cd /home/qwu26/GBR-CC_receiver
git submodule sync --recursive
git submodule update --init --recursive
```

### 2. Build `secnetperf`

Use the same build method as the sender-side `GBR-CC_sender` repository. Only
the working directory is different.

```bash
cd /home/qwu26/GBR-CC_receiver
rm -rf build
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DQUIC_BUILD_PERF=ON \
  -DQUIC_ENHANCED_PACKET_LOGGING=ON \
  -DQUIC_TLS_LIB=openssl
cmake --build build --target secnetperf -j"$(nproc)"
```

Expected binary:

```text
/home/qwu26/GBR-CC_receiver/build/bin/Release/secnetperf
```

### 3. Start the cloud receiver

Run this on the cloud server:

```bash
cd /home/qwu26/GBR-CC_receiver
./build/bin/Release/secnetperf -port:4433 -exec:maxtput -cc:bbr
```

`secnetperf` runs as a server when `-target:<host>` is not specified. Keep this
process running while the phone/host sender runs the upload experiment.

If the cloud firewall is enabled, allow UDP port `4433`.

### 4. Start the paired sender

On the sender machine, use the companion repository:

```bash
cd /home/qwu26/GBR-CC_sender
./build/bin/Release/secnetperf \
  -target:<cloud-server-ip> \
  -port:4433 \
  -cc:bbr \
  -upload:20mb \
  -ptput:1 \
  -cellular:1
```

Before running the sender command, `GBR-CC_sender` should already have:

1. `cellninjia_mobile` running on the phone.
2. `adb forward tcp:43555 tcp:43555` configured.
3. One host parser running:
   - `tools/cellninjia/diag_get_lte_msquic.py`, or
   - `tools/cellninjia/diag_get_5g_msquic.py`.

## Main Files

- `src/perf/lib/SecNetPerfMain.cpp`: `secnetperf` command-line entry point.
- `src/perf/lib/PerfServer.cpp`: `secnetperf` server/receiver implementation.
- `src/core/bbr.c`: BBR congestion-control implementation used by receiver-side tests.
- `src/core/bbr_packet_level_logging.c`: packet-level BBR logging support.
- `src/perf/readme.md`: upstream `secnetperf` usage reference.
- `bbr_logs/`: historical BBR experiment logs and generated plots.
