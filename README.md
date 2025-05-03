# barrage

[![Go version](https://img.shields.io/github/go-mod/go-version/codetesla51/barrage)](https://github.com/codetesla51/barrage)
[![CI](https://github.com/codetesla51/barrage/actions/workflows/build.yml/badge.svg)](https://github.com/codetesla51/barrage/actions/workflows/build.yml)
[![Release](https://img.shields.io/github/v/release/codetesla51/barrage)](https://github.com/codetesla51/barrage/releases)

Barrage is a load testing tool built to answer one question: **when an
application slows down, is the cause the application itself, or the database
and cache underneath it?**

It drives concurrent HTTP, database, and Redis load on a single clock, records
every layer's latencies into the same time buckets, and compares them bucket by
bucket. Instead of one latency curve that hides where the time went, you get a
correlated view of the API, the database, and the cache — and a report that
flags exactly which bucket a storage layer spiked in, and whether the
application was affected or not.

## Contents

- [Quickstart](#quickstart)
- [Commands](#commands)
  - [`barrage run`](#barrage-run)
  - [`barrage compare`](#barrage-compare)
  - [Other commands](#other-commands)
  - [Capacity sweep](#capacity-sweep)
- [The report](#the-report)
- [Concepts](#concepts)
  - [Spike correlation](#spike-correlation)
  - [Scenarios vs plain HTTP](#scenarios-vs-plain-http)
  - [Rate, concurrency, and ramp](#rate-concurrency-and-ramp)
  - [Live progress](#live-progress)
- [Configuration](#configuration)
- [Chaos testing (Toxiproxy)](#chaos-testing-toxiproxy)
- [Demo stack](#demo-stack)
  - [Profiles](#profiles)
  - [Case studies: the demo-stack progression](#case-studies-the-demo-stack-progression)
  - [Separated-box runs (cloudflared tunnel)](#separated-box-runs-cloudflared-tunnel)
- [When to use Barrage (and when not)](#when-to-use-barrage-and-when-not)
- [Development](#development)
- [Agents](#agents)

## Quickstart

```sh
curl -fsSL https://raw.githubusercontent.com/codetesla51/barrage/main/install.sh | bash
```

That grabs a prebuilt binary from GitHub releases — no Go toolchain needed.
Pin a version or change the target dir with
`bash -s -- --version v0.7.2 --dir ~/.local/bin`, or build from source with
`--from-source` (requires Go 1.25 or later). See `./install.sh --help` for all
flags. Or build from source directly:

```sh
git clone https://github.com/codetesla51/barrage && cd barrage
go build -o barrage ./cmd/barrage
# fallback without a checkout:
# go install github.com/codetesla51/barrage/cmd/barrage@latest
```

The DB runner supports **Postgres**, **MySQL**, and **SQLite** out of the box;
because it sits on `database/sql`, any other driver can be linked in by adding
a blank import and registering its name. HTTP-only runs require no backing
services.

Then point `config.yaml` at your targets (see [Configuration](#configuration))
and run:

```sh
barrage run                      # runs config.yaml, writes report.html
barrage run -o                   # ...and opens the report in your browser
barrage run --no-report --json results.json   # for CI, no browser needed
barrage compare --baseline base.json --current new.json   # diff two runs
```

The report is self-contained: Chart.js loads from a CDN, but all run data is
embedded in the page.

## Commands

### `barrage run`

A 15s run against local targets:

```
$ barrage run -c config.yaml

barrage v0.7.2
duration 15s · bucket 1s · concurrency 10 · ramp 3s
rates    http 10/s · db 5/s · redis 20/s

  ⠋ 00:15/00:15
  http  ················•••>··········  135   10/s
  db    ········•>····················   67    5/s
  redis ·············›················  269   18/s

RUNNER  REQUESTS  SUCCESS  RATE    MEAN    P50    P95     P99      MAX
──────────────────────────────────────────────────────────────────────────
http    135       100.0%   9.5/s   927µs   509µs  2.5ms   4.4ms    6.6ms
db      67        100.0%   4.5/s   12.9ms  5.5ms  69.0ms  136.2ms  136.2ms
redis   269       100.0%   17.9/s  797µs   396µs  2.3ms   3.5ms    10.6ms
──────────────────────────────────────────────────────────────────────────

correlated spikes
TIME      RUNNER  HTTP_P99  STORAGE_P99  NOTE
────────────────────────────────────────────────
20:52:22  db      <100ms    136.2ms      db-only
────────────────────────────────────────────────
Report written to report.html
```

```
$ barrage run --help

Flags:
  -b, --bucket-width duration             override the bucket width from the config
      --capacity                          sweep concurrency (double, then fine fill) to find the break point
      --capacity-max-concurrency int      cap for the capacity sweep
      --capacity-step-duration duration   per-level burst time for the capacity sweep (default 10s)
      --concurrency int                   worker count for the db/redis pools and http attackers
  -c, --config string                     path to the config file (default "config.yaml")
      --db-threshold duration             DB spike threshold for correlation (default 100ms)
  -d, --duration duration                 override the run duration from the config
      --http-threshold duration           HTTP spike threshold for correlation (default 100ms)
      --json string                       also write a JSON summary of the run to this path
      --no-progress                       disable the live progress view (plain log lines instead)
      --no-report                         skip writing the HTML report
  -o, --open                              open the report in a browser after the run
      --ramp duration                     ramp the rate from 0 up to full over this duration
      --redis-threshold duration          Redis spike threshold for correlation (default 100ms)
      --report string                 path for the HTML report (default "report.html")
  -v, --verbose                       print per-bucket detail
```

Examples:

```sh
barrage run -c staging.yaml --duration 1m --ramp 10s --concurrency 50
barrage run --http-threshold 150ms --db-threshold 250ms --redis-threshold 80ms  # adjust spike thresholds
barrage run --no-report --json results.json                 # for CI pipelines
```

Every `--` flag overrides its config counterpart.

A scenario run looks like this: the `rates` line names each journey with its
step count and weight, and the runner column carries the scenario name.

```
$ barrage run -c examples/scenario-login.yaml

barrage v0.7.2
duration 10s · bucket 1s · concurrency 10 · ramp 0s
rates    login-flow 3 steps w=1

  ⠋ 00:10/00:10
  scen  ·············›················  32,989  3298/s

RUNNER      REQUESTS  SUCCESS  RATE      MEAN        P50         P95         P99          MAX
─────────────────────────────────────────────────────────────────────────────────────────────────────
login-flow  32989     100.0%   3298.9/s  3.030392ms  2.272002ms  8.125266ms  11.957243ms  35.747684ms
─────────────────────────────────────────────────────────────────────────────────────────────────────
Report written to report.html
```

### `barrage compare`

`barrage compare` diffs two runs produced by `barrage run --json`, so an earlier
baseline can be checked against a later run — the missing piece for CI gating
and regression checking across releases.

Runners present only in one side are labeled **NEW** (or counted as fixed) —
they never show as regressions just because the baseline didn't have them, so
renaming or adding scenarios mid-project doesn't produce false alarms. Spike
diffs match by ordinal position per runner rather than wall-clock timestamps,
since two runs never share a clock.

```
$ barrage compare --baseline base.json --current new.json --fail-on 100ms

comparing base.json -> new.json (fail-on 100ms)
RUNNER  BASELINE_P99  CURRENT_P99  CHANGE  VERDICT
─────────────────────────────────────────────────────
DB      80ms          100ms        +25%    ok
HTTP    30ms          70ms         +133%   REGRESSION
Redis   20ms          22ms         +10%    ok
─────────────────────────────────────────────────────
```

Flags:

```
      --baseline string      path to the baseline JSON report
      --current string       path to the current JSON report
      --fail-on duration     fail (exit non-zero) if a runner regresses above this latency budget (default 100ms)
  -o, --open                 open the report in a browser after comparing
      --report string        path for the HTML comparison report (default "compare.html", empty skips it)
```

How it works:

- **Per-runner diff.** Each runner's P99 is compared baseline→current with a
  percentage change. A runner is flagged **REGRESSION** when its current P99
  exceeds the `--fail-on` budget while its baseline was at or under it, so a
  runner that was already slow isn't re-flagged every run. Any regression makes
  `barrage compare` exit non-zero — the signal CI uses to gate a deployment.
- **Spike diff.** Each correlated spike in both runs is classified as **new**,
  **fixed**, **worsened**, **improved**, or **unchanged**, so you can see both
  newly-introduced storage bottlenecks and ones that were resolved.
- **Overlaid timeline.** Both runs' per-bucket P99 latencies are aligned onto
  one label axis in the HTML report (baseline dashed, current solid), so you can
  see *where* in the run latency drifted.
- **Exit code as CI gate.** Combined with `barrage run --no-report --json
  baseline.json` and `--fail-on`, you can make an unstaged regression fail a
  pipeline before it ships.

### Other commands

```sh
barrage version    # print the version (from the git tag; `go run` builds show the dev default)
barrage help run   # full flags for a subcommand; --help works on all of them
```

Shell completion (generated by cobra, nothing custom):

```sh
barrage completion bash > /etc/bash_completion.d/barrage
barrage completion zsh > "${fpath[1]}/_barrage"
```

### Capacity sweep

The capacity question, answered by experiment instead of guessing. A capacity
sweep raises concurrency level by level, running a short burst at each, until
latency breaks:

```yaml
concurrency: 5 # sweep start
capacity:
  max_concurrency: 160 # search cap
  step_duration: 10s # burst per level, not the full duration
```

```sh
barrage run -c examples/capacity-pg.yaml
barrage run -c config.yaml --capacity --capacity-max-concurrency 160 --capacity-step-duration 10s
```

How it works: coarse doubling finds the rough zone fast (5→10→20→40…),
then a fine linear fill pins it down between the last ok level and the first
broken one (80→160 becomes 100, 120, 140). Each level runs a short burst;
DB and Redis connections open once and stay warm across levels so early
buckets measure strain, not reconnect cost. Paced runners (`http`/`db`/
`redis`) scale their rate with concurrency so bigger crowds push more load;
scenario load comes from the VUs themselves. A level breaks when any runner's
P99 crosses its threshold or success drops under 95% — the verdict names the
culprit (`CAUSE db`, `http,redis`, …), and the report charts concurrency vs
P99 so you see a cliff or a slope, not just one number. `ramp:` and
`duration:` are ignored while a sweep runs (set them `0s`/anything; the
loader still requires the keys). The same mode is available as
`--capacity` / `--capacity-max-concurrency` / `--capacity-step-duration`
flags; the pre-0.6 names `auto_ramp:` and `--auto-ramp`/`--ramp-max-concurrency`/
`--ramp-step-duration` still load with a rename warning.

Same-machine caveat: running the generator on the same box as the app,
database, or Redis means all of them fight for the same CPU — the break
point you find is the machine's, not the system's. Good enough for
comparing configs and finding the culprit layer; not a production capacity
number. For a real one, generate load from a separate machine.

## The report

`report.html` contains:

- **Verdict** — one plain-words line (broke at N users / clean run), the
  bottleneck, and next steps. Always rendered; empty runs explain why.
- **Run summary** — requests, success %, P50/P95/P99/max/mean, rate, throughput,
  and the HTTP status-code histogram for each runner.
- **Correlated spikes** — a table of flagged buckets (runner, per-bucket P99
  values, bottleneck verdict) and an overall "bottleneck lean" readout.
- **Latency timeline** — every runner's per-bucket P99 on a shared x-axis so
  storage and HTTP latency can be compared directly.
- **Capacity sweep** (sweep runs only) — concurrency vs P99 with the break
  point, per-level verdicts, and the error buckets naming what failed.
- **Export JSON button** — in the top bar; downloads the run as the same JSON
  the `--json` flag writes, so a report opened in a browser can still feed a
  dashboard or a CI comparison.

![Latency timeline](./docs/todo-api-run-2.png)

![Correlated spikes table](./docs/todo-api-run-1.png)

*What those two screenshots show: a 3-minute heavy run against the TodoAPI stack (Gin + Postgres + Redis) — `GET /api/todos` over HTTP at 120/s, a weighted read/write query mix against Postgres at 80/s, and Redis commands at 300/s, with a 60s ramp and concurrency 50, generator and stack on the same machine. With the app's rate limiter at production settings it absorbed nearly the whole HTTP burst as 429s — the API stayed flat at ~5ms p50 while the real load landed on the data stores. With the limiter boosted, every request reached the backend: Postgres saturates and drags HTTP P99 to multi-second territory, while Redis stays under 100ms P99. One bottleneck, three correlated curves.*

The JSON export mirrors this structure: `generated_at`, `duration`, `ramp`,
`concurrency`, per-runner metrics (latencies in milliseconds), correlated spikes
(each with `runner`, `http_p99_ms`, `storage_p99_ms`, and `masked`), the
timeline, and — when the run was a capacity sweep — the `capacity_search`
curve (per-level concurrency/requests/p99/success, `break_at`, `last_ok`).
Capacity steps also carry an `errors` map when scenario steps failed at that
level (e.g. `{"dial_timeout": 5120, "5xx": 1800}`), so the breaking run's
story answers what failed, not just how much. In
the timeline's `p99_ms` series, `-1` marks a bucket where that runner had no
request (e.g. before the ramp produced its first hit); the report chart renders
these as gaps, not as a latency of -1ms.

## Concepts

### Spike correlation

1. All runners' buckets are aligned by their unix start time
   (`HTTPBucket.Start.Unix()` == storage `Bucket.Start`).
2. Each storage runner — DB and Redis — is checked independently against the
   HTTP run. A bucket is flagged when the storage runner's **P99 exceeds its
   threshold**, and the spike is either **correlated** (HTTP also crossed
   `http-threshold`, labeled with a bottleneck verdict) or **masked** (storage
   spiked while HTTP stayed under its own).
3. Masked spikes are still reported so a storage bottleneck that does not yet
   back up the application is surfaced. The CLI marks them `db-only` /
   `redis-only`, the HTML report tags them `DB (masked)` / `Redis (masked)`,
   and the JSON export sets `masked: true`. A bucket where both DB and Redis
   spike produces two rows.

Thresholds default to 100ms each and apply per runner (`--http-threshold`,
`--db-threshold`, `--redis-threshold`). In scenario mode the app-side
reference is synthesized from the worst per-bucket journey latency.

HTTP-only buckets are deliberately not flagged: a slow endpoint that leaves the
data stores idle is an application problem, not a storage problem.

### Scenarios vs plain HTTP

`scenario:` runs sequential HTTP steps per virtual user — it *is* your HTTP
load, in journey form instead of single shots. Each VU picks one scenario
once at launch (weighted by `weight`), then loops it until `duration`
expires. `extract` maps a var name to a JSON path (`$.token`,
`$.user.id` via gjson); the value is stored per VU and `{{var}}` is
interpolated into later step `url`, `body`, and `headers`. Missing vars stay
as `{{var}}` so misconfig is visible; non-JSON or missing paths leave the
var unset.

`scenario:` cannot be combined with `http:` — and that is deliberate, not a
limitation. Correlation needs exactly one app-side reference timeline per
bucket: either the `http` runner's P99, or the worst journey P99 synthesized
from scenarios. Two app curves would double-count rates, progress, and every
verdict. To mix plain hits with flows, model the plain hit as a one-step
scenario. Note `rate` also means different things per runner (`http`/`db`/
`redis` are paced per-second targets; scenario throughput emerges from VUs
looping). Scenarios can run alongside `db`/`redis` — buckets use the same
`Start.Unix()/bucket_width` scheme so timelines align.

### Rate, concurrency, and ramp

- `rate` is the *target* rate. If `concurrency` is too small to keep up, the
  pool backs up and throughput settles below target. This is intentional: a
  real load test should expose the target's limits rather than silently
  serializing requests.
- `concurrency` for HTTP maps to vegeta's `MaxWorkers`; unset (0) lets vegeta
  scale workers on its own. For DB and Redis it is the pool size; 0 selects the
  default of 10 workers. The run header reports which mode is in effect.
- `ramp` schedules hits so the rate grows linearly from 0 to full across the
  window (a 3s ramp at 2000/s fires roughly 3000 requests during the ramp, then
  holds 2000/s). With no `ramp`, the full rate applies from the first request.
- **Weighted mixed queries.** One query is picked per request, weighted, so a
  config can mix reads and writes the way real traffic does. The cumulative
  weight table is built once per run (v0.6.4+): building it per request cost
  O(n) alloc+scan each time, so a 10k-entry list at 2k req/s melted the
  generator and the timeouts were misread as database failures.
- **Read/write routing.** Each DB query's `type` field is authoritative
  (`read` runs through `Query`, `write` through `Exec`); untyped queries fall
  back to a heuristic on the SQL text.
- **Real parallelism.** Requests are submitted to a worker pool, so `rate` is
  not a serial request stream.

Engines per runner:

| Runner | Engine | Parallelism |
|---|---|---|
| HTTP | vegeta attacker | vegeta workers (bounded by `concurrency`) |
| DB | `database/sql` + [pond](https://github.com/alitto/pond) worker pool | `concurrency` workers |
| Redis | go-redis client + pond worker pool | `concurrency` workers |

The DB and Redis runners pace requests at `rate` per second, submitting each to
a pool capped at `concurrency` workers. Results carry the submission timestamp,
so buckets reflect when load was generated, not when responses completed.

A command still in flight when the run ends is aborted rather than counted
against the target: DB reads abort cleanly (no side effects) and are not
failures, while DB **write** errors still surface since the write may have
executed server-side. Redis commands aborted at shutdown before the server
answers (`context canceled`) are likewise not failures; any other Redis error
is a real failure.

### Live progress

While a run is going, each runner gets one runway lane: a 30-cell track with
a head that advances a cell per tick and a dot trail whose length follows the
rate on a log scale, so a 300/s cache and a 20/s database are both readable
on the same track. Errors show as a red `X` for that tick. Only runners that
fired get a lane — an http+redis run renders two rows, not four.

```
  ⠋ 00:45/03:00
  http  ················•••>··········  3,900   12/s
  db    ········•X····················    900    4/s  2 err
  redis ·············›················  1,350   40/s
```

Latency columns on the results table are right-aligned so magnitudes line up
on the decimal width, and every table is closed with light rules.

Pass `--no-progress` for plain 5s log lines instead (CI default in the
reference workflows). The view only exists on a TTY — piped output gets the
plain lines too.

## Configuration

The default config file is `