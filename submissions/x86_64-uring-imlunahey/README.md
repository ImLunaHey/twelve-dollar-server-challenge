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
  frames instead of stalling the request owner. WAL retention after a successful
  reset is limited to 16 MiB.
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
- Registered-file acceptance cascades through the first four rings. Near 260k
  live sockets the remaining four acceptors activate together, distributing the
  final working set instead of driving the eighth 65,535-slot table to its edge.

The release defaults target 500,000 users, but that target is not a measured
organizer score. The exact candidate passes 42/42 official checks and the
extended protocol/crash-recovery suite. A matched separate-generator five-minute
run remains necessary before claiming any leaderboard result.

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
