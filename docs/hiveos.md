# HiveOS

SpinelMiner ships as a HiveOS **custom miner** package, `spinelminer_hiveos-<ver>.tar.gz`.

> **Unverified on a real HiveOS rig.** The package follows the public HiveOS custom-miner
> conventions (`h-manifest.conf`, `h-config.sh`, `h-run.sh`, `h-stats.sh`) and has been tested
> only against a recorded API response. Items marked *Unverified* below have not been checked on
> HiveOS itself.

## What is in the package

- `spinelminer`: a launcher that runs `spinelminer.bin` against the bundled `lib/` directory.
  `lib/` holds the ROCm 10.1 HIP runtime and its dependencies, so the rig needs no ROCm install.
  It still needs the amdgpu kernel driver, which HiveOS provides.
- `h-manifest.conf`, `h-config.sh`, `h-run.sh`, `h-stats.sh`: the HiveOS glue scripts.
- `LICENSE` (when present), `THIRD-PARTY-LICENSES`, `THIRD-PARTY-LICENSES-RUST`.

## Flight sheet

Create a flight sheet with **Miner: Custom** and fill in:

| Field | Value |
|---|---|
| Miner name | `spinelminer_hiveos` |
| Installation URL | the URL of `spinelminer_hiveos-<ver>.tar.gz` (for 0.4.0 only, the file is `spinelminer-0.4.0.tar.gz`). HiveOS splits the archive name at the last `-` into name and version, so the version must not contain `-`. |
| Wallet and worker template | `wallet.%WORKER_NAME%`, with `wallet` replaced by your PRL address |
| Pool URL | e.g. `stratum+ssl://de.pearl.herominers.com:1200`. Several URLs separated by spaces become failover pools, in order. |
| Pass | optional; ignored. The miner sends pools no password and logs a warning for a value other than empty or `x`. |
| Extra config arguments | extra command-line options, e.g. `--temp-limit 85` or `-d 0,1`. See [Options](options.md). |

How the fields become a command line (`h-config.sh`):

- Each pool URL becomes one `-o <URL>`. An empty Pool URL is an error.
- The wallet template becomes `-u <template>` (a warning is printed if it is empty).
- A non-empty Pass becomes `-p <pass>`.
- `--api-port 4067 --log-file <log basename>.log --no-color` is always added.
- The Extra config is split like a shell command line: quotes are respected, nothing is evaluated
  (no variables, no globbing). Unbalanced quotes make `h-config.sh` fail and keep the previous
  config. Extra arguments come last, so they override the defaults above (for options given more
  than once, the last one wins).

The generated arguments are written one per line to the config file (mode 600), and `h-run.sh`
reads them back unchanged, so wallets and options with spaces or `*` survive. If the config file
is missing or empty, `h-run.sh` exits with code 1 without starting the miner.

Note: the arguments, including any password, are visible to other users of the rig in `ps`.

## Logs

The miner appends its log, including the 30-second status lines, to the file given by
`--log-file`: `<CUSTOM_LOG_BASENAME>.log`. HiveOS sets `CUSTOM_LOG_BASENAME`; outside HiveOS the
manifest's default is `/var/log/miner/custom/spinelminer_hiveos`, i.e.
`/var/log/miner/custom/spinelminer_hiveos.log`. Colour codes are off (`--no-color`), so the file is plain
text.

## Stats in the HiveOS dashboard

`h-stats.sh` reads the miner's [stats API](api.md) at `/api/v1/summary`:

- Address: the last `--api-port` / `--api-host` in the generated arguments, default
  `127.0.0.1:4067`. `--api-host 0.0.0.0` is read as `127.0.0.1`.
- It needs `curl` and `jq` (both present on HiveOS).
- It sets:
  - `khs`: total hashrate (H/s) / 1000;
  - `stats`: `hs` (per-GPU hashrate in kH/s), `hs_units` = `khs`, `temp`, `fan`, `uptime`, `ver`,
    `ar` (`[accepted, rejected]`), `algo`, `bus_numbers`.
- `bus_numbers` are decimal PCI bus numbers (`0x0b` -> 11), as HiveOS expects.
- If the API is down or the reply is not a summary: `khs=0` and empty `stats`.
- A GPU without a readable sensor reports `null` temperature and fan, not 0. *Unverified:* how the
  HiveOS agent displays `null`.

To change the API port, put `--api-port <N>` in Extra config; `h-stats.sh` follows it.

## Notes

- To keep the stats API reachable only from the rig, do not add `--api-host`. The API has no
  authentication and shows your pool user (`wallet.worker`).
- The manifest's `CUSTOM_CONFIG_FILENAME` / `CUSTOM_LOG_BASENAME` defaults are only used when
  the scripts run outside the HiveOS agent; on HiveOS the agent sets both.
