# Stats API

With `--api-port <PORT>` the miner serves live statistics as JSON over HTTP. It runs in pool (`-o`)
and `--solo` runs; with `--benchmark` the option is ignored with a warning.

```
spinelminer -o ... -u ... --api-port 4067
curl http://127.0.0.1:4067/api/v1/summary
```

At startup the log shows `stats API listening on http://127.0.0.1:4067/api/v1/summary`.

## Security

- The API binds to `127.0.0.1` by default, so only the rig itself can read it.
- `--api-host <IP>` binds another address, e.g. `--api-host 0.0.0.0` for all interfaces. The API has
  **no authentication** and shows your pool user (`wallet.worker`), so do this only on a trusted
  network. The miner logs a warning when the address is not a loopback address.
- The API never shows passwords or proxy credentials.
- A request whose `Host` header is a DNS name other than `localhost` gets `421` with
  `{"error":"host not allowed"}`. Allowed are `localhost`, loopback addresses and the `--api-host`
  address (any IP address when bound to `0.0.0.0` or `::`). This stops a web page from reaching
  the API through DNS rebinding. Use the IP address, not the rig's host name, in dashboards.
- `Access-Control-Allow-Origin: *` is sent only when `--api-host` is not a loopback address, so
  on the default `127.0.0.1` no web page can read the API from the browser.

## Endpoints

| Request | Response |
|---|---|
| `GET /api/v1/summary` | the full summary (below) |
| `GET /` | the same summary |
| `GET /api/v1/gpus` | only the `gpus` array |
| any other path | `404` with `{"error":"not found"}` |
| any method other than `GET` | `405` with `{"error":"method not allowed"}` |

Responses are JSON (`Content-Type: application/json`) with `Connection: close` (and
`Access-Control-Allow-Origin: *` off loopback, see above). A query string is ignored. The data is refreshed about once a second.

If the port cannot be bound, the miner exits with code 1 and
`cannot start the stats API on <addr>: <reason> (check --api-host / --api-port)`.

## Summary (pool mode)

```json
{
  "miner": "SpinelMiner",
  "version": "0.4.0",
  "algo": "pearlhash",
  "uptime": 12345,
  "mode": "pool",
  "pool": {
    "url": "stratum+ssl://de.pearl.herominers.com:1200",
    "user": "<wallet>.<worker>",
    "latency_ms": 26,
    "connected": true,
    "difficulty": 2097184.0
  },
  "total": {"hashrate": 208600000000000.0, "effective": 205100000000000.0, "accepted": 999, "rejected": 0, "stale": 8},
  "donation": {"percent": 1.0, "accepted": 9, "rejected": 0, "in_window": false},
  "gpus": [
    {
      "index": 0, "pci_bus": 3, "pci": "03:00.0", "name": "AMD Radeon RX 9070 XT", "gfx": "gfx1201",
      "status": "mining",
      "hashrate": 138200000000000.0, "accepted": 653, "rejected": 0, "stale": 6,
      "temp": 95.0, "temp_mem": 70.0, "fan_percent": 100.0, "fan_rpm": 3100,
      "power": 330.0, "sclk": 2479, "mclk": 1148,
      "pcie_h2d_mbps": 0.5, "pcie_d2h_mbps": 0.25
    }
  ]
}
```

### Top level

| Field | Type | Meaning |
|---|---|---|
| `miner` | string | always `SpinelMiner` |
| `version` | string | miner version |
| `algo` | string | always `pearlhash` |
| `uptime` | integer | seconds since the run started |
| `mode` | string | `pool` or `solo` |
| `pool` | object | pool mode only (below) |
| `gateway` | object | `--solo` only (below) |
| `total` | object | totals over all GPUs |
| `donation` | object | developer donation counters |
| `gpus` | array | one object per mining GPU |

### `pool`

| Field | Type | Meaning |
|---|---|---|
| `url` | string | current pool URL (changes on failover) |
| `user` | string | the login as sent (`-u`, plus `.worker` with `--worker`) |
| `latency_ms` | integer or null | average share round trip; `null` before the first reply |
| `connected` | bool | session connected |
| `difficulty` | number or null | current share difficulty; `null` before the first job |

### `gateway` (`--solo` only)

| Field | Type | Meaning |
|---|---|---|
| `addr` | string | gateway address (`unix:<path>` for a socket, else `host:port`) |
| `connected` | bool | the last request to the gateway succeeded |

### `total`

| Field | Type | Meaning |
|---|---|---|
| `hashrate` | number | sum of the GPUs' hashrates, H/s |
| `effective` | number | H/s implied by accepted shares x difficulty over time (as pool dashboards compute it). In `--solo` mode it equals `hashrate`. |
| `accepted`, `rejected`, `stale` | integer | sums of the GPU counters |

### `donation`

| Field | Type | Meaning |
|---|---|---|
| `percent` | number | `--donate` value |
| `accepted`, `rejected` | integer | developer shares during donation windows |
| `in_window` | bool | a donation window is mining now |

### `gpus[]`

| Field | Type | Meaning |
|---|---|---|
| `index` | integer | device index (as in `--list-devices` and `-d`) |
| `pci_bus` | integer | PCI bus number, decimal (`0x0b` -> 11) |
| `pci` | string | PCI address `bb:dd.f` |
| `name`, `gfx` | string | device name and architecture |
| `status` | string | `mining`, `paused-temp` (paused by `--temp-limit`), `disabled` or `idle` |
| `hashrate` | number | H/s over the last 20 s; 0 when not mining |
| `accepted`, `rejected`, `stale` | integer | pool verdicts for this GPU. In `--solo` mode `accepted` counts block candidates sent. |
| `temp` | number or null | junction temperature, °C (edge if no junction sensor) |
| `temp_mem` | number or null | memory temperature, °C |
| `fan_percent` | number or null | fan, % |
| `fan_rpm` | integer or null | fan, RPM |
| `power` | number or null | power, W |
| `sclk`, `mclk` | integer or null | core and memory clock, MHz |
| `pcie_h2d_mbps`, `pcie_d2h_mbps` | number | measured host-to-device / device-to-host copy traffic of this GPU, MB/s (10^6 bytes), over the last 20 s. See [PCIe risers](pcie-risers.md). |

Hashrates are H/s as JSON numbers, where one hash is one int8 multiply-accumulate (the unit the
miner reports everywhere). Divide by 1e12 for the TH/s shown in the status lines.

Sensor fields are read from Linux sysfs, or from AMD's ADLX library on Windows (installed with the
Adrenalin driver). A sensor the host cannot read is `null`, never 0. On Windows `temp` is the hot
spot (edge on GPUs without one), `fan_percent` is always `null` (ADLX reports RPM only), and all
sensor fields are `null` when ADLX is unavailable.

## Summary (solo mode)

Same as above, with `"mode": "solo"` and `pool` replaced by:

```json
"gateway": {"addr": "unix:/tmp/pearlgw.sock", "connected": true}
```

## Who uses it

The HiveOS (`h-stats.sh`) and mmpOS (`mmp-stats.sh`) packages read `/api/v1/summary`; see
[HiveOS](hiveos.md#stats-in-the-hiveos-dashboard) and [mmpOS](mmpos.md#stats).
