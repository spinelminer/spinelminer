# Pools

Hosts, ports and fees on this page come only from the miner's developer-pool table
(`src/donate/regions.rs`) and the pool notes recorded during testing
(`tests/fixtures/pools/*/NOTES.md`, 2026-10-20). Pools change their servers; if a host here stops
working, check the pool's own website. Where no fee is recorded, check the pool's site.

## How pool URLs work

- `stratum+ssl://host:port` (or `stratum+tls://`): TLS with certificate verification.
- `stratum+tcp://host:port`: plain TCP.
- A bare `host:port` means TLS.
- Repeat `-o` for failover. The first URL is the primary. After 3 failures in a row the miner
  moves to the next URL, and while on a backup it tries the primary again every 5 minutes.
- `-u` is sent to the pool as given. `--worker NAME` appends `.NAME`.
- `-p` is accepted but never sent: every supported pool uses the `kryptex` protocol dialect, whose
  login has no password field. Password tricks such as AlphaPool's `x;d=N` fixed difficulty do not
  apply.
- The protocol dialect is chosen from the host name. Only the hosts listed under "Supported pools"
  are known. For any other host, add `--pool-dialect kryptex` (the only dialect); the miner exits
  with `no dialect for host ...` otherwise.

None of the supported pools needs an account: your wallet address is the login.

## Summary

| Pool | Status (2026-10-20) | TLS | Fee (from notes) |
|---|---|---|---|
| [HeroMiners](#herominers) | works (1 h live run clean) | yes, verified | 0 % (PROP) |
| [Kryptex](#kryptex) | works (1 h live run clean) | port 8048 | not recorded |
| [PearlFortune](#pearlfortune) | works (1 h live run clean) | no (self-signed cert) | not recorded |
| [LuckyPool](#luckypool) | works (1 h live run clean) | no (self-signed cert) | not recorded |
| [AlphaPool](#alphapool) | works (1 h live run clean) | no | 1 % (PROP) |
| [PearlHash](#pearlhash) | does not work | - | - |
| [GrandPool](#grandpool) | does not work | - | - |
| [F2Pool](#f2pool) | untested, needs an account | - | - |
| [Akoya](#akoya) | offline | - | - |
| [MinePRL](#mineprl) | offline | - | - |

"Works" means a one-hour live run on 2026-10-20 had no errors; longer dashboard checks were still
pending at that time.

## Supported pools

### HeroMiners

- Host: `<region>.pearl.herominers.com`, port **1200**. TLS and plain TCP on the same port. The TLS
  certificate is from Let's Encrypt and verifies.
- Regions confirmed by the miner's developer-pool table (each resolves and serves a valid TLS
  certificate on 1200): `de`, `fi`, `ca`, `us`, `us2`, `br`, `hk`, `sg`, `ru`, `kr`, `au`, `tr`.
  `de` is the pool's main host. The pool's page also names `fr`, `es`, `us3` and `jp`; those were
  not checked by the miner's table.
- Restricted networks: the pool offers `<region>.pearl.gfwroute.com:1200` (TCP only, per the pool's
  page).
- Login: `<wallet>.<worker>`.
- Fee: the pool software reports fee 0, PROP reward scheme.

```
spinelminer -o stratum+ssl://de.pearl.herominers.com:1200 -o stratum+ssl://fi.pearl.herominers.com:1200 -u <wallet> --worker rig1
```

### Kryptex

| Region | URL | Transport |
|---|---|---|
| global | `stratum+ssl://prl.kryptex.network:8048` | TLS (certificate verifies) |
| eu | `stratum+tcp://prl-eu.kryptex.network:7048` | plain TCP (no TLS on this port) |

- Login: `<wallet>.<worker>` or `<wallet>/<worker>` (Kryptex accepts both).
- The two ports use different start difficulties.
- Fee: not recorded in the notes; see the pool's site.

```
spinelminer -o stratum+ssl://prl.kryptex.network:8048 -o stratum+tcp://prl-eu.kryptex.network:7048 -u <wallet> --worker rig1
```

### PearlFortune

- Host: `global.pearlfortune.org`, port **8888**, plain TCP. This is the pool's only endpoint for
  third-party miners.
- Port 8888 also answers TLS, but with a self-signed certificate, so `stratum+ssl://` fails
  certificate verification. Use `stratum+tcp://`.
- Login: the bare wallet, or `<wallet>/<worker>`. No password.
- Fee: not recorded in the notes; see the pool's site.

```
spinelminer -o stratum+tcp://global.pearlfortune.org:8888 -u <wallet>/rig1
```

### LuckyPool

- Host: `pearl-<region>.luckypool.io`. Regions: `eu1`, `eu2`, `pl`, `tr`, `ru`, `us-east`,
  `us-west`, `us-central`, `us-ord`, `br`, `ca1`, `ca2`, `sg1`, `sg2`, `id`, `hk`, `in`, `jp`, `au`.
  The pool's default is `pearl-eu2.luckypool.io:3360`.
- Ports (all vardiff): **3360** (start difficulty 2M, "< 500 TH/s"), **3361** (4M), **3362** (8M).
- TLS uses a self-signed certificate, so use plain TCP (`stratum+tcp://`).
- Restricted networks: replace `luckypool.io` with `lproute.com`, same port.
- Login: `<wallet>.<worker>`.
- Fee: not recorded in the notes; see the pool's site.

```
spinelminer -o stratum+tcp://pearl-eu2.luckypool.io:3360 -u <wallet> --worker rig1
```

### AlphaPool

- Host: `<region>.alphapool.tech`, port **5571**, plain TCP (no TLS).
- Regions: `us1` (North America East), `us2` (North America West), `eu1` (Europe), `ru1`
  (Russia/Eurasia), `sg1` (Singapore).
- Only port 5571 works with spinelminer. Ports 5566 and 5573 are for the pool's own miner and use a
  different protocol.
- Login: `<wallet>.<worker>`. The pool's `x;d=N` password option for fixed difficulty does not
  apply, because spinelminer sends no password; the pool uses variable difficulty.
- Fee: 1 %, PROP.

```
spinelminer -o stratum+tcp://eu1.alphapool.tech:5571 -u <wallet> --worker rig1
```

## Pools that do not work with spinelminer

These are listed so you do not have to find out the hard way. No dialect is registered for them.

### PearlHash

The pool closes the connection right after the login that spinelminer (and KRig / SRBMiner) send; it
speaks a different protocol. Endpoints from its site: `pool.pearlhash.xyz`,
`global.pearlhash.net`, `global2.pearlhash.net`, `hk.pearlhash.net`, ports 9000 (TCP) / 9443 (TLS).

### GrandPool

The pool only accepts logins from miners on its allowlist and answers spinelminer with error 24
"miner agent not allowed". Endpoints: `fr.grandpool.io`, `kz.grandpool.io`, `us.grandpool.io`,
ports 8070 (TCP) / 8071 (TLS). Fee from its page: PROP 0 %, SOLO 1 %.

### F2Pool

Needs an F2Pool account: the login is `accountname.workername`, not a wallet address. Endpoint
`pearl.f2pool.com:5500` (plain TCP). Wallet logins are closed without a reply. Untested with an
account.

### Akoya

Offline on 2026-10-20 (no DNS for `pool.akoyapool.com` / `pool-v2.akoyapool.com`). Its documented
protocol is gRPC for its own miner, not Stratum.

### MinePRL

Offline on 2026-10-20 (the site says so; `pool.mineprl.com` does not resolve). It never published a
Stratum endpoint and used its own H100/H200 worker. Fee listed elsewhere: 4.4 % PPLNS.

## Solo mining

Instead of a pool you can mine against your own node: run `pearld` and
[pearl-gateway](https://github.com/pearl-research-labs/pearl), then

```
spinelminer --solo /tmp/pearlgw.sock
spinelminer --solo 127.0.0.1:8337
```

The address is a Unix socket path, `unix:<path>`, or `host:port` (only `host:port` on Windows).
Blocks pay the gateway's `PEARLD_MINING_ADDRESS`; spinelminer has no address of its own. The miner
cannot see whether a block was accepted; check your node or wallet.
