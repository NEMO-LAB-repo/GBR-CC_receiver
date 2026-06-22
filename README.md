# msquic-cloud-server

This repository contains the cloud-side MsQuic/secnetperf receiver used with the
GBR-CC sender repository:

```text
https://github.com/QiangWu769/msquic_cellular
```

Use `msquic-cloud-server` on the remote cloud machine to build and run the
`secnetperf` server. Use `msquic_cellular` on the phone/host sender side for
CellNinjia, DIAG parsing, GBR ratio calculation, and the GBR-CC-enabled
`secnetperf` client.

## Repository Roles

```text
Android phone + host sender                    Cloud server
--------------------------------               ------------------------------
msquic_cellular                                msquic-cloud-server

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
git clone https://github.com/QiangWu769/msquic-cloud-server.git
cd msquic-cloud-server
git submodule update --init --recursive
```

Existing checkout:

```bash
cd /home/qwu26/msquic-cloud-server
git submodule sync --recursive
git submodule update --init --recursive
```

### 2. Build `secnetperf`

Use the same build method as the sender-side `msquic_cellular` repository. Only
the working directory is different.

```bash
cd /home/qwu26/msquic-cloud-server
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
/home/qwu26/msquic-cloud-server/build/bin/Release/secnetperf
```

### 3. Start the cloud receiver

Run this on the cloud server:

```bash
cd /home/qwu26/msquic-cloud-server
./build/bin/Release/secnetperf -port:4433 -exec:maxtput -cc:bbr
```

`secnetperf` runs as a server when `-target:<host>` is not specified. Keep this
process running while the phone/host sender runs the upload experiment.

If the cloud firewall is enabled, allow UDP port `4433`.

### 4. Start the paired sender

On the sender machine, use the companion repository:

```bash
cd /home/qwu26/msquic_cellular
./build/bin/Release/secnetperf \
  -target:<cloud-server-ip> \
  -port:4433 \
  -cc:bbr \
  -upload:20mb \
  -ptput:1 \
  -cellular:1
```

Before running the sender command, `msquic_cellular` should already have:

1. `cellninjia_mobile` running on the phone.
2. `adb forward tcp:43555 tcp:43555` configured.
3. One host parser running:
   - `tools/cellninjia/diag_get_lte_msquic.py`, or
   - `tools/cellninjia/diag_get_5g_msquic.py`.

## Logging Notes

This repository keeps the cloud-side MsQuic/BBR logging and analysis utilities.
Typical receiver logs can be redirected from the `secnetperf` server process:

```bash
./build/bin/Release/secnetperf -port:4433 -exec:maxtput -cc:bbr \
  > server_secnetperf.log 2>&1
```

Phone-side DIAG logs, GBR ratio logs, and sender-side GBR-CC control logs belong
to the companion `msquic_cellular` repository.

## Main Files

- `src/perf/lib/SecNetPerfMain.cpp`: `secnetperf` command-line entry point.
- `src/perf/lib/PerfServer.cpp`: `secnetperf` server/receiver implementation.
- `src/core/bbr.c`: BBR congestion-control implementation used by receiver-side tests.
- `src/core/bbr_packet_level_logging.c`: packet-level BBR logging support.
- `src/perf/readme.md`: upstream `secnetperf` usage reference.
- `bbr_logs/`: historical BBR experiment logs and generated plots.
