# SpinelMiner

<!-- Source: README.md, docs/user-guide/, RESULTS.md and quantus/README.md in the private repo KL0nLutiy/spinel-miner -->
Closed-source GPU miner for **Pearl (PRL)** and **Quantus (QTC)** on AMD graphics cards, tuned for
the RX 9070 and RX 9060 series. This repository holds no source code: only documentation and
[GitHub Releases](https://github.com/spinelminer/spinelminer/releases). The use of the binaries is
governed by [LICENSE.txt](LICENSE.txt).

| Coin | Algorithm | Select with | Modes |
|---|---|---|---|
| Pearl (PRL) | PearlHash | default (`-a pearl`) | pool, solo (`--solo`), benchmark |
| Quantus (QTC) | QPoW (Poseidon2) | `-a quantus` (also `qtc`, `qpow`) | pool, benchmark |

- [Speed](#speed) · [Supported GPUs](#supported-gpus) · [Download](#download) ·
  [Quick start](#quick-start) · [Pools](#pools) · [Features](#features) · [Donation](#donation) ·
  [Telemetry](#telemetry) · [Documentation](#documentation)

## Speed

<!-- Source: RESULTS.md "M7", "M8"; CHANGELOG.md 0.5.0 -->
| GPU | Pearl (PRL) | Quantus (QTC) |
|---|---|---|
| RX 9070 XT | 153.4 TH/s | 339.1 MH/s |
| RX 9060 XT 16 GB | 84.1 TH/s | 186.7 MH/s |

Measured on the developer's rig with the overclock below; these are not stock speeds. Pearl:
`--benchmark --seconds 120`, both GPUs at once, all shares verified. Quantus: live pool mining.
Your numbers depend on the card, cooling, settings and driver. Earlier versions are in
[CHANGELOG.md](CHANGELOG.md).

Overclock used (LACT), the same for both coins:

| GPU | Power limit | GPU clock offset | Voltage offset |
|---|---|---|---|
| RX 9070 XT | 280 W | +300 MHz | -100 mV |
| RX 9060 XT | 170 W | +150 MHz | -90 mV |

Every card is different: if shares get rejected, reduce the undervolt by 10-20 mV.

## Supported GPUs

<!-- Source: README.md "Supported GPUs", docs/user-guide/index.md -->
| GPU | Architecture | Pearl | Quantus |
|---|---|---|---|
| RX 9070 series | gfx1201 (RDNA4) | validated | validated |
| RX 9060 series | gfx1200 (RDNA4) | validated | validated |
| RDNA3 / RDNA3.5 | gfx1100-gfx1103, gfx1150-gfx1153 | **unvalidated** | **unvalidated** |
| CDNA | gfx908, gfx90a, gfx942, gfx950 | **unvalidated** | **unvalidated** |
| Vega / RDNA1 / RDNA2 | gfx900, gfx906, gfx1010-gfx1012, gfx1030-gfx1036 | **unvalidated** | **unvalidated** |

"Unvalidated" means kernels are included but were never run on that hardware, and they are not
tuned or benchmarked. The miner protects you from a wrong kernel:

- **Pearl**: every GPU runs a known-answer self-test of its kernel at startup (about 25 ms on RDNA4;
  unvalidated kernels run it in a child process, so a crashing kernel cannot take the miner down)
  and falls back to the next kernel for that GPU if it fails. Unvalidated GPUs then auto-tune their
  work shape once (`--autotune`).
- **Quantus**: unvalidated architectures use a portable kernel (about 60% of the tuned speed when
  run on RDNA4). Such a GPU hashes 4096 nonces at startup and is skipped unless they match the CPU
  reference. Every share is re-hashed on the CPU before it is submitted.

The only unvalidated device tried so far is an RDNA2 integrated GPU (gfx1036): it works but is not
worth mining on (about 0.5 TH/s Pearl, 5 MH/s Quantus). Integrated GPUs are never used by default,
only when named with `-d` / `--devices-pci`.

VRAM for Pearl: about 6.1 GiB on an RX 9070 XT or a 16 GB RX 9060 XT, about 3.1 GiB on an 8 GB
RX 9060 XT, plus small buffers.

## Download

<!-- Source: tools/package.sh finish() naming; .github/workflows/release.yml "Release notes" table -->
Pick a file from the [latest release](https://github.com/spinelminer/spinelminer/releases/latest).
Replace `<ver>` with the version, for example `0.5.0`; release tags are `v<ver>`.

| File | For |
|---|---|
| `spinelminer-<ver>-linux.tar.xz` | Linux, ROCm runtime included, no ROCm install needed (start with `./spinelminer`) |
| `spinelminer_hiveos-<ver>.tar.gz` | HiveOS custom miner |
| `spinelminer-<ver>-mmpos.tar.gz` | mmpOS |
| `spinelminer-<ver>-windows.zip` | Windows (AMD Adrenalin driver with HIP runtime) |
| `SHA256SUMS` | checksums of all of the above |

Direct URLs have the form
`https://github.com/spinelminer/spinelminer/releases/download/v<ver>/<file>`. Each file also has a
`<file>.sha256`; verify a download with `sha256sum -c SHA256SUMS --ignore-missing`. Linux binaries
need glibc >= 2.28.

## Quick start

<!-- Source: docs/user-guide/install-linux.md, install-windows.md, hiveos.md, mmpos.md -->
Replace `<wallet>` with your own address (PRL `prl1...`, QTC `qz...`). Stop the miner with Ctrl+C.

### Linux

No ROCm install needed, only the amdgpu kernel driver with `/dev/kfd` (kernel 6.14 or newer for
RX 9000 cards, or AMD's amdgpu-dkms) and your user in the `render` and `video` groups
(`sudo usermod -aG render,video $USER`, then log in again). Details:
[Install on Linux](docs/install-linux.md).

```
tar xf spinelminer-<ver>-linux.tar.xz
cd spinelminer-<ver>
./spinelminer --list-devices
./spinelminer --benchmark --seconds 60
```

Pearl:

```
./spinelminer -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1 --api-port 4067
```

Quantus:

```
./spinelminer -a quantus -o stratum+ssl://de.quantus.herominers.com:1155 -u <wallet> --worker rig1 --api-port 4067
```

### Windows

Install a current AMD Adrenalin driver (it provides the HIP runtime), unzip
`spinelminer-<ver>-windows.zip`, then run the same commands with `spinelminer.exe`:

```
spinelminer.exe --list-devices
spinelminer.exe -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1
spinelminer.exe -a quantus -o stratum+ssl://de.quantus.herominers.com:1155 -u <wallet> --worker rig1
```

Pearl mining was tested natively on one Windows 11 rig (RX 9070 XT, RX 9060 XT). `--solo`, CDNA
cards and Quantus mining are not yet tested on Windows. GPU temperature, fan RPM and power come
from the AMD driver's ADLX library, so the stats API, `--temp-limit` and `--power-spike-limit` work
on Windows too. See [Install on Windows](docs/install-windows.md).

### Start scripts

The Linux and Windows packages include ready-to-edit start scripts, one per pool
(`start_herominers`, `start_kryptex`, `start_luckypool`, ... as `.sh` / `.bat`), described in
`START-SCRIPTS.md` inside the package. The Quantus ones are `start_quantus_herominers` and
`start_quantus_kryptex`.

### HiveOS

Custom miner, installation URL
`https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer_hiveos-<ver>.tar.gz`.
Not yet verified on a real HiveOS rig. Steps: [hiveos.md](hiveos.md), details:
[docs/hiveos.md](docs/hiveos.md).

### mmpOS

Custom (external) miner package `spinelminer-<ver>-mmpos.tar.gz`. Unverified against mmpOS: see
[docs/mmpos.md](docs/mmpos.md).

## Pools

<!-- Source: docs/user-guide/pools.md -->
None of these pools needs an account: your wallet address is the login.

| Coin | Pools checked live |
|---|---|
| Pearl | HeroMiners, Kryptex, PearlFortune, LuckyPool (with optional NOCK merge mining), AlphaPool |
| Quantus | HeroMiners, Kryptex |

Repeat `-o` for failover backups. Hosts, ports, TLS and example command lines:
[docs/pools.md](docs/pools.md). Pearl can also be mined solo through your own pearl-gateway
(`--solo`).

## Features

<!-- Source: docs/user-guide/options.md, api.md -->
- JSON stats API (`--api-port`): [docs/api.md](docs/api.md).
- GPU temperature limit (`--temp-limit`) and power spike limit (`--power-spike-limit`); Pearl only
  for now.
- SOCKS5 proxy (`--proxy`), log file (`--log-file`), JSON config file (`--config`).
- GPU selection by index or PCI address (`-d`, `--devices-pci`).
- Pearl mining needs almost no PCIe bandwidth, so x1 risers are fine:
  [docs/pcie-risers.md](docs/pcie-risers.md).

Every option: [docs/options.md](docs/options.md).

## Donation

<!-- Source: docs/user-guide/donation.md -->
SpinelMiner donates **1% of its mining time** to the developer by default. This is announced at
every start (`Donation: 1.0% of mining time to the developer (dev pools: HeroMiners, Kryptex; see
README)`) and logged at every window. `--benchmark` and `--list-devices` never donate.

- **Rate**: `--donate <PCT>`, default and minimum 1.0, maximum 50.0.
- **Schedule**: the first window starts 3 to 5 minutes after the run's first job, then there is one
  window per 100 minutes of mining, 60 s long at 1%.
- **In a window** the GPUs mine for the developer on a developer pool (HeroMiners first, then
  Kryptex). Your own pool session stays connected and your own shares never go to the developer.
- **Developer addresses**: Pearl `prl1plpy7056ym0cyhk2agqdvq9kmkjamk745jfqv69hktd9djj0cu3pqg2csjt`,
  Quantus `qznqVouET7NKFoL5fi57jLCjVKqaJ4ixXaqFs18XhrQy4VSJ7` (login `<address>.devfee`).
- **Stop rule**: if 3 windows in a row, over at least 10 minutes, deliver nothing while your own
  pool is working, the miner stops with exit code 4 and says so. A broken network or an outage of
  your own pool never stops mining.

Regions, the fallback through your own pool and the exact rules: [docs/donation.md](docs/donation.md).

## Telemetry

<!-- Source: docs/user-guide/telemetry.md -->
Off by default: nothing is sent and no connection is made unless you pass `--telemetry`. With it,
the miner sends an anonymous kernel report at most once per GPU architecture, miner version and
kernel: GPU model and architecture, which kernel ran, self-test and auto-tune results, and a
hashrate and temperature summary, plus a random install ID used only to de-duplicate reports.

**Never sent**: wallet, pool address, user or worker name, password, host name, IP address, file
paths, the command line, or any error message text.

Every field and the timing: [docs/telemetry.md](docs/telemetry.md).

## Documentation

User guide: [docs/index.md](docs/index.md) (install, pools, options, stats API, donation,
telemetry, troubleshooting, FAQ). Changes per version: [CHANGELOG.md](CHANGELOG.md).

## Support

Bug reports and feature requests: [GitHub issues](https://github.com/spinelminer/spinelminer/issues)
(use the templates; never post your wallet or pool password). Vulnerabilities: see [SECURITY.md](SECURITY.md).
