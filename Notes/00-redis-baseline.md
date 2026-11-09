# Redis Baseline Benchmark — Local Docker (M3 Pro)

*As of 2026-11-05 · KSV*

One Redis 8 container on an M3 Pro served about 340k SET and 368k GET requests per second, with a median latency under 0.08 ms. Pipelining 16 commands raised throughput 6.7–10×. Each command's median latency rose 2.5–3.7× at the same time. Moving the client from inside the VM to macOS cut throughput to about one third (113–115k requests per second).

## Purpose

This is the first entry in the benchmark log for the Distributed Confidential Caching thesis. It records plaintext Redis performance on a laptop, with no TEE involved. Later runs inside SEV-SNP and SGX environments are compared against baselines of this kind. This run is a rehearsal of the method, not a publishable baseline (see Threats to validity).

## Setup

| Item | Value |
| --- | --- |
| Machine | MacBook Pro, Apple M3 Pro, 18 GB unified memory |
| Container runtime | Not recorded: Docker Desktop or Colima? (`docker context ls` shows it) |
| VM CPU given to Docker | 11 cores visible to the container (inferred: RediSearch starts min(16, CPU cores) workers and started 11) |
| VM RAM given to Docker | Not recorded (`docker info`, line "Total Memory") |
| Redis image | `redis:8` → Redis 8.10.3, 64-bit, native arm64 (ARM CNTVCT clock, so no x86 emulation) |
| Modules loaded by default | RedisBloom, RediSearch (11 worker threads), RedisTimeSeries, RedisJSON |
| Persistence | Default RDB snapshots (after 1 change in 3600 s, 100 in 300 s, or 10,000 in 60 s) plus a final snapshot on shutdown |
| Network exposure | Port published on all host interfaces, no password (Redis logged a warning) |
| Client | Runs 1–3: `redis-benchmark` inside the same container, over loopback. Run 4: Homebrew `redis-benchmark` on macOS, through Docker's port forward into the VM |
| Clients in parallel | 50 (default) |
| Requests per test | 100,000 |
| Keys | One key only (no `-r` flag) |
| Run date | 5 November 2026 (runs 1–3), 6 November 2026 (run 4) |

```bash
docker run -d --name redis-lab -p 6379:6379 redis:8

# Run 1: baseline
docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get
# Run 2: pipelining, 16 commands per round-trip
docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get -P 16
# Run 3: 1 KB values instead of 3 bytes
docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get -d 1024
# Run 4: client on macOS (brew install redis), crossing into the VM
redis-benchmark -q -n 100000 -t set,get
```

## Results

Pipelining changed throughput by an order of magnitude. A 340× larger value cost only 10–24%.

| Run | Flags | Op | Throughput (req/s) | p50 latency (ms) | Throughput vs run 1 | p50 vs run 1 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 Baseline | — | SET | 340,136 | 0.079 | 1.00× | 1.00× |
| 1 Baseline | — | GET | 367,647 | 0.071 | 1.00× | 1.00× |
| 2 Pipelined | `-P 16` | SET | 2,272,727 | 0.295 | 6.7× | 3.7× |
| 2 Pipelined | `-P 16` | GET | 3,703,704 | 0.175 | 10.1× | 2.5× |
| 3 Large value | `-d 1024` | SET | 306,748 | 0.087 | 0.90× | 1.10× |
| 3 Large value | `-d 1024` | GET | 279,330 | 0.095 | 0.76× | 1.34× |
| 4 Host client | client on macOS | SET | 112,867 | 0.239 | 0.33× | 3.0× |
| 4 Host client | client on macOS | GET | 114,548 | 0.239 | 0.31× | 3.4× |

### Raw output

```text
❯ docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get
SET: 340136.06 requests per second, p50=0.079 msec
GET: 367647.03 requests per second, p50=0.071 msec

❯ docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get -P 16
SET: 2272727.25 requests per second, p50=0.295 msec
GET: 3703703.50 requests per second, p50=0.175 msec

❯ docker exec -it redis-lab redis-benchmark -q -n 100000 -t set,get -d 1024
SET: 306748.47 requests per second, p50=0.087 msec
GET: 279329.59 requests per second, p50=0.095 msec

❯ redis-benchmark -q -n 100000 -t set,get
SET: 112866.82 requests per second, p50=0.239 msec
GET: 114547.53 requests per second, p50=0.239 msec
```

### Server log timeline

Times are local (UTC+1). Redis's own timestamps inside the log are UTC.

| Time | Log event | Cause |
| --- | --- | --- |
| 14:32:30 | Redis 8.10.3 starts, loads 4 modules, ready on port 6379 | `docker run` |
| 15:32:31 | Snapshot: 1 key, 104 bytes | Rule "1 change in 3600 s" |
| 16:31:46 | Snapshot: 2 keys, 126 bytes | Rule "100 changes in 300 s", reached during run 1 |
| 16:32:48 | Snapshot: 2 keys, 1,148 bytes | Rule "10,000 changes in 60 s"; the 1 KB value shows run 3 had written |
| 16:33:17 | SIGTERM, final snapshot (2 keys, 1,148 bytes), exit | `docker stop` |

## Interpretation

The bottleneck is fixed per-round-trip overhead, not the work of storing or reading data.

**1. Baseline: overhead dominates.** Client and server share one machine, so a "round-trip" crosses no real network. It still costs two system calls (write, read), a pass through the kernel TCP stack and one event-loop wake-up on each side. A single GET in RAM takes far less time than that. So run 1 mostly measures overhead, which is why SET and GET differ by only 8%.

**2. Pipelining: throughput up, per-command latency up.** With `-P 16`, one round-trip carries 16 commands, so the fixed overhead is shared 16 ways. Throughput rose 6.7–10×. Each command now waits for its whole batch, so median latency rose too. Little's law ties the three numbers together for a closed-loop benchmark like this one:

```math
\text{mean latency} = \frac{\text{requests in flight}}{\text{throughput}}
```

| Run | Op | In flight | Implied mean latency (ms) | Measured p50 (ms) |
| --- | --- | --- | --- | --- |
| 1 | SET | 50 | 0.147 | 0.079 |
| 1 | GET | 50 | 0.136 | 0.071 |
| 2 | SET | 800 | 0.352 | 0.295 |
| 2 | GET | 800 | 0.216 | 0.175 |
| 4 | SET | 50 | 0.443 | 0.239 |
| 4 | GET | 50 | 0.437 | 0.239 |

In run 1 the mean is about twice the median, which points to a long tail of slow requests. `-q` hides that tail (see Threats to validity).

**3. Pipelining exposes the real cost of each command.** Once overhead is shared, the server's own work becomes visible. GET outruns SET by 1.6× in run 2, against 1.08× in run 1. A SET must allocate and replace a value; a GET only reads one.

**4. Value size: still overhead-bound.** Moving from 3 bytes to 1 KB cut throughput by 10% (SET) and 24% (GET). At about 300 MB/s of payload, the machine is nowhere near its memory or loopback bandwidth limit. Why GET suffered more cannot be settled from one run each. One hypothesis: the 1 KB reply must be copied into an output buffer and parsed by the client.

**5. Crossing the VM boundary costs two thirds of throughput.** In run 4 every request left macOS, passed through Docker's port forward into the Linux VM, and came back. Throughput fell to 0.31–0.33× of run 1 and median latency rose 3.0–3.4×. This happened even though the client no longer competed with Redis for VM CPU. SET and GET became identical (0.239 ms), so the boundary now hides the server's work completely. This is a small version of the thesis question: SEV-SNP guests also pay extra on every I/O, because data must be copied through unencrypted shared buffers.

## Threats to validity

These numbers are good for learning the method but too weak to cite in the thesis.

- **One key.** Without `-r`, every request hits the same key. That key stays hot in the CPU cache, which a real cache workload never allows.
- **One run per test.** There is no repetition and no spread, so differences under about 10% cannot be told apart from noise.
- **Median only.** `-q` prints p50 alone. The p99 and maximum, where caches actually hurt users, were never seen.
- **Client steals server CPU.** `redis-benchmark` ran in the same container as Redis and competed with it for the VM's cores.
- **Runs 1–3 never crossed the VM boundary.** Their traffic stayed on the VM's loopback. Run 4 crossed it, but a day later and with a Homebrew client whose version was not recorded (`redis-benchmark --version`).
- **Background snapshots during runs.** Redis forked to write snapshots during run 1 (16:31:46) and around run 3 (16:32:48). With 2 keys the cost was negligible (0 MB copy-on-write); with many keys a fork adds latency spikes.
- **Environment partly recorded.** The server log gave the Redis version, CPU count and persistence settings. The container runtime and VM RAM are still missing.

## Next runs

- [ ] Record what the log could not show: runtime (`docker context ls`) and VM RAM (`docker info`)
- [ ] Recreate the server with snapshots off and the port bound to localhost only, then save its startup log (`docker logs redis-lab`) to record which modules loaded:

```bash
docker rm -f redis-lab
docker run -d --name redis-lab -p 127.0.0.1:6379:6379 redis:8 \
  redis-server --save "" --appendonly no
```

- [ ] Use 100,000 random keys and a full latency report (no `-q`), repeated 5 times:

```bash
for i in 1 2 3 4 5; do
  docker exec redis-lab redis-benchmark -n 1000000 -r 100000 -t set,get
done
```

- [ ] Run the client in its own container, so it stops competing with Redis inside one container:

```bash
docker network create bench
docker run -d --name redis-net --network bench redis:8 \
  redis-server --save "" --appendonly no
docker run --rm --network bench redis:8 \
  redis-benchmark -h redis-net -n 1000000 -r 100000 -t set,get
```

- [x] Do the host-side run (`brew install redis`, then `redis-benchmark` from macOS) and compare it with the in-VM run (run 4)
- [ ] Repeat run 4 with `-P 16` to measure how much pipelining recovers across the boundary
- [ ] Commit this report and the raw output to the thesis repo as `notes/00-redis-baseline.md`
