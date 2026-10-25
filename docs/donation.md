# Donation (dev fee)

SpinelMiner donates a share of its mining time to the developer. This is on by default, announced at
every start and logged at every donation window. It applies to pool (`-o`) and `--solo` runs.
`--benchmark` and `--list-devices` never donate.

The startup line reads:

```
Donation: 1.0% of mining time to the developer (dev pools: HeroMiners, Kryptex; see README)
```

## Rate

`--donate <PCT>`: default **1.0**, minimum **1.0**, maximum **50.0** (percent of mining time).
You can raise it to give more; you cannot lower it below 1.0. A value outside the range exits
with code 1:
`--donate must be a number from 1.0 to 50.0 (percent of mining time), got ...`.

## Schedule

- The first window starts **3 to 5 minutes** after the run's first job.
- After that there is **one window per 100 minutes of mining**.
- A window lasts `PCT / 100 x 6000` seconds: **60 s at 1 %** (120 s at 2 %, 3000 s at 50 %).
- Only mining time counts: time when no pool or gateway has a job does not.
- Nothing is stored between runs.

## What happens in a window

- The GPUs mine for the developer on a developer pool.
- Your own pool session stays connected. New jobs from your pool during the window are applied
  when the window ends.
- The log shows `donation window: <s> s on <pool> <region> (<rtt> ms)` at the start and
  `donation window done: accepted A, rejected R` at the end.
- Status lines gain `dev A/R` once the developer has shares. The [stats API](api.md) shows the
  same counters under `donation`.

## Developer pools and regions

HeroMiners first, then Kryptex:

| Region | Pool | URL |
|---|---|---|
| `de` | HeroMiners | `stratum+ssl://de.pearl.herominers.com:1200` |
| `fi` | HeroMiners | `stratum+ssl://fi.pearl.herominers.com:1200` |
| `ca` | HeroMiners | `stratum+ssl://ca.pearl.herominers.com:1200` |
| `us` | HeroMiners | `stratum+ssl://us.pearl.herominers.com:1200` |
| `us2` | HeroMiners | `stratum+ssl://us2.pearl.herominers.com:1200` |
| `br` | HeroMiners | `stratum+ssl://br.pearl.herominers.com:1200` |
| `hk` | HeroMiners | `stratum+ssl://hk.pearl.herominers.com:1200` |
| `sg` | HeroMiners | `stratum+ssl://sg.pearl.herominers.com:1200` |
| `ru` | HeroMiners | `stratum+ssl://ru.pearl.herominers.com:1200` |
| `kr` | HeroMiners | `stratum+ssl://kr.pearl.herominers.com:1200` |
| `au` | HeroMiners | `stratum+ssl://au.pearl.herominers.com:1200` |
| `tr` | HeroMiners | `stratum+ssl://tr.pearl.herominers.com:1200` |
| `global` | Kryptex | `stratum+ssl://prl.kryptex.network:8048` |
| `eu` | Kryptex | `stratum+tcp://prl-eu.kryptex.network:7048` |

- `--donate-region list` prints this table (one line per region: `<name>  <pool>  <url>`) and
  exits. It needs no pool and no GPU.
- `--donate-region auto` (default) picks the fastest region by connect time. A background probe
  measures it and never affects your mining. Pools keep their order (HeroMiners first); within a
  pool the fastest region is tried first.
- `--donate-region <NAME>` tries that region first; the others remain as fallbacks.
- An unknown name exits with code 1 and lists the valid names.
- With `--proxy`, developer-pool connections and the probe also go through the proxy.

## Developer address

`prl1plpy7056ym0cyhk2agqdvq9kmkjamk745jfqv69hktd9djj0cu3pqg2csjt`, login
`<address>.devfee`.

## Fallback

If every developer pool fails, the miner logs in to **your own pool** a second time as the
developer wallet (`<address>.devfee`) and donates there. A route only counts once it sends a job
within 10 s. If that fails too, the window is retried (300 s apart).

In `--solo` mode there is no pool login, so there is no such fallback.

## Stop rule (exit code 4)

If **3 donation windows in a row, over at least 10 minutes,** deliver nothing (neither a developer
pool nor the fallback works) **while your own pool is working**, the miner stops with **exit code
4** and prints:

```
Donation could not be delivered in 3 attempts over 10+ minutes while your pool was reachable. Dev pool hosts appear blocked: <dev hosts>. Unblock them (or allow your pool to accept the developer wallet); see README "Donation".
```

"Working" means:

- pool mode: connected throughout, and at least one of your shares accepted since the first
  failed window;
- `--solo`: every gateway poll succeeded, and at least one job on a new block arrived since the
  first failed window.

A broken network or an outage of your own pool **never** stops mining. Any successful window
resets the count.

To fix an exit code 4, allow the developer pool hosts listed in the message through your firewall,
DNS filter or proxy (or let your pool accept the developer wallet).
