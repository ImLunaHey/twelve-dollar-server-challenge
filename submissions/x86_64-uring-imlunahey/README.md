# x86-64 NASM / raw io_uring / SQLite

This is a direct HTTP/1.1 server for Linux x86-64. The event loop, HTTP parser,
router, JSON reader/writer, base64url, SHA-256/HMAC, JWT checks, response
buffering, and SQLite integration are written in NASM. SQLite is compiled from
its pinned 3.54.0 amalgamation and is the only C component.

It serves `HOST:PORT` directly, without Nginx. Raw `io_uring` uses sparse fixed
file tables and provided buffers; a raw `epoll` backend is retained for kernels
that deny `io_uring`. One request owner and one NOMUTEX SQLite connection retain
the single-threaded request path. A second NOMUTEX connection belongs only to a
checkpoint helper thread. Writes found in the same frozen completion batch share
a transaction, but responses are not made visible until `COMMIT` succeeds.

## Performance choices

- SQLite 3.54.0 is built from its checksum-pinned official amalgamation with
  `-O3`, native-code LTO, and synthetic file-database PGO. Multi-thread mode is
  compiled in, while both separately owned connections disable connection mutexes.
- WAL, `synchronous=NORMAL`, a 1 GiB mmap window, and prepared statements are
  used. All request data is still read from SQLite during that request.
- Passive WAL page copying runs on the helper connection every 4,096 additional
  frames. At 32,768 frames the request owner attempts a restart checkpoint at a
  transaction-free batch boundary so sustained writes cannot indefinitely
  outrun passive copying. A busy helper is retried at the next boundary without
  waiting on its checkpoint lock. WAL retention after reset is limited to 16 MiB.
- The default feed discovers and serializes the newest 20 live rows once, scans
  the bounded live like-ID range, and fills in the counts in assembly. Sparse or
  unusually dense ranges fall back to the general live query.
- Read responses are serialized directly from SQLite columns in assembly.
- JWT HMAC-SHA256 is recomputed for every authenticated request.
- Fixed response fragments use compile-time-sized copies; ASCII JSON spans use
  an SSE2 scanner; integers use reciprocal multiplication instead of division.
- Completion collection does not sleep when a ring already has work and permits
  at most one timed wait per complete eight-ring owner sweep. This reduces the
  high-population tail stalls seen with a timed wait on every ring.
- Sends are drained before receives under pressure, and exhausted provided
  buffers are rearmed from bounded deferred queues.
- Teardown cancels only still-pending operations, using exact generation-tagged
  keys instead of FD-wide cancellation scans during mass disconnects.
- Registered-file acceptance cascades through the first four rings. Near 260k
  live sockets the remaining four acceptors activate together, distributing the
  final working set instead of driving the eighth 65,535-slot table to its edge.

The release defaults target 500,000 users, but that target is not a measured
organizer score. The candidate passes 42/42 official checks and the extended
protocol/crash-recovery suite. The preceding checkpoint-guard binary passed an
unchanged-k6 five-minute hold at 40,000 users: p95 0.668 ms, p99 4.972 ms, zero
errors and 580.2 MiB peak server-cgroup memory. The newer teardown-fixed binary
completed a native mjc five-minute hold at 275,000 users with zero errors, but
**failed p99** (3,543.983 ms; p95 74.842 ms). No five-minute pass above 250,000
users or maximum-user score has been established yet.
A matched separate-generator run remains necessary before claiming any
leaderboard result.

## Provenance

Derived under the repository's MIT license from Melroy van den Berg's
`asm-nasm-melroy89` submission (PR #86), including its test suite and build
system. This variant integrates that submission's published experimental
collection policy and integer formatter and enables its validated SSE2 quoting
and fixed-literal paths by default. Changes and authorship are intentionally
documented rather than presented as a clean-room implementation.

## Build

On clean Ubuntu 24.04 x86-64:

```bash
sudo bash install.sh
bash build.sh
SQLITE_PATH=/path/to/feed.db JWT_SECRET=twelve-dollar-challenge \
  HOST=0.0.0.0 PORT=80 bash start.sh
```

`bash build.sh diagnostic` skips SQLite PGO for a quicker debug build. Build
outputs and downloaded dependencies are ignored.

## Reproduce local profiling

`tools/profile.py` runs a fresh-database mixed saturation diagnostic (the
challenge's 100:100:15:2 endpoint proportions, without think time). It supports
`--profile` for sampled CPU profiling and `--server-arg=--checkpoint=passive`
to compare the old checkpoint policy in the same binary. It is not a VU score.

`tools/qualify.py --users 40000` runs the unchanged `bench/load.js`: a two-minute
1,000-user warmup, 60-second ramp, full five-minute hold, and normal ramp-down.
It requires passing latency/error thresholds, all response checks, no OOM kill,
clean shutdown, and an SQLite integrity check. Both runners record the binary
hash and resource evidence under ignored `results/` directories.

`bash tools/build-mjc.sh` builds the pinned alternate generator with Rust 1.99.1
and locked dependencies inside Ubuntu 24.04. Then `python3 tools/mjc.py --users
275000 --duration 5m --warmup` performs a fixed-count native confirmation. The
runner invokes the native orchestrator directly, avoiding the upstream shell
launcher's host sysctl changes. It uses a separate client cgroup and requires
native thresholds, zero failed endpoint checks, clean shutdown and database
integrity before recording a pass. `--score --start-users 275000 --max-users
500000` selects the native search workflow; a selected-count trial is not a
maximum search. The same Nix-runtime limitation applies to this local wrapper.

These local Docker runners currently use Nix runtime executables and the
`redis:8-alpine` image solely as a container shell; no Redis server is started.
Run under `nix-shell -p python3 k6 wrk perf`. The server gets one CPU and 2 GiB
with no swap; clients are separately accounted. Private same-host loopback,
CPU placement, and a seccomp exception for io_uring make this diagnostic setup
different from the organizer's unpinned droplet and separate load generator.
See [PERFORMANCE.md](PERFORMANCE.md) for results and limitations.
