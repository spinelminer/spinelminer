# PCIe risers (x1 slots)

Will SpinelMiner run on a PCIe x1 riser? Short answer: yes, bandwidth is not the limit. The
miner moves only a few bytes per slice over the bus; the operands are generated on the GPU.

## Measure it yourself

Every host-to-device (H2D) and device-to-host (D2H) copy the miner issues is counted per GPU
(`hipMemcpy` goes through one wrapper, `src/hip/mod.rs`, `DeviceBuffer::upload` / `download`).

- Status lines: `GPU0 ...: 139.0 TH/s, ..., pcie 0.0003/0.0100 MB/s` (H2D/D2H, every status interval).
- Stats API: `pcie_h2d_mbps`, `pcie_d2h_mbps` per GPU in `/api/v1/summary` and `/api/v1/gpus`
  (20 s window). See [api.md](api.md).
- Kernel report: `summary.pcie_h2d_mbps` / `pcie_d2h_mbps` (mean over the mining window).

1 MB = 10^6 bytes. Kernel launch arguments (tens of bytes per launch), the one-off code-object
load at start and driver-internal command packets are not counted; they are far smaller still.

## Traffic formula (from the code)

Constants: `SLICE_BLOCKS` = 16384 blocks per slice, block = `BLOCK_M x BLOCK_N` = 128 x 256 MACs
per k step, `size_of::<HitRecord>()` = 48 bytes, tuned `k` = 4096 or 8192.

- MACs per slice = 16384 x 128 x 256 x k = 2^29 x k (k 4096: 2.20e12; k 8192: 4.40e12).
- Slices per second = hashrate / MACs per slice.
- **Per slice** (`HitBuffer::drain_counted`, `src/gpu/mine.rs`): D2H 4 B (hit counter) + 48 B per
  stored hit; H2D 4 B (counter reset to zero).
- **Per share/hit**: 48 B D2H. Hits arrive at one per ~2^46 MACs at pool difficulty
  ~2^21 (about 2 per second at 139 TH/s), so about 100 B/s.
- **Per work unit** (`Miner::build`, `ZeroRoot::root`, `src/gpu/merkle.rs`): D2H of the last
  Merkle level of the zero-matrix root for A, at most 256 chaining values x 32 B = 8 KiB (B's is
  cached per job). A unit is a 131072 x 131072 grid, 32 slices.
- **Operands, seeds, noise, headers**: no copies. Operands and noise are generated on the GPU;
  seeds and the header go in as kernel arguments. Proof building (`src/proof.rs`) is host-only.

Total per GPU:

    H2D  = 4 B x slices/s
    D2H  = 4 B x slices/s + 48 B x hits/s + (<= 8 KiB) x units/s

## Expected MB/s (computed)

| Rate | k | slices/s | units/s | H2D MB/s | D2H MB/s (upper bound) |
|---|---|---|---|---|---|
| 139 TH/s (RX 9070 XT) | 4096 | 63.2 | 1.97 | 0.00025 | 0.0006 + roots, <= 0.017 |
| 139 TH/s | 8192 | 31.6 | 0.99 | 0.00013 | 0.0004 + roots, <= 0.009 |
| 70 TH/s (RX 9060 XT) | 4096 | 31.8 | 0.99 | 0.00013 | 0.0003 + roots, <= 0.009 |
| 70 TH/s | 8192 | 15.9 | 0.50 | 0.00006 | 0.0002 + roots, <= 0.005 |
| measured, RX 9070 XT (k 8192, 2026-10-25) | 8192 | | | 0.0001 | 0.0009 |
| measured, RX 9060 XT (k 4096, 2026-10-25) | 4096 | | | 0.0001 | 0.0006 |

The upper bound assumes the full 8 KiB root per unit; real roots are smaller (the root of the
2^24-row A commitment is a few chaining values). The "+ roots" term is therefore below the bound.

## x1 link bandwidth (per direction)

| PCIe x1 | usable |
|---|---|
| gen1 | ~250 MB/s |
| gen2 | ~500 MB/s |
| gen3 | ~985 MB/s |
| gen4 | ~1970 MB/s |

## Conclusion

Peak traffic is about 0.02 MB/s per GPU, roughly 10,000 times below even a gen1 x1 link
(250 MB/s), so bandwidth does not limit hashrate on a riser. The traffic is also a handful of
tiny synchronous copies per second (about 63 slice drains/s at most), so latency (about 1 ms on a
USB-style x1 riser) costs well under 1% of wall time and the GPU keeps queued slices running
meanwhile. Remaining x1 concerns are not bandwidth: the one-off start-up load of the kernel
module, and riser power and signal quality (watch for watchdog resets). The measured rows (status-line counters on the owner's rig, first minute of
pool mining on HeroMiners) match the formula.

## Weak CPUs

The CPU builds each share's proof. Measured on a Ryzen 7 9800X3D rig (RX 9070 XT + RX 9060 XT,
`--benchmark --seconds 60`, 2026-10-25), pinning the miner to fewer cores with `taskset`:

| CPU cores | RX 9070 XT | RX 9060 XT | Benchmark shares dropped |
|---|---|---|---|
| all 16 threads | 134.4 TH/s | 70.4 TH/s | 0 of 22 |
| 2 (`taskset -c 0,1`) | 133.4 TH/s | 70.0 TH/s | 1 of 21 |
| 1 (`taskset -c 0`) | 133.3 TH/s | 70.0 TH/s | 6 of 30 |

Hashrate does not depend on the CPU (under 1 % difference). One core proves about 0.4 shares/s.
The benchmark's share target is far easier than a pool's: on HeroMiners this rig found about 0.023
shares/s (817 shares in 10 h), so even one core of a slow CPU has a wide margin for several GPUs.
Only very low pool difficulty on many GPUs could exceed it; the miner then logs
"dropped … all proof threads busy". Give it 2+ cores on big rigs.
