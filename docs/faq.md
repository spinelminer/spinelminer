# FAQ

**Which GPUs work?**
RX 9060 series (gfx1200) and RX 9070 series (gfx1201) are validated. Kernels for RDNA3/3.5, CDNA
and Vega/RDNA1/RDNA2 are included but have never run on that hardware. See [the index](index.md#supported-gpus).
NVIDIA GPUs and CPUs are not supported.

**Do I need to install ROCm?**
Not with the bundled Linux tarball or the HiveOS / mmpOS packages: they carry the HIP runtime.
The slim Linux tarball needs the ROCm 6.x or 7.x HIP runtime (`libamdhip64`). You always need the
amdgpu kernel driver. See [Install on Linux](install-linux.md).

**Is the binary called `prlminer` or `spinelminer`?**
`spinelminer`. The miner was called `prlminer` before it was renamed SpinelMiner; the start
scripts still pick up an old `prlminer` binary placed next to them.

**Which pool should I use?**
Any of the pools listed as working in [Pools](pools.md): HeroMiners, Kryptex, PearlFortune,
LuckyPool, AlphaPool. Pick the nearest region and add a backup with a second `-o`.

**Why is the pool password ignored?**
The login used by every supported pool has no password field, so the miner never sends one. `-p`
is accepted so flight sheets keep working, and a value other than empty or `x` logs a warning.

**How do I set the worker name?**
Either `-u <wallet>.<worker>` or `-u <wallet> --worker <worker>`. Not both.

**Why does the pool show a lower hashrate than the miner?**
Pools estimate hashrate from accepted shares, which is noisy. The miner's `effective` figure in the
status line is computed the same way; over hours it approaches the local TH/s.

**What are stale shares?**
Shares found on a job that the pool had already replaced (usually a new block). They are dropped,
not sent.

**How much is the dev fee? Can I turn it off?**
1.0 % of mining time by default, at least 1.0 %, at most 50.0 %. It cannot be turned off or
lowered below 1.0 %. See [Donation](donation.md).

**Why did the miner exit with code 4?**
Donations could not reach any developer pool (or the developer wallet on your pool) in 3 windows
in a row while your own pool worked, usually because a firewall or DNS filter blocks the developer
pool hosts. See [Donation](donation.md#stop-rule-exit-code-4).

**Why did the miner exit with code 5?**
The watchdog disabled every GPU because their work never finished: the GPU or driver is hung.
Reboot the rig. See [Troubleshooting](troubleshooting.md#exit-code-5-gpu-hang).

**How do I limit GPU temperature?**
`--temp-limit <C>` pauses a GPU at that junction temperature and resumes 5 °C lower. Use it as a
safety cap above normal working temperature, not as a throttle. On Windows it works where the AMD driver's ADLX library reports the temperature. See
[Options](options.md#temperature-and-gpu-health).

**How do I see stats from another program?**
Start with `--api-port 4067` and read `http://127.0.0.1:4067/api/v1/summary`. See [Stats API](api.md).

**Can I mine through a SOCKS5 proxy?**
Yes: `--proxy socks5://host:port` (with `user:pass@` if needed). All pool and developer-pool
connections go through it; `--solo` gateway connections do not.

**Can I keep my options in a file?**
Yes: `--config options.json`. See [Options](options.md#config-file---config).

**Can I mine solo?**
Yes, with your own `pearld` and `pearl-gateway`: `--solo <gateway address>`. Blocks pay the
gateway's mining address. See [Pools](pools.md#solo-mining).

**Is there a TUI or a remote control interface?**
No. `--no-tui` is accepted and does nothing. The stats API is read-only.

**Can it stop if a GPU's power spikes?**
Yes: `--power-spike-limit 0:400,1:230` stops the whole run when GPU 0 or 1 reports more than that
many watts. It is off by default and does not set a hardware cap. See
[Options](options.md#temperature-and-gpu-health).

**Do I need a fast PCIe slot or can I use x1 risers?**
Bandwidth is not the limit: the miner moves well under 1 MB/s per GPU. See [PCIe risers](pcie-risers.md).

**Does it work on Windows?**
Yes. The Windows build has run on real Radeon cards (RX 9070 XT, RX 9060 XT); `--solo` and CDNA cards are not yet tested there. Temperature, fan RPM and power come from the AMD driver's ADLX library. See
[Install on Windows](install-windows.md).
