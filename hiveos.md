# HiveOS custom miner setup

<!-- Source: docs/user-guide/hiveos.md; tools/package.sh (HiveOS file name); .github/workflows/release.yml -->
> **Not yet verified on a real HiveOS rig.** The package follows the public HiveOS custom-miner
> conventions and has been tested only against a recorded API response. Full details:
> [docs/hiveos.md](docs/hiveos.md).

The HiveOS package is the release file `spinelminer_hiveos-<ver>.tar.gz` (HiveOS splits the
archive name at the last `-` into name and version, so the version must not contain `-`). It carries
the ROCm 10.1 HIP runtime, so the rig needs no ROCm install.

## Flight sheet

Create a flight sheet with **Miner: Custom** and fill in:

| Field | Value |
|---|---|
| Miner name | `spinelminer_hiveos` |
| Installation URL | `https://github.com/spinelminer/spinelminer/releases/download/v<ver>/spinelminer_hiveos-<ver>.tar.gz` (for 0.4.0 only, the file is named `spinelminer-0.4.0.tar.gz`: `.../releases/download/v0.4.0/spinelminer-0.4.0.tar.gz`; from 0.4.1 it is `spinelminer_hiveos-<ver>.tar.gz`) |
| Hash algorithm | not used by the package; leave the default |
| Wallet and worker template | `<wallet>.%WORKER_NAME%` with `<wallet>` replaced by your PRL address (`prl1...`) |
| Pool URL | e.g. `stratum+ssl://de.pearl.herominers.com:1200`; several URLs separated by spaces become failover pools, in order |
| Pass | optional; ignored (the miner sends pools no password) |
| Extra config arguments | extra command-line options, e.g. `--temp-limit 85`, `-d 0,1` or `--telemetry`; see [docs/options.md](docs/options.md) |

The package always adds `--api-port 4067 --log-file <log basename>.log --no-color`; extra arguments
come last and override those defaults. Check the download against the `SHA256SUMS` file of the same
release (`spinelminer_hiveos-<ver>.tar.gz.sha256` is also published).

## Stats and logs

The HiveOS dashboard reads the miner's stats API at `/api/v1/summary` (needs `curl` and `jq`, present
on HiveOS). The log is `<CUSTOM_LOG_BASENAME>.log`, by default
`/var/log/miner/custom/spinelminer_hiveos.log`. Do not add `--api-host` unless you want the API (which has no
authentication) reachable from the network.
