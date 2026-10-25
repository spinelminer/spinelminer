# mmpOS

SpinelMiner ships as an mmpOS **custom (external) miner** package.

> **Unverified against mmpOS.** No official mmpOS custom-miner specification was available when
> the package was written. The command line mmpOS passes, the stats field names and formats, the
> descriptor keys, the `Miner API connection failed` text and whether `jq` exists on mmpOS are
> **assumptions**, taken from community-style launchers. The package has been tested only against
> recorded API responses. Check it on a real rig before you rely on it, and please report what you
> find.

## What is in the package

- `mmp-external.conf`: the descriptor (`EXTERNAL_NAME=spinelminer`, `EXTERNAL_ALGO=pearlhash`,
  `EXTERNAL_API_PORT=4067`; field names *unverified*).
- `mmp-launch.sh`: starts the miner.
- `mmp-stats.sh`: prints stats for mmpOS.
- `spinelminer`: a launcher that runs `spinelminer.bin` against the bundled `lib/` directory (ROCm
  10.1 HIP runtime and its dependencies). The rig needs no ROCm install, only the amdgpu kernel
  driver.
- Licence files: `LICENSE` (when present), `THIRD-PARTY-LICENSES`, `THIRD-PARTY-LICENSES-RUST`.

`mmp-launch.sh` and `mmp-stats.sh` need `curl` and `jq` (*unverified* that mmpOS has `jq`).

## Miner profile fields

*Unverified:* mmpOS is believed to call the launcher as

```
./mmp-launch.sh --coin %coin% tcp|tls --pool %pool_server%:%pool_port% \
                --user %user% --password %password% --api-port %api_port% [extra args...]
```

`mmp-launch.sh` turns this into a miner command line:

| mmpOS input | Becomes |
|---|---|
| coin `PRL`, `PEARL` or empty | `-a pearlhash`. Any other coin prints a warning and adds no `-a`. |
| `tls` + `host:port` | `-o stratum+ssl://host:port` |
| `tcp` + `host:port` | `-o stratum+tcp://host:port` |
| pool that already has a scheme (`stratum+ssl://...`) | used as given |
| user | `-u <user>` (e.g. `<wallet>.<worker>`) |
| password | `-p <password>`, which the miner accepts and ignores (it sends pools no password) |
| api port (missing or 0 -> 4067) | `--api-port <port>` |
| extra arguments | passed on unchanged, after the above (see [Options](options.md)) |

`--no-color` is always added. A missing pool aborts the launch with
`mmp-launch: no --pool given; not starting`. Arguments are passed as an array, so passwords and
values with spaces are not split or globbed.

To preview the miner arguments without starting it:

```
MMP_ARGS_ONLY=1 ./mmp-launch.sh --coin PRL tls --pool de.pearl.herominers.com:1200 --user <wallet>.rig1
```

## Logs

The launcher does not add `--log-file`: the miner writes its log to stderr and its status lines to
stdout. *Unverified:* whether and where mmpOS stores that output. To also write a log
file, add `--log-file /path/to/spinelminer.log` to the extra arguments.

## Stats

`mmp-launch.sh` records the final API port and host in the files `api-port` and `api-host` next to
it (the last `--api-port` / `--api-host` wins, as in the miner). `mmp-stats.sh` reads
`http://<host>:<port>/api/v1/summary` (default `127.0.0.1:4067`; `0.0.0.0` is read as `127.0.0.1`)
and prints one JSON object (all field names and layouts *unverified*):

| Field | Content |
|---|---|
| `busid` | decimal PCI bus numbers, one per GPU |
| `hash` | per-GPU hashrate in kH/s |
| `units` | `khs` |
| `air` | `["<accepted>", "0", "<rejected>"]` (strings) |
| `shares` | `{accepted: [...], rejected: [...], invalid: [0, ...]}` per GPU |
| `temp`, `fan` | per GPU; `null` when a GPU has no readable sensor |
| `uptime` | seconds |
| `miner_name`, `miner_version` | `spinelminer` and the miner version |

If the API is unreachable or the reply is not a summary, it prints `Miner API connection failed`
and exits with 0.
