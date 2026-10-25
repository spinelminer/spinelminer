# Options

This page lists every command-line option of `spinelminer`. `spinelminer
--help` prints the same list. Where an option is given more than once, the last one wins (except
the list options `-o`, `-d` and `--devices-pci`, which add up).

## Modes

Exactly one mode is needed. Without one, the miner exits with code 1 and
`nothing to do: pass -o (pool), --solo, --benchmark or --list-devices (see --help)`.

| Option | Meaning |
|---|---|
| `-o`, `--url <URL>` | Mine in a pool. Repeat for failover backups (the first is the primary). Requires `-u`. Alias: `--pool`. Cannot be combined with `--solo`, `--benchmark` or `--list-devices`. |
| `--solo <ADDR>` | Mine solo for a pearl-gateway at ADDR: a Unix socket path, `unix:<path>` or `host:port` (Windows: `host:port` only). Cannot be combined with `-o`, `--benchmark` or `--list-devices`. |
| `--benchmark` | Mine a synthetic job on every supported GPU (or those chosen with `-d`), check every share on the CPU, and print one summary line per GPU. Never donates. |
| `--list-devices` | Print index, full PCI address, name, gfx and whether each device is supported, then exit. |

## Pool login

| Option | Meaning |
|---|---|
| `-u`, `--user <WALLET>` | Pool login, e.g. `<wallet>` or `<wallet>.<worker>`. Sent as given unless `--worker` appends one. Requires `-o`. Alias: `--wallet`. |
| `--worker <NAME>` | Appends `.NAME` to `-u`. NAME must be non-empty and contain no `.` or `/`, and `-u` must not already contain `.` or `/`. Requires `-o`. |
| `-p`, `--pass <PASS>` | Accepted for flight-sheet compatibility but ignored: no password is sent to the pool. A value other than empty or `x` logs a warning. Requires `-o`. Alias: `--password`. |
| `--pool-dialect <NAME>` | Force the pool protocol dialect for all URLs (default: chosen by host). Needed for a host the miner does not know. Known dialects: `kryptex`. Requires `-o`. |
| `-a`, `--algo <NAME>` | Accepted for flight-sheet compatibility: `pearl`, `pearlhash` or `prl` (any case). Any other name exits with code 1. |

See [Pools](pools.md) for URLs and login formats.

## GPU selection

| Option | Meaning |
|---|---|
| `-d`, `--devices <LIST>` | Comma-separated device indices, e.g. `-d 0,1` (default: all supported discrete GPUs). An index that does not exist or an unsupported device is an error. Integrated GPUs are only used when named here or with `--devices-pci`. |
| `--devices-pci <LIST>` | Select GPUs by PCI address, comma-separated, as `bb:dd.f` or `dddd:bb:dd.f` (domain defaults to `0000`), e.g. `--devices-pci 03:00.0,0000:04:00.0`. Cannot be combined with `-d`. `--list-devices` prints the addresses. |

## Tuning

| Option | Meaning |
|---|---|
| `--autotune <MODE>` | `on` (default): a GPU without a validated kernel (anything but RDNA4) tunes its work shape on first start while mining, for at most 60 s, and caches the result. `off`: mine with the default shape. `redo`: ignore the cache and tune every GPU again, RDNA4 included. |
| `--kernel-report <FILE>` | Write an anonymous JSON kernel report (GPU model, chosen kernel, self-test, auto-tune, speed summary) to FILE 10 minutes after start and at exit. No wallet, pool, worker, host name, IP, path or command line. Send it to the developer if a GPU is slow or has no working kernel. |
| `--telemetry` | Off by default. Send the same report anonymously to the developer, at most once per GPU model, miner version and kernel build. See README "Telemetry". |

The cache is `$XDG_CACHE_HOME/spinelminer/tune.json`, else `~/.cache/spinelminer/tune.json`
(`%LOCALAPPDATA%\SpinelMiner\tune.json` on Windows). An unreadable or unwritable cache is warned
about once and is never fatal.

## Benchmark

These only affect `--benchmark`.

| Option | Meaning |
|---|---|
| `--seconds <SECONDS>` | Duration, at least 1. Default 60. |
| `--share-seconds <SHARE_SECONDS>` | Share target: about one share per this many seconds at 100 TH/s. Default 5. |

## Donation

See [Donation](donation.md) for how the donation works.

| Option | Meaning |
|---|---|
| `--donate <PCT>` | Percent of mining time donated to the developer, 1.0 to 50.0. Default 1.0. Pool and `--solo` runs only. A value outside the range exits with code 1. |
| `--donate-region <REGION>` | Developer pool region: `auto` (default, fastest by connect time), a region name to try first, or `list` to print the region table and exit (needs no pool and no GPU). |

## Stats API

See [Stats API](api.md).

| Option | Meaning |
|---|---|
| `--api-port <PORT>` | Serve JSON stats over HTTP on this TCP port. Pool and `--solo` runs only; ignored with a warning in `--benchmark`. A port that cannot be bound exits with code 1. |
| `--api-host <IP>` | Address the API binds to. Default `127.0.0.1`. Requires `--api-port`. The API has no authentication; a non-loopback address logs a warning. |

## Temperature and GPU health

| Option | Meaning |
|---|---|
| `--temp-limit <C>` | Pause a GPU while its junction temperature (edge if there is no junction sensor) is at or above C °C, allowed range 30-150. Details below. Pool and `--solo` runs only; ignored with a warning in `--benchmark`. On Windows it needs the AMD driver's ADLX sensors (see [Install on Windows](install-windows.md)). |
| `--power-spike-limit <GPU:WATTS,...>` | Stop the whole mining run if a listed GPU reports power above its threshold, e.g. `0:400,1:230`. Off by default. Details below. Pool and `--solo` runs only (an error with `--benchmark`). |
| `--no-watchdog` | Debugging only. Keeps logging GPU watchdog warnings but never resets or disables a GPU for a hang. A truly hung GPU then stalls instead of being taken out of service. |

How `--temp-limit` behaves:

- At or above the limit the GPU launches no new work (work already queued finishes). The log shows
  `GPUn: paused at T °C (limit C)` and the API shows status `paused-temp`.
- It resumes when the temperature is at or below C - 5 **and** at least 10 s after it paused
  (`GPUn: resumed at T °C`).
- A GPU without a readable sensor is warned about once and never paused.
- If a paused GPU's sensor stops reading (or goes stale for 5 s), it stays paused for 30 s, then
  resumes.
- Donation time counts as paused only when every GPU is paused.
- Use it as a safety cap above the card's normal working temperature, not as a throttle. On RDNA4
  the junction temperature drops close to the edge temperature within about 1 s of a pause, so a
  limit below the working junction temperature causes repeated pause/resume cycles (each pause
  lasts at least 10 s).

How `--power-spike-limit` behaves:

- Indices are the ones `--list-devices` prints, also when you select a subset with `-d`. Each
  watt value must be a positive finite number and the GPU must be one of the selected cards;
  otherwise the miner exits with code 1. A card that is not listed has no power threshold.
- Power is sampled once per second. The first reading strictly above the threshold is kept until
  the mining loop stops with an error naming the GPU, the watts and the limit
  (`GPUn: power spike W W exceeds limit L W; stopping mining`). A network operation in progress
  can delay the shutdown until it returns.
- A missing or stale reading never triggers a shutdown; it logs
  `GPUn: no power sensor reading; --power-spike-limit cannot protect this GPU` once per GPU.
- It acts on the sensor-reported power. It does not set a hardware power cap (use your driver or
  overclocking tool for that), and a spike shorter than the sensor's own update interval can be
  missed.
- Config file key: `"power-spike-limit": "0:400,1:230"`.

## Network

| Option | Meaning |
|---|---|
| `--proxy <URL>` | Send every pool connection, developer-pool connection (including the donation fallback on your pool and the region latency probe) through a SOCKS5 proxy: `socks5://[user:pass@]host:port`. `socks5h://` is accepted and means the same: the proxy always resolves host names. `--solo` sockets are never proxied. User name and password are used raw, without `%xx` decoding (write `p@ss`, not `p%40ss`); the user name cannot contain `:`. The password is never logged. |

## Logging and output

| Option | Meaning |
|---|---|
| `--log-file <PATH>` | Also append the log to PATH (same lines as stderr). The 30-second status lines are appended too. With a log file, colour is off on stderr as well. |
| `--log-level <LEVEL>` | `error`, `warn`, `info` (default) or `debug`. The `RUST_LOG` environment variable overrides it when set. |
| `--no-color` | Disable coloured output. Colour is also off when stderr is not a terminal. |
| `--no-tui` | Accepted for compatibility; does nothing (there is no TUI). |

Status lines go to stdout; log lines go to stderr.

## Other

| Option | Meaning |
|---|---|
| `--config <FILE.json>` | Load options from a JSON file (below). |
| `-h`, `--help` | Print help. |
| `-V`, `--version` | Print the version. |

## Environment variables

| Variable | Meaning |
|---|---|
| `SPINEL_HIP_LIB` | Full path of the HIP runtime library to load (`libamdhip64.so` / `amdhip64_7.dll`). Only that file is tried. The bundled Linux launcher sets it. |
| `SPINEL_PREBUILT_HSACO_DIR` | Build time only: directory of prebuilt kernel code objects, used instead of `hipcc`. |
| `PRL_HIP_LIB` | Deprecated alias of `SPINEL_HIP_LIB` (logs one warning). `PRLMINER_PREBUILT_HSACO_DIR` is likewise a deprecated alias of `SPINEL_PREBUILT_HSACO_DIR`. |
| `RUST_LOG` | Overrides `--log-level` when set and non-empty. |

## Config file (`--config`)

`--config FILE.json` reads options from a JSON object whose keys are long option names (without
`--`):

```json
{
  "url": ["stratum+ssl://de.pearl.herominers.com:1200", "stratum+ssl://prl.kryptex.network:8048"],
  "user": "<wallet>",
  "worker": "rig1",
  "api-port": 4067,
  "temp-limit": 90,
  "no-color": true
}
```

Rules:

- A string or number value becomes `--<key>=<value>`.
- `true` sets a flag; `false` and `null` are ignored.
- A list repeats the option, in file order (so failover order is kept).
- Options on the command line override the file. For these groups, giving any member on the
  command line drops the file's entries for the whole group:
  - `url` / `pool` / `solo` / `benchmark` (and `-o`): a `-o` list on the command line replaces the
    file's `url` list; `--solo` on the command line replaces the file's pool URLs;
  - `devices` / `devices-pci` (and `-d`).
- An unknown key exits with code 1 and names it (`unknown option ...`). Invalid JSON, a value that
  is not a string, number or list of them, or a file that cannot be read also exit with code 1.

Keys a config file may set:

| Group | Keys |
|---|---|
| Mode | `url`, `pool`, `solo`, `benchmark` |
| Pool login | `user`, `wallet`, `pass`, `password`, `worker`, `pool-dialect`, `algo` |
| GPUs | `devices`, `devices-pci` |
| Benchmark | `seconds`, `share-seconds` |
| Donation | `donate`, `donate-region` |
| Stats API | `api-port`, `api-host` |
| Health | `temp-limit`, `power-spike-limit`, `no-watchdog` |
| Tuning | `autotune`, `kernel-report`, `telemetry` |
| Network | `proxy` |
| Output | `log-file`, `log-level`, `no-color`, `no-tui` |

`list-devices` and `config` cannot be set from a file.

Keep config files and start scripts private if they contain a proxy password.
