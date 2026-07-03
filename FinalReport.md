# JMeter Load Test Report: Multithreaded WebServer Comparison

*All figures in this report are computed directly from the raw `.jtl` result files
(pandas, verified against JMeter's own HTML report output) — no numbers were
hand-transcribed.*

## What was tested

Three Java TCP socket server implementations from the same project, each running
in its own Docker container (1 CPU, 512MB memory cap):

- **SingleThreaded** — one thread spawned per accepted connection, no pooling
- **Multithreaded** — same per-connection thread model as above (see note below)
- **ThreadPool** — fixed-size thread pool serving connections

Each server responds with a single line of text and closes the connection
immediately — there is no HTTP layer, so JMeter's TCP Sampler was used with
`eolByte=10` (newline) configured via a `user.properties` file, `reUseConnection=false`,
`closeConnection=true`, and a 5000ms timeout.

## Test configuration

- Apache JMeter 5.5 (client), Java 1.8.0_275, running in the `justb4/jmeter`
  Docker image, Linux/WSL2 host
- Servers built on `eclipse-temurin:17-jdk`
- Two load tiers: **100 threads** and **1000 threads**, 1 loop each (one request
  per virtual user — this is a concurrency-ramp test, not a sustained-duration
  soak test)
- Ramp-up: **2 seconds** for the 100-thread tier, **15 seconds** for the
  1000-thread tier

**Methodology caveat:** because ramp-up differs between tiers, the two tiers
aren't a clean apples-to-apples comparison — the 1000-thread runs spread their
requests over ~15s instead of ~2s, so peak *simultaneous* connections were
likely well below 1000 at any single instant, and throughput between tiers
reflects this pacing difference as much as server capacity. Treat the 100 vs
1000 comparison as directional, not precise.

## Verified results

| Server | Load | Samples | Errors | Mean (ms) | Median (ms) | Min (ms) | Max (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Throughput (req/s) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| SingleThreaded | 100  | 100  | 0 | 32.72 | 5.0 | 1 | 198 | 99.0  | 103.25 | 198.00 | 52.80 |
| SingleThreaded | 1000 | 1000 | 0 | 35.39 | 2.0 | 0 | 702 | 100.0 | 193.15 | 384.06 | 67.23 |
| Multithreaded  | 100  | 100  | 0 | 9.69  | 3.0 | 1 | 74  | 37.1  | 54.30  | 74.00  | 52.94 |
| Multithreaded  | 1000 | 1000 | 0 | 14.61 | 2.0 | 1 | 303 | 55.0  | 98.00  | 187.09 | 67.18 |
| ThreadPool     | 100  | 100  | 0 | 30.72 | 2.0 | 1 | 292 | 116.3 | 204.05 | 209.83 | 53.28 |
| ThreadPool     | 1000 | 1000 | 0 | 7.14  | 2.0 | 0 | 177 | 10.0  | 53.10  | 98.00  | 67.03 |

**Zero errors across all 3,300 requests, all six runs** — confirmed against the
`success` field in the raw data (parsed as genuine booleans, not a string-comparison
artifact).

## What the data actually shows

**No single architecture wins at both load levels.** This is the main thing the
earlier draft of this report got wrong by only spotlighting ThreadPool's 1000-thread
numbers:

- At **100 threads**, Multithreaded had the best mean (9.69ms) and best max (74ms)
  latency of the three — ThreadPool was actually the *worst* on max latency at this
  tier (292ms), likely pool-saturation or startup effects.
- At **1000 threads**, ThreadPool pulled ahead on every metric (7.14ms mean, 177ms
  max) — Multithreaded's max latency nearly quadrupled (74ms → 303ms) between tiers.
- **SingleThreaded was the weakest performer at both tiers** on tail latency
  specifically — max latency roughly doubled from 198ms to 702ms between tiers, the
  largest degradation of the three servers. That said, it never actually errored or
  crashed in these runs, even at 1000 threads — worth noting since a raw
  one-thread-per-connection design is generally expected to hit resource limits
  under load; the 15-second ramp-up on the 1000-thread tier likely kept peak
  simultaneous thread count from ever spiking hard enough to hit your container's
  512MB cap. A future test with `rampup=0` or `1` (a true instantaneous burst) would
  be a more honest stress test of SingleThreaded's actual breaking point.

**Throughput is not a strong differentiator here.** All three servers land within
~52–67 req/s regardless of architecture, and the jump between tiers (52-53 → 67
req/s) is more consistent with the ramp-up-duration difference than with any
architecture actually processing requests faster — this metric doesn't cleanly
separate the three implementations in this test design.

**Tail latency (max, p95, p99) is where the architectures actually diverge**, not
mean/median — all three had very similar median latency (2–5ms) at both tiers.
If you're arguing for one design over another in your writeup, tail latency under
load is the metric that supports it, not throughput or median response time.

## Recommendation for your writeup

Rather than declaring one architecture an outright "winner," the more defensible
claim from this data is:

- ThreadPool showed the most stable tail latency as concurrency scaled up (100→1000),
  which is the theoretically expected behavior for a bounded worker pool
- Multithreaded performed best at lower concurrency but showed the sharpest tail-latency
  growth of the three as load increased — consistent with unbounded thread creation
  cost compounding under load
- SingleThreaded consistently had the worst tail latency at both tiers, though a more
  aggressive (near-zero ramp-up) test would be needed to actually observe a hard
  failure point, since it didn't error out here

## Suggested next step

If you want a genuinely conclusive "winner" claim, the current data doesn't quite
support one — a same-ramp-up comparison (e.g. rerun both tiers at `rampup=2`) plus
one true burst test (`rampup=0` or `1` at high thread count) would isolate
architecture effects from pacing effects and likely surface SingleThreaded's actual
failure point.
