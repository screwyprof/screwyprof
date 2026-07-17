# Maksim Shcherbo — Staff Backend Engineer

**I build infrastructure that must not fail — and stop outages before they happen.**

Distributed Systems · Reliability · Scalability · Go · Rust · Ethereum

🌐 **[happygopher.nl](https://happygopher.nl)** · 📄 **[Résumé](https://happygopher.nl/resume)** · ✍️ **[Writing](https://happygopher.nl/writing)** · *(also known as [@screwyprof](https://github.com/screwyprof))*

---

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev)
[![Rust](https://img.shields.io/badge/Rust-D34516?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Docker](https://img.shields.io/badge/Docker-257bd6?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com)
[![gRPC](https://img.shields.io/badge/gRPC-protocol-blue?style=flat-square&logo=grpc)](https://grpc.io)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-blue?style=flat-square&logo=opentelemetry)](https://opentelemetry.io/)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-6BA539?style=flat-square&logo=openapiinitiative&logoColor=white)](https://www.openapis.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?style=flat-square&logo=amazon-aws&logoColor=white)](https://aws.amazon.com)
[![Ethereum](https://img.shields.io/badge/Ethereum-3C3C3D?style=flat-square&logo=ethereum&logoColor=white)](https://ethereum.org)

---

> I build **high-load, fault-tolerant** backends in **Go** and **Rust**, and ship fixes upstream into the open-source infrastructure they run on. **Every claim below is backed by a merged PR, a public case study, or a number you can check.**

## About

I make reliability systematic, not reactive. Over ~13 years I've taken fragile financial, e-commerce, and blockchain backends and made them scale predictably and recover on their own — pairing deep engineering discipline with simple, legible design. I set technical direction as a hands-on IC: standards, RFCs/ADRs, and CI gates, not headcount.

---

## 🧩 Open-source contributions

Public and independently verifiable — issue numbers, merged PRs, measured impact.

#### [KurrentDB Rust client](https://github.com/kurrent-io/KurrentDB-Client-Rust) — 42 ms → 311 µs (~135×)

Every gRPC call in the official Rust client (formerly EventStoreDB) paid a **~40 ms latency penalty**, silently slowing every read, append, and subscription on low-latency networks. Root cause: the transport never enabled `TCP_NODELAY`, so Nagle's algorithm and TCP delayed `ACK` stalled each request behind a kernel timer. My **one-line fix** dropped the median read from **42 ms to 311 µs (~135×)** and was merged upstream. [Issue #232](https://github.com/kurrent-io/KurrentDB-Client-Rust/issues/232), [PR #233](https://github.com/kurrent-io/KurrentDB-Client-Rust/pull/233), and the [write-up](https://happygopher.nl/writing/kurrentdb-rust-nagle/).

#### [Lighthouse consensus client](https://github.com/sigp/lighthouse) — O(n) → O(1), restarts eliminated

Validator updates re-decrypted **every** keystore on **every** call — an `O(n)` scrypt password-hash that restarted validators every **~2 hours** and blocked institutional upgrades. I made the path `O(1)` (skip decryption when there are no local keys); upgrades resumed at **99.9% attestation**, unblocking tier-1 staking providers. [Issue #4936](https://github.com/sigp/lighthouse/issues/4936), [PR #4126](https://github.com/sigp/lighthouse/pull/4126), and the [write-up](https://happygopher.nl/writing/lighthouse-web3signer-scrypt/).

#### [MEV-Boost](https://github.com/flashbots/mev-boost) — ~26% lower latency

Introduced a **per-validator relay architecture** — relays configurable per validator, hot-reloadable via an atomic lock-free swap, with the `Prometheus` instrumentation it originally lacked. **~26% lower latency.** [Issue #455](https://github.com/flashbots/mev-boost/issues/455), [PR #470](https://github.com/flashbots/mev-boost/pull/470), and the [write-up](https://happygopher.nl/writing/mev-boost-hot-reloadable-relay-config/).

#### [Autonity](https://github.com/autonity/autonity) — BFT proof-of-stake on go-ethereum

Patched go-ethereum to run a Tendermint-style BFT consensus layer with delegated PoS and 1-second blocks — a working prototype **before Ethereum's Merge**.

---

## 💼 Selected experience

Full history, with metrics and context, lives on **[happygopher.nl/resume](https://happygopher.nl/resume)**.

#### [ConsenSys](https://consensys.io/staking) — MetaMask Staking backend · $2B+ staked, 33k+ validators

Built the multi-tenant staking backend behind MetaMask and institutional clients from the ground up. Correctness-first API: row-level-security isolation, idempotent writes, a checkpointed state machine, and end-to-end audit trails — no duplicate side-effects, safe resume after partial failures. Established SDLC practice as an IC — OpenTelemetry-by-default, RFC-first features, ADRs, CI gates — adopted across my team and picked up by adjacent ones. Deep dive: [Building Financial Infrastructure That Must Not Fail](https://happygopher.nl/writing/building-financial-infrastructure-that-must-not-fail/).

#### [Lazada (Alibaba Group)](https://www.lazada.com/en/) — product-catalog backend · 16k+ RPS

Migrated the product-catalog domain from a legacy PHP monolith to distributed Go microservices — **16k+ RPS/instance, p95 <100 ms, 99.9%** through 11.11 / 12.12 mega-sale surges across six markets. CQRS read/write split over sharded MySQL; advocated Clean Architecture and TDD across teams.

#### Auction & Tender House — regulated e-procurement · 99.9% uptime

Rebuilt a fragile monolithic trading platform into an event-driven system with **DDD, CQRS, and Event Sourcing** — critical paths from **minutes to milliseconds**, deterministic verifiable records, **FSB/FSTEK-audited**, DDoS-hardened. Introduced CI/CD pipelines and structured on-call, setting the org's reliability standards.

#### Distributed randomness (R&D) — DAO.Casino

Pioneered verifiable, unbiasable on-chain randomness — a threshold-BLS random beacon built into Tendermint consensus, delivered end-to-end (patched Tendermint, a Cosmos SDK app, and a Go DKG library).

---

## 💎 Notable projects

*Personal projects showcasing systematic approaches to reliability, architecture, and testing.*

- **[payment](https://github.com/screwyprof/payment)** — a complete `DDD / CQRS / Clean Architecture` implementation: event-sourced aggregates, invariant enforcement, full test coverage.
- **[cqrs](https://github.com/screwyprof/cqrs)** — event-sourced CQRS library with a Given/When/Then DSL for business-readable domain specs — a rare pattern in Go.
- **[form3api](https://github.com/screwyprof/form3api)** — Form3 API client with GitHub-style HATEOAS pagination and executable documentation as acceptance tests.
- **[delegator](https://github.com/screwyprof/delegator)** — high-performance Tezos delegation service (CQRS): **15 s** full sync of **760k+** delegations, **<1 ms** API responses, 92% coverage.
- **[favkit](https://github.com/screwyprof/favkit)** — Rust CLI for macOS Finder favorites, with ADR-based architectural reasoning and Nix reproducibility.

---

## ✍️ Writing

Full archive on **[happygopher.nl/writing](https://happygopher.nl/writing)**.

- [Making Operational Decisions Explainable](https://happygopher.nl/writing/making-operational-decisions-explainable/)
- [Designing Hot-Reloadable Per-Tenant Routing](https://happygopher.nl/writing/mev-boost-hot-reloadable-relay-config/)
- [A Password Hash on Every API Call: The Bug That Restarted Our Validators Every Two Hours](https://happygopher.nl/writing/lighthouse-web3signer-scrypt/)
- [What Actually Moved the Score: A Rust Auth Server on highload.fun](https://happygopher.nl/writing/highload-fun-auth-server/)
- [TCP_NODELAY: One Line, 135x Faster in KurrentDB's Rust Client](https://happygopher.nl/writing/kurrentdb-rust-nagle/)
- [Why VS Code can't install extensions to a remote when your local ones are read-only](https://happygopher.nl/writing/vscode-remote-readonly-extensions/)
- [Building Financial Infrastructure That Must Not Fail](https://happygopher.nl/writing/building-financial-infrastructure-that-must-not-fail/)

---

## 🎯 What I'm looking for

Open to **Senior / Staff+** backend & distributed-systems roles — reliability, architecture, and scale. **Remote-only**, hands-on IC, **no people management**.

I keep complex business domains safe to change and easy to reason about, and raise the engineering bar by proof, not mandate.

---

## 📫 Contact

<a href="https://linkedin.com/in/maksim-shcherbo"><img src="https://custom-icon-badges.demolab.com/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin-white&logoColor=fff" alt="LinkedIn profile"></a> <a href="mailto:max@happygopher.nl"><img src="https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white" alt="Email"></a> <a href="https://happygopher.nl"><img src="https://img.shields.io/badge/Site-happygopher.nl-1E293B?style=flat-square&logo=astro&logoColor=white" alt="happygopher.nl"></a>

---

<p align="center">
  <a href="https://go.dev"><img alt="Go" src="https://img.shields.io/badge/Go-1E293B?style=flat-square&logo=go&logoColor=white&labelColor=1E293B"></a>
  <a href="https://www.rust-lang.org/"><img alt="Rust" src="https://img.shields.io/badge/Rust-1E293B?style=flat-square&logo=rust&logoColor=white&labelColor=1E293B"></a>
  <a href="https://aws.amazon.com"><img alt="AWS" src="https://img.shields.io/badge/Cloud-Amazon_AWS-1E293B?style=flat-square&logo=amazon-aws&logoColor=white&labelColor=1E293B"></a>
  <a href="https://kubernetes.io"><img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-1E293B?style=flat-square&logo=kubernetes&logoColor=white&labelColor=1E293B"></a>
  <a href="https://www.docker.com"><img alt="Docker" src="https://img.shields.io/badge/Docker-1E293B?style=flat-square&logo=docker&logoColor=white&labelColor=1E293B"></a>
  <a href="https://ethereum.org"><img alt="Ethereum" src="https://img.shields.io/badge/Ethereum-1E293B?style=flat-square&logo=ethereum&logoColor=white&labelColor=1E293B"></a>
</p>
