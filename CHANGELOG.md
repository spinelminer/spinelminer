# Changelog

## 0.4.2

<!-- Source: Cargo.toml version = "0.4.2"; RESULTS.md "M7 0.4.2" -->
- Faster RDNA4 mining, same power draw. Kernel: 4 waves per SIMD (A fragments loaded in 4 parts),
  each accumulator's 4 WMMA steps chained back to back, serpentine tile order, and 256-row blocks
  on the RX 9070 series so pairs of waves share B through the L0 cache. Engine: the Merkle root of A
  reuses its cached chunk groups, B noise is generated once per job, and a host-visible hit flag
  replaces per-slice counter copies (non-mining GPU time ~1.7% -> ~0.7%).
- Developer's rig, same overclock for both versions (`--benchmark --seconds 120`, both GPUs, 2
  rounds, all shares verified): RX 9070 XT 149.4 -> **153.4 TH/s** (+2.6%), RX 9060 XT 81.8 ->
  **83.5 TH/s** (+2.1%).
- Overclock used (LACT): RX 9070 XT 280 W power limit, +300 MHz GPU clock offset, -100 mV
  voltage offset; RX 9060 XT 170 W, +150 MHz, -90 mV. Against the previous -60 / -50 mV at the
  same power limits that adds +3.2% / +3.2%. Every card is different: lower the undervolt if
  shares are rejected.
- Checked on HeroMiners: 7 h at the earlier 300 W settings, 604 accepted / 0 rejected / 1 stale;
  50 min with the overclock above, 87 accepted / 0 rejected, rig 236.8 TH/s.

## 0.4.1

<!-- Source: Cargo.toml version = "0.4.1"; RESULTS.md "M6 wave64 kernel and A = B = 64 operands" -->
- Faster RDNA4 mining: new wave64 kernel (`mine_rdna4_w64`, 8 x 4 WMMA tiles per wave) and committed
  operand matrices filled with 64 instead of 0, so the noised operands are never negative and the
  card switches fewer bits per multiply. Power-capped cards clock higher at the same power. On the
  developer's rig (owner's power limits and undervolt, `--benchmark --seconds 90`, both GPUs):
  RX 9070 XT 135.6 -> **146.1 TH/s** (+7.7%), RX 9060 XT ~71 -> **79.6 TH/s** (+12%).
- Checked on HeroMiners for 6 h: 539 accepted, 0 rejected, 2 stale; pool-credited rate within 0.5%
  of the local rate. The previous kernel (`mine_rdna4`) stays as the self-test fallback.
- HiveOS package renamed to `spinelminer_hiveos-<ver>.tar.gz` (top folder `spinelminer_hiveos/`, miner name `spinelminer_hiveos`, log `/var/log/miner/custom/spinelminer_hiveos.log`), following the SRBMiner/BzMiner convention so HiveOS derives the miner name from the file name. Release 0.4.0 shipped it as `spinelminer-0.4.0.tar.gz`; reinstall with the new URL when updating.

## 0.4.0

<!-- Source: Cargo.toml version = "0.4.0"; git log of KL0nLutiy/spinel-miner; docs/superpowers/specs (m3, m4a, m4b, m4c, windows-adlx-sensors); RESULTS.md; README.md -->
First release. The miner was called `prlminer` during development and renamed SpinelMiner
(`spinelminer`) on 2026-10-21.

### Mining

- Pool mining over Stratum (`-o`, `stratum+ssl://` and `stratum+tcp://`) with failover to backup
  URLs and primary-pool probing; solo mining through a pearl-gateway (`--solo`); `--benchmark` on a
  synthetic job with every share verified on the CPU.
- Pool login dialect `kryptex`, used by Kryptex, HeroMiners, PearlFortune, LuckyPool and AlphaPool
  (live 1 h runs clean on 2026-10-20; longer checks pending). PearlHash, GrandPool, F2Pool, Akoya
  and MinePRL are untested or unusable, see the user guide's Pools page.
- Same-height job continuity for pools that accept superseded jobs; per-rig row offsets so rigs on
  identical headers rarely collide.

### GPUs and speed

- Validated on RDNA4: RX 9070 series (gfx1201) and RX 9060 series (gfx1200). Measured 132.4 / 69.0
  TH/s (RX 9070 XT / RX 9060 XT) at stock power limits, `--benchmark --seconds 120`.
- Per-GPU tuning of the RDNA4 kernel (rasterization group per GPU, k = 8192 on gfx1201, later kernel
  changes worth about +0.3% to +0.9%).
- Unvalidated kernels for RDNA3 / RDNA3.5, CDNA, Vega, RDNA1 and RDNA2 (never run on that
  hardware). Startup self-test of every GPU's kernel with a fallback chain (unvalidated kernels are
  tested in a child process), and on-device auto-tune with a local cache (`--autotune`).
- Integrated GPUs are only used when named with `-d` / `--devices-pci`.

### Options and tooling

- `--api-port` / `--api-host`: JSON stats API (`/api/v1/summary`, `/api/v1/gpus`).
- `--temp-limit`, `--proxy` (SOCKS5), `--config` (JSON), `--log-file`, `--log-level`, `--worker`,
  `--devices-pci`, `--no-watchdog`.
- `--power-spike-limit <GPU:WATTS,...>`: optional per-GPU power limit that stops the whole run.
- `--kernel-report <file>`: anonymous JSON report of kernel and speed data.
- `--telemetry`: opt-in (off by default) anonymous kernel reports.
- Windows GPU sensors read through AMD's ADLX library (hot-spot or edge temperature, memory
  temperature, power, fan RPM and clocks where the driver reports them; `fan_percent` stays
  `null`). The stats API, HiveOS-style stats and `--temp-limit` therefore work on Windows. Without
  ADLX the sensors read `null` and the miner warns that `--temp-limit` cannot protect the GPU.
- Per-GPU PCIe traffic counter: host-to-device and device-to-host copy rates (MB/s) in the status
  lines, the stats API (`pcie_h2d_mbps`, `pcie_d2h_mbps`) and the kernel report. See the user guide's
  PCIe risers page.

### Developer donation

- 1.0% of mining time by default (`--donate`, range 1.0 to 50.0), with developer pools and region
  selection, a fallback to your own pool, and a stop rule (exit code 4). See the README.

### Packages

- Linux package (`spinelminer-<ver>-linux.tar.xz`, ROCm runtime bundled and stripped, no slim build; use `SPINEL_HIP_LIB` for your own ROCm), HiveOS custom-miner package, mmpOS package (both `.tar.gz`), Windows zip.
  Linux binaries are built against glibc 2.28. Kernels are built with ROCm 10.1 / HIP 7.16.
- HiveOS, mmpOS and Windows packages have not been verified on real HiveOS, mmpOS or Windows
  hardware at the time of this release.
