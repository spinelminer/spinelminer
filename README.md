# SpinelMiner

<!-- Source: README.md (intro), docs/user-guide/index.md in the private repo KL0nLutiy/spinel-miner -->
Closed-source GPU miner for Pearl (PRL) on AMD graphics cards. This repository holds no source code:
only documentation and [GitHub Releases](https://github.com/spinelminer/spinelminer/releases).
The use of the binaries is governed by [LICENSE.txt](LICENSE.txt).

SpinelMiner mines in a pool over Stratum (`-o`, with failover), solo through your own
pearl-gateway (`--solo`), or runs a synthetic-job benchmark with locally verified shares
(`--benchmark`). It has a JSON stats API, a GPU temperature limit, an optional power spike limit, SOCKS5 proxy support and
HiveOS / mmpOS packages.

## Supported GPUs

<!-- Source: README.md "Supported GPUs", docs/user-guide/index.md, RESULTS.md "Measured (TH/s)" rows O2 and O3 -->
| GPU | Architecture | Status |
|---|---|---|
| RX 9070 series | gfx1201 (RDNA4) | validated |
| RX 9060 series | gfx1200 (RDNA4) | validated |
| RDNA3 / RDNA3.5 | gfx1100-gfx1103, gfx1150-gfx1153 | **unvalidated**: kernels included, never run on that hardware |
| CDNA | gfx908, gfx90a, gfx942, gfx950 | **unvalidated**: kernels included, never run on that hardware |
| Vega / RDNA1 / RDNA2 (no matrix units) | gfx900, gfx906, gfx1010-gfx1012, gfx1030-gfx1036 | **unvalidated**: kernels included, never run on that hardware |

**Measured speeds (RDNA4, `--benchmark --seconds 120`, both GPUs at once, all shares verified):**

| Setting | RX 9070 XT | RX 9060 XT |
|---|---|---|
| stock power limits (330 W / 170 W caps), earlier kernel build (RESULTS.md row O2) | 132.4 TH/s | 69.0 TH/s |
| **with the owner's -50 mV VDDGFX undervolt on both cards**, 0.4.0 kernel build (row O3) | 139.2 TH/s | 70.8 TH/s |
| **0.4.1**, owner's settings (300 W / 170 W limits, undervolt), `--seconds 90` (RESULTS.md M6) | **146.1 TH/s** | **79.6 TH/s** |

These come from one run each on the developer's rig. The 139.2 / 70.8 and 146.1 / 79.6 figures were
measured with the owner's undervolt and power limits and are not stock speeds; there is no stock-settings
measurement of the 0.4.0 or 0.4.1 kernel builds. Your numbers depend on the card, cooling, undervolt and driver.

Other AMD architectures are not benchmarked. At startup every GPU runs a known-answer self-test of
its kernel (about 25 ms on RDNA4; unvalidated kernels run it in a child process, so a crashing kernel
cannot take the miner down) and falls back to the next kernel for that GPU if it fails. Unvalidated
GPUs then auto-tune their work shape once (`--autotune`). The only unvalidated device tested so far is
an RDNA2 iGPU (gfx1036): it passes the self-test but mines about 0.5 TH/s, which is not worth it.
Integrated GPUs (APU iGPUs) are never used by default, only when named with `-d` / `--devices-pci`.

VRAM: about 6.1 GiB on an RX 9070 XT and about 3.1 GiB on an RX 9060 XT, plus small buffers.

## Download

<!-- Source: tools/package.sh finish() naming; .github/workflows/release.yml "Release notes" table -->
Pick a file from the [latest release](https://github.com/spinelminer/spinelminer/releases/latest).
Replace `<ver>` with the version, for example `0.4.1`; release tags are `v<ver>`.

| File | For |
|---|---|
| `spinelminer-<ver>-linux.tar.xz` | Linux, ROCm runtime included, no ROCm install needed (start with `./spinelminer`) |
| `spinelminer_hiveos-<ver>.tar.gz` | HiveOS custom miner |
| `spinelminer-<ver>-mmpos.tar.gz` | mmpOS |
| `spinelminer-<ver>-windows.zip` | Windows (AMD Adrenalin driver with HIP runtime) |
| `SHA256SUMS` | checksums of all of the above |

Direct URLs:

```
https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer-<ver>-linux.tar.xz
https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer_hiveos-<ver>.tar.gz
https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer-<ver>-mmpos.tar.gz
https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer-<ver>-windows.zip
https://github.com/spinelminer/spinelminer/releases/download/v<ver>/SHA256SUMS
```

Each file also has a `<file>.sha256`. Verify a download with
`sha256sum -c SHA256SUMS --ignore-missing`. Linux binaries need glibc >= 2.28.

## Quick start

<!-- Source: docs/user-guide/install-linux.md, install-windows.md, hiveos.md, mmpos.md -->
Replace `<wallet>` with your own PRL address (`prl1...`). Stop the miner with Ctrl+C.

### Linux

No ROCm install needed: only the amdgpu kernel driver (kernel 6.14+ for RX 9000). You need it with `/dev/kfd` (kernel 6.14 or newer for RX 9000 cards, or AMD's
amdgpu-dkms) and your user in the `render` and `video` groups
(`sudo usermod -aG render,video $USER`, then log in again). Details: [Install on Linux](docs/install-linux.md).

```
tar xf spinelminer-<ver>-linux.tar.xz
cd spinelminer-<ver>
./spinelminer --list-devices
./spinelminer --benchmark --seconds 60
./spinelminer -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1 --api-port 4067
```

### Windows

Tested natively on one Windows 11 rig (RX 9070 XT, RX 9060 XT); `--solo` and CDNA cards are not
yet tested there. GPU temperature, fan RPM and power come from the AMD driver's ADLX library, so
the stats API, `--temp-limit` and `--power-spike-limit` work on Windows too. See
[Install on Windows](docs/install-windows.md). Install a current AMD Adrenalin driver (it provides
the HIP runtime), unzip `spinelminer-<ver>-windows.zip`, then:

```
spinelminer.exe --list-devices
spinelminer.exe --benchmark --seconds 60
spinelminer.exe -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1
```

### HiveOS

Custom miner, installation URL
`https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer_hiveos-<ver>.tar.gz`.
Not yet verified on a real HiveOS rig. Steps: [hiveos.md](hiveos.md), details: [docs/hiveos.md](docs/hiveos.md).

### mmpOS

Custom (external) miner package `spinelminer-<ver>-mmpos.tar.gz`. Unverified against mmpOS:
see [docs/mmpos.md](docs/mmpos.md).

## Pools

<!-- Source: README.md "Pools" table, docs/user-guide/pools.md -->
Live 1 h runs were clean on Kryptex, PearlFortune, LuckyPool, HeroMiners and AlphaPool (2026-10-20);
longer dashboard checks were still pending. See [docs/pools.md](docs/pools.md) for hosts and ports.

## Donation

<!-- Source: README.md "Donation" section, verbatim -->

SpinelMiner donates a share of its mining time to the developer. This is on by default, announced at
every start (`Donation: 1.0% of mining time to the developer (dev pools: HeroMiners, Kryptex; see
README)`) and logged at every window. It applies to pool (`-o`) and `--solo` runs; `--benchmark` and
`--list-devices` never donate.

- **Rate**: `--donate <PCT>`, default 1.0, minimum 1.0, maximum 50.0. Raise it to give more.
- **Schedule**: the first window starts 3 to 5 minutes after the run's first job, then there is one
  window per 100 minutes of mining. A window lasts `PCT / 100 x 6000` seconds, e.g. 60 s at 1%.
  Nothing is stored between runs.
- **What happens in a window**: the GPUs mine for the developer on a developer pool; your own pool
  session stays connected, and any new jobs are applied when the window ends. The log shows
  `donation window: <s> s on <pool> <region> (<rtt> ms)` and `donation window done: accepted A,
  rejected R`; status lines gain `dev A/R` once the developer has shares.
- **Developer pools and regions**: HeroMiners first, then Kryptex. `--donate-region list` prints one
  line per region (`<name>  <pool>  <url>`) and exits; it needs no pool and no GPU.
  `--donate-region auto` (default) picks the fastest region by connect time, measured by a
  background probe that never affects your mining; `--donate-region <NAME>` tries that region first.
- **Developer address**: `prl1plpy7056ym0cyhk2agqdvq9kmkjamk745jfqv69hktd9djj0cu3pqg2csjt`
  (login `<address>.devfee`).
- **Fallback**: if every developer pool fails, the miner logs in to your own pool a second time as
  the developer wallet (`<address>.devfee`) and donates there. A route only counts once it sends a
  job within 10 s. If that fails too, the window is retried (300 s apart). In `--solo` mode there is
  no pool login, so there is no such fallback.
- **Stop rule**: if 3 donation windows in a row, over at least 10 minutes, deliver nothing (neither a
  developer pool nor the fallback works) while your own pool is working, the miner stops with exit
  code 4 and says so. "Working" means: connected throughout and at least one of your shares accepted
  since the first failed window; with `--solo`, every gateway poll succeeded and at least one job on a
  new block arrived since the first failed window. A broken network or an outage of your own pool
  never stops mining.

## Telemetry

<!-- Source: README.md "Telemetry" section, verbatim -->

Off by default. Nothing is sent and no connection is made to the report service unless you pass
`--telemetry` (or set `"telemetry": true` in `--config`; HiveOS/mmpOS: add `--telemetry` to the extra
arguments). It then announces itself at start with `Telemetry: on (anonymous kernel reports; see
README "Telemetry")`. A build without a report endpoint compiled in (`SPINEL_TELEMETRY_URL` at build
time) exits with code 1 and `telemetry is not configured in this build`.

**What is sent** is the `--kernel-report` document, split so that each report covers the GPUs of one
architecture (`gfx`) and kernel build:

- `schema` (report format), and `miner`: `version`, `kernel_hash` (BLAKE3 of the kernel code object),
  `os` (e.g. `linux-x86_64`), and `hip_runtime` / `driver` when known;
- `gpus[]`, per GPU: `gfx`, `name` (model), `cu_count`, `pci_device_id`, `vram_mb`, `variant` (the mining
  kernel) and `variant_status` (`validated` / `unvalidated`), `fallback_path` (kernels that failed the
  self-test first: `variant`, `status`, `code`, and the failing `lane`, `hip_error` name, `signal` or
  `exit_code`), `selftest` (`result`, `code`, `ms`), `tune` (`state`, `candidates` as `shape` +
  `mac_per_s`, `chosen`), `summary` (`th_s`, `slice_ms` p50/p95/max, `local_rejects`, `watchdog_events`,
  `temp_pauses`, `max_junction_c`);
- the HTTP header `X-Install-Id`: a random UUID created on first use and stored in `install-id` in the
  cache directory (`$XDG_CACHE_HOME/spinelminer/`, else `~/.cache/spinelminer/`;
  `%LOCALAPPDATA%\SpinelMiner\` on Windows). It only lets the service de-duplicate reports and limit
  each install to 10 a day.

**Never sent**: wallet, pool address, user or worker name, password, host name, IP address, file paths,
the command line, or any error message text (failures are codes). The service sees the connecting IP
address like any web server; it is used only for rate limiting and never stored.

**When**: 10 minutes after start the miner asks the service (`GET /v1/need?gfx=&ver=&kernel=`) whether
it still needs reports for that GPU architecture, miner version and kernel; only if so it sends one
(`POST /v1/report`, gzip, at most 32 KB) and records it in `telemetry-sent.json`, so each combination is
sent at most once per install. A GPU on which every kernel fails the self-test is reported at once,
without asking, once per install per architecture, mining kernel variant (the first one it tried) and
miner version. A report covers at most 16 GPUs (the first 16 of that architecture); a report too large
for 32 KB first loses the auto-tune candidates, then all but 4 failed-kernel entries per GPU, then GPUs.

Everything runs in the background over HTTPS (through `--proxy` when set) with a 5 s timeout. A network
error or server error (5xx) is retried once 10 minutes later; a refusal (any 4xx, including the rate
limit 429) is not retried, and a report the service rejects as invalid (400) or too large (413) is
recorded as sent so it is not tried again. Send errors are logged at debug level only; a cache file
(`install-id`, `telemetry-sent.json`) that cannot be read or written is warned about once (the install
ID then lasts only that run). Mining is never delayed. At exit the miner waits at most 12 s for reports
still queued or being sent; Ctrl-C (SIGINT/SIGTERM) still stops it at once.

## Documentation

User guide: [docs/index.md](docs/index.md) (install, pools, options, stats API, donation,
troubleshooting, FAQ).

## Support

Bug reports and feature requests: [GitHub issues](https://github.com/spinelminer/spinelminer/issues)
(use the templates; never post your wallet or pool password). Vulnerabilities: see [SECURITY.md](SECURITY.md).
