# Install on Linux

## Supported systems

| OS | Status | Notes |
|---|---|---|
| Ubuntu 26.04 LTS | Supported, tested | kernel 7.0 works out of the box |
| Ubuntu 24.04 LTS | Supported | RDNA4 needs the HWE kernel (6.14+) or AMD amdgpu-dkms; older GPUs work on the stock kernel |
| Ubuntu 22.04 LTS | Supported with AMD's driver | RDNA4 needs amdgpu-dkms (stock 6.8 kernel lacks it) |
| HiveOS, mmpOS | Supported | own kernels/drivers; see [HiveOS](hiveos.md) and [mmpOS](mmpos.md) |
| Debian 12/13, Fedora, Arch, other glibc distros | Should work, untested | glibc >= 2.28, amdgpu with KFD |
| Ubuntu 20.04 and older | Not supported | |

Release binaries are built against glibc 2.28, so any x86-64 distribution with glibc 2.28 or newer
can load them.

## What the host needs

Whichever tarball you use:

1. The **amdgpu kernel driver** with `/dev/kfd` (the GPU compute device). For RX 9000 cards
   (RDNA4) this needs **kernel 6.14 or newer**, or AMD's **amdgpu-dkms** driver package on an
   older kernel.
2. amdgpu firmware for your GPU (RDNA4 firmware for RX 9000 cards).
3. Your user in the **`render` and `video` groups**:

   ```
   sudo usermod -aG render,video $USER
   ```

   Log out and back in (or reboot) for the new groups to take effect. Check with `groups`.

The miner checks the driver at startup and stops with a clear message if `/dev/kfd` is missing or
not accessible; see [Troubleshooting](troubleshooting.md#driver-and-devkfd-messages).

## Choose a tarball

Two Linux tarballs are published:

| Tarball | Contents | Needs on the host |
|---|---|---|
| `spinelminer-<ver>-linux.tar.gz` (slim) | the miner binary, docs, start scripts | ROCm installed: `libamdhip64.so.7` or `.so.6` (ROCm 6.x or 7.x HIP runtime) |
| `spinelminer-<ver>-linux-bundled.tar.gz` (bundled) | the same plus `lib/` with 15 ROCm runtime libraries (about 270 MB unpacked) | no ROCm install; only the kernel driver and permissions above |

In the **bundled** tarball, start the miner with the `spinelminer` launcher. It sets `SPINEL_HIP_LIB`
and `LD_LIBRARY_PATH` to the `lib/` directory next to it and runs the real binary,
`spinelminer.bin`. Do not run `spinelminer.bin` directly unless you set those yourself.

In the **slim** tarball, `spinelminer` is the miner binary itself. To use a specific HIP runtime,
set `SPINEL_HIP_LIB` to the full path of `libamdhip64.so`.

If in doubt, take the bundled tarball: it avoids a ROCm install.

## Install and check

```
tar xzf spinelminer-<ver>-linux-bundled.tar.gz
cd spinelminer-<ver>
./spinelminer --list-devices
```

`--list-devices` prints one line per GPU: index, PCI address, name, gfx architecture and
`supported`, `unsupported` or `supported (integrated, not used by default)`. For example:

```
[0] 0000:03:00.0 AMD Radeon RX 9070 XT gfx1201 supported
```

Then run a short benchmark, which mines a synthetic job and checks every share on the CPU:

```
./spinelminer --benchmark --seconds 60
```

## Start mining

```
./spinelminer -o stratum+ssl://de.pearl.herominers.com:1200 -u <wallet> --worker rig1 --api-port 4067
```

See [Pools](pools.md) for other pools and [Options](options.md) for every option.

The tarballs also contain ready-to-edit start scripts, one per pool (`start_herominers.sh`,
`start_kryptex.sh`, and so on, described in `START-SCRIPTS.md`). Each one restarts the miner 10
seconds after it exits. **Edit `WALLET` before you run one**: the scripts ship with the developer's
address filled in, so anything mined with an unedited script goes to the developer.

## Building from source

You need Rust stable 1.87 or newer and ROCm with `hipcc` on `PATH` (or `$HIPCC` pointing at it),
because the GPU kernels are compiled at build time:

```
cargo build --release
```

The binary is `target/release/spinelminer`.
