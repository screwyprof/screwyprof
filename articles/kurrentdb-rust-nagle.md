<!---
title: "TCP_NODELAY: One Line, 135x Faster in KurrentDB's Rust Client"
description: 'A read from a local KurrentDB took 40ms while the database answered in ~1ms. The fix was one socket option: TCP_NODELAY made the client 135x faster.'
date: 2026-06-24
tags: [performance, debugging, rust]
related: []
--->
# TCP_NODELAY: One Line, 135x Faster in KurrentDB's Rust Client

> 📍 **Canonical version: [happygopher.nl/writing/kurrentdb-rust-nagle](https://happygopher.nl/writing/kurrentdb-rust-nagle/).** This copy is kept for existing links.

I have a small Rust service backed by an in-memory KurrentDB, all on localhost. No disk, no network, and still it felt slower than it had any right to be. The access log gave it away: almost every response for one endpoint came back around 40ms, the same figure on every call, no matter what I asked for. An in-memory store on loopback has no business being that slow, or that consistent about it. So I started pulling the thread.

**TLDR.** The Rust client's gRPC transport never set `TCP_NODELAY`. With Nagle's algorithm on, every small request stalled in the kernel waiting for a delayed ACK that never came. On localhost, with no network latency to hide behind, that wait was the whole cost: a 42ms median read. One line, `http.set_nodelay(true)`, dropped it to 311µs. That's 135x from a single socket option.

## 1. The symptom is a suspiciously round number

The first clue is how _round_ the number is. Real work is jagged, shifting with payload, cache, and load, and it rarely lands on the same value twice. A latency pinned to one round figure points to a timer somewhere in the path.

KurrentDB also exposes a plain HTTP API, so I ran the same lookup over it. It answered in ~1ms: same node, same data, a fortieth of what my gRPC client was seeing. The database was doing its job in a millisecond. The other 39 were overhead in the client path, getting the question there and the answer back over loopback.

## 2. The benchmark: 42ms → 311µs

I measured the cheapest call I could think of: a read of a stream that doesn't exist. The server has almost nothing to do, so the number is mostly round-trip plus client overhead. In essence:

```rust
// divan runs this in a tight loop and reports the distribution
let stream = format!("bench-worksheet-{}", Uuid::new_v4());
bencher.bench(|| rt.block_on(async { store.load(&stream).await }));
```

| Configuration            | Median | Fastest |
| ------------------------ | ------ | ------- |
| Before (Nagle on)        | 42ms   | 40ms    |
| After (`TCP_NODELAY` on) | 311µs  | 225µs   |

_Setup: a single-node KurrentDB (`kurrentplatform/kurrentdb:26.1.0`, in-memory, insecure) in a local dev container on Apple Silicon, reached over plaintext gRPC through the official Rust SDK (the `kurrentdb` crate, 1.x, built with Rust 1.96). Timings come from [divan](https://github.com/nvzqz/divan); the ~1ms baseline is the same node answering over its AtomPub HTTP endpoint. Client and server share the container's Linux network, so the ~40ms below is Linux's delayed-ACK default._

Look at the "fastest" column, not the median. Even at its best, the old path couldn't get under 40ms. That hard floor is the fingerprint of a fixed timer. Turn the option on and it disappears: about 135x on the median.

## 3. Nagle's algorithm, 1984

The bug goes back to RFC 896, John Nagle, 1984. Picture that year's internet: congested links, telnet sessions firing a TCP packet for every keystroke. A one-byte payload in 40 bytes of headers is roughly 4000% overhead, and enough of those tiny packets could collapse a link.

Nagle's fix was clever: if you have unacknowledged data in flight, hold new small writes and coalesce them instead of sending another undersized segment. Wait for the ACK, then send what's accumulated. On a 1984 telnet session, a clear win.

And it's still on by default, four decades later, in a world that looks nothing like 1984.

## 4. Delayed ACK, the other half of the trap

Nagle alone wouldn't hurt you. The damage comes from a second optimization nobody designed to interact with the first. Delayed ACK: the receiver doesn't acknowledge every segment immediately. It waits a short while (tens of milliseconds; ~40ms is common), hoping to piggyback the ACK on an outgoing response and save a bare packet.

Now watch the two meet on a request/response protocol like gRPC. The client sends a request. If the framing leaves a small trailing write while data is still unacknowledged, Nagle holds it back. The server has nothing to respond with yet, so it sits on its delayed-ACK timer. Neither side is misbehaving; both are politely waiting for the other until that timer finally fires. Your "instant" local query has aged by a full timer, stuck in a deadlock of good intentions.

It needs the right setup to bite. A lone request that fits one segment, with nothing unacknowledged, slips through. The trap is the multi-write pattern gRPC inherits from HTTP/2 (headers in one write, data in the next): the trailing write waits on an ACK that delayed ACK is busy postponing.

Nagle himself put it best: _"The combination of the two is awful."_

Why is this loudest on localhost? Over a real network, round-trip latency keeps data moving in both directions, the stall conditions rarely line up, and the symptom hides. On loopback, with sub-millisecond round trips, nothing masks the timer. The faster your setup, the more obvious the bug. Marc Brooker's post ["It's always TCP_NODELAY. Every damn time."](https://brooker.co.za/blog/2024/05/09/nagle.html) makes the same case: on modern hardware the overhead Nagle was built to fight is long gone, so for anything latency-sensitive, turn `TCP_NODELAY` on and don't think twice.

## 5. The one-line fix (and why a mature client shipped without it)

The bug is mundane. The interesting part is how it survived in a widely-used client. The Rust SDK doesn't use tonic's default transport; it hand-builds a connector on hyper to control TLS and a few other things. The moment you do that, you opt out of the defaults the higher-level stacks set for you, and `set_nodelay(true)` is one of them. Go's gRPC sets it, Java's gRPC sets it, tonic's own transport sets it. This connector didn't.

So here's the whole fix, in `kurrentdb/src/grpc.rs`:

```rust
let mut http = HttpConnector::new();
http.enforce_http(false);
// gRPC sends small, latency-sensitive frames; don't let Nagle batch them
http.set_nodelay(true);
```

That last line, [`grpc.rs:899`](https://github.com/kurrent-io/KurrentDB-Client-Rust/blob/d76e58ba464b2dc77c196ffefbca330ce9df938d/kurrentdb/src/grpc.rs#L899), is the entire change (pinned to the merge commit).

I filed [issue #232](https://github.com/kurrent-io/KurrentDB-Client-Rust/issues/232) with the patch, and the maintainers applied it in [#233](https://github.com/kurrent-io/KurrentDB-Client-Rust/pull/233). The 42ms median became 311µs.

**The takeaway.** If you ever hand-build a connector for a request/response protocol, set `TCP_NODELAY` yourself. The libraries that feel like they "just work" are the ones already doing it for you. The cost stays invisible until the day a 1ms database takes 40ms to answer on loopback. The cost of fixing it is one line. Go grep your transport setup.

## References

- [KurrentDB-Client-Rust issue #232](https://github.com/kurrent-io/KurrentDB-Client-Rust/issues/232): the report and proposed patch
- [KurrentDB-Client-Rust PR #233](https://github.com/kurrent-io/KurrentDB-Client-Rust/pull/233): the applied fix
- [The fixed line, pinned to the merge commit](https://github.com/kurrent-io/KurrentDB-Client-Rust/blob/d76e58ba464b2dc77c196ffefbca330ce9df938d/kurrentdb/src/grpc.rs#L899): `http.set_nodelay(true)` in `kurrentdb/src/grpc.rs`
- [Marc Brooker, "It's always TCP_NODELAY. Every damn time."](https://brooker.co.za/blog/2024/05/09/nagle.html): Nagle's algorithm and delayed ACK
- [RFC 896](https://www.rfc-editor.org/rfc/rfc896): Nagle's original 1984 description
