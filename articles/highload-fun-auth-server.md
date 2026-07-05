<!---
title: "What Actually Moved the Score: A Rust Auth Server on highload.fun"
description: "From a 9.198s first pass to a 5.750s record on highload.fun and first place, at ~20ms average latency against the previous record's ~34ms (~87k req/s, 2 cores, 10,000 connections). What actually moved the score (a TCP RST bug, write coalescing), why the usual suspects (locks, allocation, CPU) were already flat, and why io_uring was a dead end."
date: 2026-07-05
category: performance
--->
# What Actually Moved the Score: A Rust Auth Server on highload.fun

A [LinkedIn post](https://www.linkedin.com/pulse/auth-server-competition-back-time-never-ends-sergey-svistunov-jswze/) from [Sergey Svistunov](https://www.linkedin.com/in/sergey-svistunov/) crossed my feed, announcing that his [Auth Server Competition on highload.fun was back, permanent and always-on](https://highload.fun/challenges/server/authserver/overview). I know him from `Lazada`, where he worked on low-level performance for a platform under real traffic. He is one of the few people I have met who say "high load" and mean it, not the rehearsed version you get in a systems-design interview. I do not usually do challenges like this one, but this came from him, and the pitch was good: optimize a whole HTTP service under real load, not a tight inner loop. So I took a look.

[highload.fun](https://highload.fun) runs a brutal little benchmark: an HTTP auth server, shipped as a Docker image, hit with 500,000 requests across 10,000 concurrent connections on a 2-core, 2-GB container. Score is test duration plus one second per error, lower wins. Security counts as much as speed: the checker actively tries to break in. Every run uses a fresh, hidden dataset, so there is nothing to overfit.

I took the server from a **9.198s** first correct run to **5.750s**, past the previously recorded leader at **5.964s**, and it has held first place since.

**TL;DR.** The usual performance work (locks, allocations, CPU) was already flat and moved nothing. Every second I gained came from the network. The biggest single win was a TCP RST-on-close bug that truncated `GET /user` responses and was invisible on localhost, worth 37 seconds of score; the rest came from coalescing pipelined writes. The result was **5.750s** and first place, holding **~20ms** average latency against the previous record's **~34ms**. The one lever left below the syscall floor is io_uring, and the platform's seccomp blocks it.

One honest note before the tuning. A laptop can generate load, but it cannot test a network. Over loopback there is no round-trip time and nothing is ever in flight, so the TCP-level bugs never appear; my only real network was the platform's benchmark harness, a black box I poked by pushing an image and reading the score back. Every change that moved the score lived there. Allocation, locking, and CPU I had kept tight from the first commit, so the network was the only bottleneck left.

## Correctness first, performance rules from line one

None of the speed matters until the server is correct, and correctness here is adversarial. A byte-exact checker forges tokens (the JWT secret is public, so a valid signature proves nothing on its own), probes IP and geo blocks, and reaches for users it should not see. Every incorrect response costs a full second; fail all of them and you score 500,000 seconds. So the first milestone was correctness: the first submission that came back with zero errors scored **9.198s**.

From the first commit the code followed a few performance rules, because on a 2-core box under 10,000 connections you do not get to bolt that on later:

- **As few allocations as possible, and zero on the per-request hot path.** Responses serialize into a reused, fixed-size buffer rather than a freshly allocated one, so no request touches the heap. An allocation-counting test guards it, so a regression fails the build. The only allocations are per-connection setup, amortized across every request that connection serves.
- **Every lookup in memory and O(1).** User, blacklist, and subnet checks are hash lookups, never scans.
- **Benchmarks and no-alloc tests as first-class citizens**, written alongside the code, not bolted on after.

And a deliberate call on JSON: on the hot path, JSON is not your friend. But the contract is fixed, byte-identical, and the API surface is small, so I hand-rolled byte scanners for the request bodies and templated the responses straight into that fixed buffer. `serde_json` earns its keep only in the one-time startup load, where it also doubles as an independent oracle in the tests.

## The grind: bench, profile, optimize

With a correct 9.198s server, the next phase was the loop every optimization story is actually made of: benchmark, profile, change one thing, measure. On the `hyper`-based server that meant the usual tuning: HTTP/1.1 pipeline flush, vectored writes, dropping the date header and its per-response timer, a two-thread runtime, keeping the hot path allocation-free, `mimalloc` as the global allocator, and `TCP_NODELAY` on every connection.

That last one I already knew well. Some time ago [Nagle's algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm) cost me 40ms per call on the *client* side of a different system, because its gRPC transport never set `TCP_NODELAY` ([I wrote that one up here](https://github.com/screwyprof/screwyprof/blob/main/articles/kurrentdb-rust-nagle.md)). On the server it is the same lesson from the other end: leave Nagle on and the kernel batches your own small responses to save packets, trading latency for bandwidth nobody asked for. Or as Marc Brooker titled his own account of it, ["It's always TCP_NODELAY. Every damn time."](https://brooker.co.za/blog/2024/05/09/nagle.html)

Each push shaved off a little less than the last, and around 7.4s they stopped helping at all: more tuning, same score. (One detour, a thread-per-core runtime, I tried and reverted; it did not help on this workload.)

So I stopped tuning and profiled the `hyper` server under load. Half the CPU sat in the kernel, in epoll and the TCP stack, which nothing inside the epoll model gets you under. My handler was about 1%. Between them, at roughly 10%, was `hyper`'s own HTTP/1 machinery: header handling, its write path, and its buffered I/O layer, pure framework overhead between my handler and the socket (the raw parse, `httparse`, was a separate ~1%). That was the biggest reachable slice on the board, and it was worth more than the 10% suggests, because the ceiling was I/O, not CPU: `hyper`'s state machine made four to six trips through the event loop per request, where a hand-rolled loop needs about two.

So I dropped `hyper` from the hot path. This was not writing an HTTP server: `httparse` kept doing the parsing, the same crate `hyper` calls under the hood, and parsing was never the cost. What I replaced was how `hyper` drives connections. And I could, because this is not a general-purpose server: the contract is a small, fixed set of methods, so the loop can skip most of HTTP/1, the keep-alive corner cases, chunked encoding, the rest of the legacy. What remains is thin: read into a buffer, run `httparse` over the head with no I/O, dispatch, write.

Going raw also meant hand-rolling the accept path, and that had a trap. At first the `setsockopt` calls that set `TCP_NODELAY` and `TCP_USER_TIMEOUT` per connection ran inside the accept loop. Ten thousand connections is nothing to `tokio` by itself, but the container has two cores and no slack: doing that work in the loop starved it under a burst of new connections, overflowed the accept queue, and dropped SYNs, so clients got an RST before sending a byte. Moving the calls into the per-connection task fixed it. On two cores, even setting a socket option is a scheduling problem.

The raw server was faster than `hyper`. It also introduced a ghost.

## The fixes that moved the score

### The phantom truncation, worth 37 seconds

The raw server came back from its first benchmark with about 51 errors, all identical: "Cannot read from server: Connection reset by peer." Every one on `GET /user`. Never on `POST /auth`, never anywhere else. The old `hyper` server had zero on the exact same workload.

The count was maddeningly stable, somewhere between 44 and 51, run after run, no matter what I changed at the socket level: listen backlog, FIN ordering, `TCP_USER_TIMEOUT`. A bug that holds its value under every change you throw at it is telling you that you have not found its actual cause.

I ruled out, with evidence: response-buffer overflow, `Content-Length` mismatches, a serialization panic, lock contention, and a pipelining bug (the integration tests replayed pipelines cleanly). And the loudest false signal of all: running it locally at 500k requests over 10k connections, repeatedly, I got zero errors every single time. The bug did not exist on my machine, and once I understood why, I understood the bug.

When you close a TCP socket that still has unread bytes in its kernel receive buffer, the kernel does not send a polite FIN. It sends an **RST**. An RST also discards your own unacknowledged send buffer, so a response the client is still reading is destroyed in flight, and its read fails with `ECONNRESET`, the "connection reset by peer" from the error. Two things made it `GET /user` only:

1. It returns the largest response body, so it takes the longest to send and is the most likely to still be unacknowledged in the send buffer when the socket closes. The token and empty responses are small and clear first.
2. Loopback cannot show it. `localhost` delivers instantly, so by the time the socket closes the send buffer is already drained. You need real-network round-trip time to leave bytes in flight, and my dev machine had none.

The fix is a pattern with an old name, a **lingering close**: send the FIN first so the peer stops expecting answers, then drain the receive buffer to EOF, bounded by a timeout. Generically:

```rust
// FIN first, then drain the peer to EOF, bounded by a timeout,
// so the kernel closes with a FIN instead of an RST that would
// discard our still-unacked response.
async fn lingering_close(sock: &mut TcpStream) {
    let _ = sock.shutdown().await;               // send FIN
    let mut scratch = [0u8; 1024];
    let _ = timeout(DRAIN, async {
        while sock.read(&mut scratch).await.unwrap_or(0) > 0 {}
    }).await;
}
```

This is roughly what `hyper` does internally, which is why `hyper` had zero errors. Two parts were easy to get wrong. A bare `shutdown()` is not enough: closing with unread receive data resets the connection regardless of a prior half-close, so you need the FIN *and* the drain. And a naive drain is worse than the disease: my first attempt read and discarded before closing, ate pipelined requests the client was still waiting on, and produced a different truncation. That version got reverted. The FIN has to come first.

It landed in two deploys. The first cut errors from 51 to 37; the residual 37 were the drain timing out under 2-CPU scheduling starvation, so I gave it more headroom and widened the parser's read limits to match what `hyper` accepts. That took 37 to 0.

Here is the payoff. At 37 errors the score was 42.823s, which means the work itself took about 5.8s and the other 37 were pure penalty. Clearing the last of them cost about 0.7s of real duration, because the longer, more patient drain has to wait on each closing connection, and the error-free score settled at **6.572s**. I would take that trade every run: a TCP close-semantics bug, invisible on my machine, was worth more than any throughput work I did before or after.

### Coalescing writes

With errors at zero, the question turned pure: throughput. The leader was 5.964s. An `strace -c` on the raw server showed where its syscall time went (time in syscalls, not total CPU), and it was blunt:

- `sendto` ~56%, `recvfrom` ~42%. Send and receive together were **98.6%** of it.
- `epoll_pwait` ~1.4%, one wakeup serving ~62 requests, so the event loop was already well batched.
- `futex` ~0. The store's `RwLock` was completely uncontended.

That is a server sitting on the "epoll floor": about two syscalls per request, and those syscalls are the whole cost. There was no lock to fix, no allocation to remove, no CPU to reclaim.

The production client pipelines, which I knew because the reverted naive drain had eaten pipelined requests. Its depth is its own business; I only needed a pipeline to reproduce the effect, so I built a local client that pipelines eight deep and re-profiled. The ratio was 8 `sendto` to 1 `recvfrom`: one read pulled in eight requests, and the server wrote eight separate responses. `hyper` coalesces those; my hand-rolled loop did not.

The fix is structural: drain every buffered request in one pass, append each response to a single accumulator, and flush the batch with one `write_all`. N requests become one write, and when there is nothing to coalesce it still writes once, so it costs nothing when it cannot help. The verification is my favorite detail of the project: a single `GET /user` response is exactly 177 bytes, and after the change the syscall trace showed `sendto` calls of 1416 bytes, which is 8 × 177, eight responses in one write, confirmed on the wire. Result: **+13% throughput** and **5.807s**, past the leader.

### The io_uring dead end

The profile was clear about the floor. At two syscalls per request with send and receive as the whole cost, the only way lower is to stop making one syscall per operation, which means [io_uring](https://en.wikipedia.org/wiki/Io_uring): a completion model that batches many operations into a single `io_uring_enter`. So I built a version of the server on it. It was faster and correct. Then I deployed it, and every worker panicked instantly: `io_uring_setup` returned `EPERM`, and every one of the 500,000 requests failed. The production container's seccomp profile blocks the io_uring syscalls, which is Docker's default. io_uring is the one lever left below the epoll floor, and the platform forbids it.

Back on the proven server, one small gain was left, the kind that does not touch syscalls at all: draining the read buffer with a cursor instead of an O(n²) shift per request, and a bigger response accumulator so even deep pipelines coalesce into one write. That took 5.807 to **5.750s**. Modest, but it was the record, and it has held since.

## Where the cost actually lived

On a workload like this, the cost lives in a predictable order, and most of it stops mattering once the groundwork is done. The usual suspects, roughly in that order, are I/O, then allocation and syscalls, then CPU work like crypto. These are orthogonal: each one can sink you by itself, and each has to be paid down separately. I paid the lower ones down from the first commit, so by the time I profiled under load they were already flat, and the entire remaining cost had moved to I/O:

- **I/O was the whole cost.** Both wins that moved the score were about I/O: not truncating responses (the RST fix) and making fewer, bigger writes (coalescing).
- **CPU was not the constraint.** The handler does about a microsecond of real work, HS256 signing included. Crypto can be expensive, but here it was ~1% of CPU, and the server ran at about 66% of two cores, so it was never CPU-bound.
- **Allocation and locking were flat by design.** The hot path was allocation-free and the lock never contended, both rules from the first commit, so there was nothing left to win there. You cannot speed up what already costs nothing (Amdahl); grinding on either would have been optimizing a dimension I had already closed.

The latency numbers followed from this too. Every endpoint sat at a uniform ~20ms, which by [Little's Law](https://en.wikipedia.org/wiki/Little%27s_law) is just in-flight concurrency divided by throughput (~1740 ÷ ~87k is about 20ms). There is no term for latency to attack on its own; the only way down is to raise throughput. When I did, the leader's ~34ms per endpoint became ~20ms for me, and connection setup fell to 0.573s against the leader's 0.794s.

## Where it landed

The score walked down **9.198 → 7.406 → 6.572 → 5.807 → 5.750s**, zero errors, past a recorded leader at 5.964s. As of this writing, Sunday 5 July 2026, it still holds first place. The leaderboard never closes, so that will not last forever, but for now it is mine.

The wins were specific to this platform: RST-on-close semantics and syscall coalescing, in a regime where the CPU was never the constraint. The method carries over even when the numbers do not: on a service like this, find your I/O cost before you touch anything else, and be wary of any optimization you can only measure on your own machine.

*(This was an entry in an open, always-on competition, so this post stays at the level of technique and lesson. The illustrative snippets are generic patterns, not the competition source.)*
