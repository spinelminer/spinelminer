# SpinelMiner user guide

SpinelMiner is a GPU miner for Pearl (PRL) on AMD graphics cards. The program is called
`spinelminer` (`spinelminer.exe` on Windows).

It can:

- mine in a pool over Stratum (`-o`, with failover to backup pools),
- mine solo through your own `pearl-gateway` (`--solo`),
- run a benchmark on a synthetic job (`--benchmark`),
- serve JSON statistics over HTTP (`--api-port`).

## Supported GPUs

| GPU | Architecture | Status |
|---|---|---|
| RX 9060 series | gfx1200 (RDNA4) | validated |
| RX 9070 series | gfx1201 (RDNA4) | validated |
| RDNA3 / RDNA3.5 | gfx1100-gfx1103, gfx1150-gfx1153 | kernels included, never run on that hardware |
| CDNA | gfx908, gfx90a, gfx942, gfx950 | kernels included, never run on that hardware |
| Vega / RDNA1 / RDNA2 (no matrix units) | gfx900, gfx906, gfx1010-gfx1012, gfx1030-gfx1036 | kernels included, never run on that hardware |

Every GPU listed in the table is reported as "supported" by `--list-devices`. Other devices are
listed as unsupported and skipped. Integrated GPUs (APU iGPUs) are only used when you name them
with `-d` or `--devices-pci`.

Before a GPU mines, the miner runs a short self-test of its mining kernel (unvalidated kernels in a child process). If a kernel fails, the
next kernel for that architecture is tried (see [Troubleshooting](troubleshooting.md#kernel-self-test)).

VRAM use: about 6.1 GiB on an RX 9070 XT and about 3.1 GiB on an RX 9060 XT, plus small buffers.

## Contents

| Page | What it covers |
|---|---|
| [Install on Linux](install-linux.md) | OS support, tarballs, driver, permissions |
| [Install on Windows](install-windows.md) | Release zip, driver, HIP runtime, ADLX sensors, limits (partly untested) |
| [HiveOS](hiveos.md) | Custom-miner package, flight sheet, stats |
| [mmpOS](mmpos.md) | Custom-miner package (unverified) |
| [Pools](pools.md) | Hosts, ports, TLS and example command lines per pool |
| [Options](options.md) | Every command-line option and the `--config` JSON file |
| [Stats API](api.md) | `/api/v1/summary` and `/api/v1/gpus` |
| [PCIe risers](pcie-risers.md) | host<->GPU traffic, x1 riser feasibility |
| [Donation](donation.md) | The developer donation (dev fee) |
| [Troubleshooting](troubleshooting.md) | Error messages and what to do |
| [FAQ](faq.md) | Short answers to common questions |

## Quick start

```
spinelminer --list-devices
spinelminer --benchmark --seconds 60
spinelminer -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1
```

Replace `<wallet>` with your own PRL address (`prl1...`). Stop the miner with Ctrl+C.

Every 30 seconds the miner prints one status line per GPU and a total, for example:

```
GPU0 AMD Radeon RX 9070 XT: 138.2 TH/s, accepted 12, rejected 0, stale 0
Total: 138.2 TH/s, effective 131.0 TH/s, pool de.pearl.herominers.com, latency 26 ms
```

- `accepted` and `rejected` are the pool's verdicts. `stale` shares were dropped because their
  job had been replaced.
- `effective` is accepted shares x difficulty over time, which is how pool dashboards compute
  hashrate. It is noisy at first and moves towards the local TH/s over hours, not minutes.
- `dev A/R` is added once the developer donation has shares (see [Donation](donation.md)).

In `--solo` mode each GPU line shows `TH/s, candidates C, local rejects R` instead.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | error: bad arguments, an invalid pool URL or `--solo` address, an unknown pool dialect, every GPU disabled (not all by the watchdog), every pool refusing the login in one full round, a share that failed the local check in `--benchmark`, and similar |
| 2 | HIP runtime not found or failed to initialize |
| 3 | no usable (supported) GPU, or the GPU driver check failed |
| 4 | donation blocked (see [Donation](donation.md#stop-rule-exit-code-4)) |
| 5 | GPU hang: the watchdog disabled every GPU (see [Troubleshooting](troubleshooting.md#exit-code-5-gpu-hang)) |
