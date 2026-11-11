# Telemetry

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
