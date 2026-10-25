# Install on Windows

> **Tested on one rig.** The Windows build has run natively on Windows 11 with an Adrenalin driver
> (RX 9070 XT, RX 9060 XT and a Ryzen iGPU): device listing, the self-test, `--benchmark`, pool
> mining with the stats API and `--temp-limit`. **Not yet run on Windows:** `--solo` mining, CDNA
> cards, and GPU sensors on GPU generations other than RDNA4 and an RDNA2 iGPU. Please report what
> you see.

The program is `spinelminer.exe`. It loads the AMD HIP runtime (`amdhip64_*.dll`) at run time, so
it needs no ROCm installation of its own.

## Download

Get `spinelminer-<ver>-windows.zip` from the GitHub release page and verify it with
`SHA256SUMS`. It holds `spinelminer.exe`, the README, `LICENSE.txt`, `docs/windows.md`, the five `.bat`
start scripts and `START-SCRIPTS.md`.

## 1. Install the AMD driver

Install a current **AMD Software: Adrenalin Edition** driver.

- AMD's Windows system-requirements tables list the Radeon RX 9070 / 9070 XT / 9070 GRE (gfx1201)
  as supported from HIP SDK 6.4.2 on. Check AMD's table for gfx1200 (RX 9060) cards.
- A 7.x HIP runtime is preferred. AMD's 7.1.1 release notes list an intermittent HIP memory-API
  failure on RX 9070 cards, so take the newest driver available.

## 2. HIP runtime

The driver installs the HIP runtime (`amdhip64_6.dll` and/or `amdhip64_7.dll`) into `System32`.
spinelminer looks for it in this order, always by full path, newest runtime first:

1. `amdhip64_7.dll` in `%HIP_PATH%\bin` (set by the HIP SDK installer);
2. `amdhip64_7.dll` in `%SystemRoot%\System32` (usually `C:\Windows\System32`);
3. `amdhip64_6.dll` in `%HIP_PATH%\bin`;
4. `amdhip64_6.dll` in `%SystemRoot%\System32`.

A copy of the HIP runtime, or of a DLL it depends on, in the current folder or on `PATH` is not
used. Keep `spinelminer.exe` in a folder only you can write to: Windows also searches the exe's
own folder for some DLLs.

To use one specific file, set `SPINEL_HIP_LIB`:

```
set SPINEL_HIP_LIB=C:\path\to\amdhip64_7.dll
```

Then only that file is tried. If no runtime is found the miner exits with code 2 and the message
`HIP runtime not found; install an AMD Adrenalin driver or the HIP SDK (ROCm 6.x or 7.x, amdhip64_*.dll) or set SPINEL_HIP_LIB to its path; tried: ...`.

## 3. Check

```
spinelminer.exe --list-devices
spinelminer.exe --benchmark --seconds 60
```

On the developer's rig the 60 s benchmark gave 137.1 TH/s on an RX 9070 XT and 70.0 TH/s on an
RX 9060 XT, about 2% below the same cards on Linux (a build and driver difference, not yet
explained). Other cards are expected to be within driver differences of their Linux figure.

## 4. Start mining

```
spinelminer.exe -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1
```

Ready-to-edit `.bat` start scripts (one per pool, with failover and automatic restart) are in the
release zip (and in the source tree under `examples/windows/`). **Edit `WALLET` before running one**: the scripts ship with
the developer's address filled in.

## Differences from Linux

- **`--solo`** only accepts `<host>:<port>` on Windows. Unix-socket addresses (`unix:<path>` or a
  path containing `/`) are rejected at startup. Run `pearl-gateway` with a TCP listener.
- **Pool mining** (`-o`) and **`--benchmark`** behave as on Linux .
- **GPU sensors come from AMD's ADLX library** (`amdadlx64.dll`, installed with the Adrenalin
  driver; no third-party tool is needed). The [stats API](api.md), `--temp-limit` and
  `--power-spike-limit` work wherever ADLX reports the value:
  - `temp` is the hot-spot (junction) temperature, or the edge temperature on GPUs without one
    (iGPUs); `temp_mem`, `power`, `fan_rpm`, `sclk` and `mclk` are reported when the GPU has them.
  - `fan_percent` is always `null` on Windows: ADLX reports fan speed in RPM only.
  - A value the GPU does not report is `null`. A GPU ADLX cannot see (for example one disabled in
    Device Manager) reads `null` and is looked up again once a minute.
  - Without ADLX (no AMD driver) the miner logs `GPU sensors unavailable` once, the API reports
    `null`, and `--temp-limit` cannot protect a GPU: it logs
    `GPUn: no temperature sensor reading; --temp-limit C cannot protect this GPU` once per GPU
    and never pauses. Likewise `--power-spike-limit` warns and cannot stop the run.
- The HiveOS and mmpOS packages are Linux only.

## Building from source

1. Install the **AMD HIP SDK for Windows** (6.4.2 or newer; it sets `HIP_PATH`) and the driver.
2. Install Rust with the MSVC toolchain (`rustup` plus the Visual Studio C++ Build Tools), or the
   GNU toolchain if you have mingw-w64.
3. `cargo build --release`

Cross-compiling from Linux is described in the developer document `docs/windows.md`.

The HIP device-properties offsets the miner uses were checked against a native Windows build.
