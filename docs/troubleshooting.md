# Troubleshooting

Messages are quoted as the miner prints them. Fatal errors start with `error:` and are printed to
stderr (and to the `--log-file`, if set). `GPUn` stands for the device index.

## Driver and `/dev/kfd` messages

On Linux the miner checks the GPU compute driver before it loads the HIP runtime. These checks
exit with code 3.

**`GPU compute driver not found (/dev/kfd). Install the amdgpu kernel driver with ROCm support (kernel >= 6.14 for RX 9000 cards, or AMD's amdgpu-dkms); see README "Linux support".`**

The amdgpu driver is not loaded, or it was built without compute (KFD) support.

- RX 9000 (RDNA4) cards need kernel 6.14 or newer, or AMD's amdgpu-dkms package. On Ubuntu 24.04
  install the HWE kernel; on Ubuntu 22.04 install amdgpu-dkms. See [Install on Linux](install-linux.md).
- Check that the module is loaded: `lsmod | grep amdgpu`, and that `/dev/kfd` exists.
- In a container, pass `/dev/kfd` and `/dev/dri` into it.

**`no permission for /dev/kfd: add your user to the render and video groups`**

```
sudo usermod -aG render,video $USER
```

Then log out and back in (or reboot). Check with `groups`.

**`this kernel's amdgpu driver does not support <GPU> (<gfx>); install a newer kernel (>= 6.14 for RX 9000 cards) or AMD's amdgpu-dkms; see README "Linux support".`**

The miner supports the card, but the running kernel's amdgpu driver is too old to drive it. HIP
then sees no GPU. Use a newer kernel or amdgpu-dkms. This hint is appended to
`error: no GPU found: ...` or `error: the HIP runtime reports no devices`.

The `/dev/kfd` check is skipped when `SPINEL_HIP_LIB` is set or under WSL2 (`/dev/dxg` present); HIP
then reports any problem itself.

## HIP runtime messages

**`HIP runtime not found; install ROCm 6.x or 7.x (libamdhip64) or set SPINEL_HIP_LIB to its path; tried: ...`** (exit code 2)

- Use the Linux package (`spinelminer-<ver>-linux.tar.xz`) and start it with the `spinelminer` launcher, which needs no
  ROCm install; or install the ROCm 6.x / 7.x HIP runtime; or set `SPINEL_HIP_LIB` to the full path
  of `libamdhip64.so`.
- On Windows the text says `install an AMD Adrenalin driver or the HIP SDK (ROCm 6.x or 7.x, amdhip64_*.dll)`;
  see [Install on Windows](install-windows.md).

**`error: no GPU found: ...`** or **`error: the HIP runtime reports no devices`** (exit code 3)

The runtime loaded but sees no GPU. Check the driver messages above and `--list-devices`.

**`error: no supported GPU found`** (exit code 3)

No device is supported, or none could be opened. `--list-devices` shows which devices the miner
considers supported. Integrated GPUs are skipped unless you name them with `-d` or
`--devices-pci` (log: `GPUn <name>: integrated GPU skipped (select it with -d n to mine on it)`).

## Device selection messages

| Message | Cause |
|---|---|
| `device N does not exist (valid indices: 0..M); see --list-devices` | `-d` names an index that does not exist (exit 1) |
| `device N <name> (<gfx>) is not supported (supported: ...)` | `-d` names an unsupported device (exit 3) |
| `no device at PCI address <addr>; available:` followed by the device list | `--devices-pci` names an address with no GPU (exit 1) |
| `invalid --devices-pci entry "...": expected bb:dd.f or dddd:bb:dd.f such as 03:00.0` | malformed PCI address (exit 1) |
| `cannot open device N: ...` | HIP could not open that device; the miner continues with the others |

## Kernel self-test

Before a GPU mines, its mining kernel runs a known-answer job and must reproduce the CPU result
exactly. Validated (RDNA4) kernels are tested in the miner process; unvalidated kernels are tested
in a child process, so a hang or crash there fails only that kernel. If a kernel fails, the next
kernel for that architecture is tried (the last fallback on gfx11 and CDNA is the `mine_dot`
kernel). On a GPU without a validated kernel, `--autotune` (default `on`) then tunes its work shape
once and caches it; see [Options](options.md#tuning).

| Log line | Meaning |
|---|---|
| `GPUn: kernel <name> passed self-test in <ms> ms` | normal |
| `GPUn: kernel <name> failed self-test (<reason>); trying <next>` | that kernel failed; a fallback is tried |
| `GPUn: kernel <name> failed self-test (<reason>)` | the last kernel in the chain failed |
| `GPUn: no working kernel for <gfx>; run with --kernel-report <file> and send it to the developer` | every kernel failed; the GPU is disabled |

`<reason>` is one of:

- `hash mismatch at lane N` or `wrong hit set`: the kernel computed a wrong result;
- `launch error: ...`: the kernel could not be loaded or started;
- `timeout after 2 s`: the kernel did not finish. For a validated (RDNA4) kernel, which is tested
  in the miner process, nothing more is tested on that GPU (the remaining kernels report
  `launch error: not tested: the GPU timed out on an earlier kernel`), the device is not reset, and
  the GPU is disabled. An unvalidated kernel is tested in a child process, which is killed on
  timeout, so only that kernel fails and the next one is tried.

What to do:

- A failing self-test on gfx1200 / gfx1201 (validated cards) usually points at the driver, an
  unstable overclock or undervolt, or a hung GPU. Return to stock clocks and reboot.
- On the unvalidated architectures (RDNA3, CDNA, Vega/RDNA1/RDNA2) a failure may be a real kernel
  problem. Report it with the log.
- Note: the build this guide was checked against has **no `--kernel-report` option**, even
  though the message mentions it. Send the log lines above (run with `--log-file` to capture them)
  instead.

A GPU with no working kernel is skipped (pool mode logs `GPUn <name>: failed to start, skipping it: ...`)
and the others keep mining. If no GPU can start, the run ends with exit code 1
(`error: no GPU engine could be started` in pool and `--solo` mode).

## Temperature limit

| Log line | Meaning |
|---|---|
| `GPUn: paused at T °C (limit C)` | junction temperature reached `--temp-limit`; no new work is launched |
| `GPUn: resumed at T °C` | temperature at or below C - 5 and at least 10 s after the pause |
| `GPUn: no temperature sensor reading; --temp-limit C cannot protect this GPU` | no readable sensor (on Windows: ADLX unavailable or the GPU not visible to it); the GPU is never paused |
| `GPUn: resumed: no temperature reading for 30 s` | the sensor stopped reading while paused; the GPU resumes after 30 s |
| `--temp-limit T is ignored with --benchmark (pool and --solo runs only)` | warning |
| `error: --temp-limit T is outside 30-150 degrees C` | exit 1 |

**The GPU keeps pausing and resuming.** The limit is below the card's normal working junction
temperature. On RDNA4 the junction temperature falls close to the edge temperature within about
1 s of a pause, so the card resumes and heats up again. Set the limit a few degrees above the
normal working temperature, as a safety cap, or improve cooling. Each pause lasts at least 10 s.

## Power spike limit

| Message | Meaning |
|---|---|
| `GPUn: power spike W W exceeds limit L W; stopping mining` | a reading went above the `--power-spike-limit` threshold; the run ends with an error |
| `GPUn: no power sensor reading; --power-spike-limit cannot protect this GPU` | no readable power sensor (on Windows: ADLX unavailable); that GPU is never checked |
| `error: --power-spike-limit is supported only with pool or --solo mining` | used with `--benchmark` or without a mode (exit 1) |
| `--power-spike-limit: GPUn is not a selected mining device` | the index is not among the GPUs in use (see `--list-devices`, `-d`); exit 1 |

If the miner stops on a spike that is normal for your card, raise the threshold well above the
highest power you see in the API (`power`) or the logs.

## Watchdog

The watchdog notices GPU work that does not finish.

| Log line | Meaning |
|---|---|
| `GPUn: slice took X ms (expected Y ms); N slow slices since the last warning` | work finished but slowly; only reported |
| `GPUn: slice not done after X ms (expected Y ms for the queued work): ResetWorker` | work is stuck; the GPU's worker is reset |
| `GPUn: resetting worker` | the reset is happening |
| `host-wide stall: all N GPUs' work stalled for X ms; reset, not counted against any GPU` | every GPU stalled at once (e.g. another card's ring reset, a PCIe or VM pause, an overloaded host); GPUs are reset, nothing is disabled |
| `GPUn disabled: GPUn watchdog: queued slices made no progress (3 resets within 10 minutes)` | the GPU hung repeatedly and was taken out of service |
| `GPUn: slice not done after X ms (...); --no-watchdog: no reset` | `--no-watchdog` is set; nothing is reset |

A GPU is also disabled if more than 3 of its shares fail the local CPU check within 10 minutes.

### Exit code 5: GPU hang

```
error: every GPU was disabled by the watchdog (its work never completed): the GPU or driver is hung; reboot the rig
```

Every GPU was disabled by the watchdog. The GPU or the driver is hung, and only a **reboot**
reliably recovers it. Then check for unstable overclocks or undervolts, power or riser problems,
and the kernel/driver version. A stall that hits all GPUs at once is treated as a host-wide stall
and never causes exit code 5. With `--no-watchdog` no GPU is disabled for hangs (it then stalls
instead).

If GPUs were disabled for other reasons (or some by the watchdog and some not), the run ends with
`error: all GPUs disabled` and exit code 1.

### Exit code 4: donation blocked

See [Donation](donation.md#stop-rule-exit-code-4). Allow the developer pool hosts named in the
message through your firewall, DNS filter or proxy.

## Pool messages

| Message | Cause and fix |
|---|---|
| `invalid pool url "...": expected ...` | the `-o` value is malformed; use `stratum+ssl://host:port`, `stratum+tcp://host:port` or `host:port` (exit 1) |
| `no dialect for host <host>; use --pool-dialect <name> (known: kryptex)` | the miner does not know this pool host; add `--pool-dialect kryptex` if the pool speaks the Kryptex-style protocol (exit 1). See [Pools](pools.md). |
| `unknown pool dialect <name> (known: kryptex)` | wrong `--pool-dialect` value (exit 1) |
| `every pool refused the login: <url>: <reason>; ...` | every URL refused the login in one full round; check the wallet / login format for that pool (exit 1) |
| `pool <url>: disconnected: ...` | the connection dropped; the miner reconnects and fails over after 3 failures in a row |
| `-p/--pass is ignored by the kryptex dialect: no password is sent to the pool` | warning only; `-p` is never sent |
| `--worker "..." must be non-empty and contain no '.' or '/'` | bad `--worker` value (exit 1) |
| `--worker cannot be combined with a -u that already contains a worker ('.' or '/')` | put the worker in `-u` or in `--worker`, not both (exit 1) |
| `GPUn: dropped N valid hits since the last warning (...); the share target is easier than the proof threads can serve` | the pool's difficulty is very low for this hashrate; use a port with a higher start difficulty if the pool offers one |

A TLS URL to a pool with a self-signed certificate (PearlFortune, LuckyPool, PearlHash) fails
certificate verification. Use `stratum+tcp://` for those pools.

## Stats API messages

| Message | Cause |
|---|---|
| `cannot start the stats API on <addr>: <reason> (check --api-host / --api-port)` | the port is in use or the address is not on this host (exit 1) |
| `the stats API on <addr> has no authentication and shows the pool user (wallet.worker) to anyone who can reach it` | warning for a non-loopback `--api-host` |
| `--api-port is ignored with --benchmark (pool and --solo runs only)` | warning |

## Proxy errors

At startup, a malformed `--proxy` exits with code 1:

```
error: invalid proxy url "socks5://user:***@host": expected `socks5://[user:pass@]host:port`
```

The password is always shown as `***`. Common causes: a scheme other than `socks5://` or
`socks5h://`, a missing or zero port, a user name without a password, or a user name containing
`:`.

While mining, connection errors through the proxy read `SOCKS5 proxy <host:port>: <step>: <detail>`:

| Detail | Meaning |
|---|---|
| `connect: ...` / `timed out` | the proxy itself is not reachable |
| `not a SOCKS5 proxy (version byte N)` | the address is not a SOCKS5 proxy (e.g. an HTTP proxy) |
| `proxy requires username/password but none was given` | add `user:pass@` to `--proxy` |
| `username/password rejected by proxy` | wrong credentials. They are used raw: write `p@ss`, not `p%40ss`. |
| `proxy accepts none of the offered authentication methods` | the proxy wants an authentication method the miner does not support |
| `proxy could not reach <host>:<port>: <reason> (code N)` | the proxy is fine but cannot reach the pool; `<reason>` is e.g. `connection refused`, `host unreachable`, `network unreachable` or `connection not allowed by ruleset` |

The proxy resolves host names (as with `socks5h://`). `--solo` gateway connections never use the
proxy.

## Other messages

| Message | Cause |
|---|---|
| `nothing to do: pass -o (pool), --solo, --benchmark or --list-devices (see --help)` | no mode given (exit 1) |
| `error: unsupported algorithm "..."; this miner supports only pearl (aliases: pearlhash, prl)` | wrong `-a` value (exit 1) |
| `error: invalid --solo address: ...` | malformed `--solo` value; on Windows also a Unix socket address (exit 1) |
| `gateway requires certificate version N, which this miner does not support` | the gateway is newer than this miner; update the miner (exit 1) |
| `cannot read --config <path>: ...`, `--config <path>: invalid JSON: ...`, `--config <path>: unknown option ...` | config file problem (exit 1); see [Options](options.md#config-file---config) |
| `cannot open --log-file <path>: ...` | the log file cannot be created (exit 1) |
