<!-- Maksim Shcherbo GitHub profile (screwyprof) -->
<!-- Maksim Shcherbo github -->
<!-- Summary: Software Engineer specializing in backend & distributed systems, reliability, clean architecture, and Ethereum infrastructure. -->
# Maksim Shcherbo — Software Engineer · Backend & Distributed Systems

Backend & distributed-systems engineer building high-scale, fault-tolerant systems in **Go** and **Rust** — reliability, scalability, and Ethereum staking infrastructure.

*(also known as [@screwyprof](https://github.com/screwyprof))*

---

[![Go](https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev)
[![Rust](https://img.shields.io/badge/Rust-D34516?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?style=flat-square&logo=amazon-aws&logoColor=white)](https://aws.amazon.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Docker](https://img.shields.io/badge/Docker-257bd6?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com)
[![gRPC](https://img.shields.io/badge/gRPC-protocol-blue?style=flat-square&logo=grpc)](https://grpc.io)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-blue?style=flat-square&logo=opentelemetry)](https://opentelemetry.io/)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-6BA539?style=flat-square&logo=openapiinitiative&logoColor=white)](https://www.openapis.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?style=flat-square&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Ethereum](https://img.shields.io/badge/Ethereum-3C3C3D?style=flat-square&logo=ethereum&logoColor=white)](https://ethereum.org)

---

### “I stop outages before they happen”

> I turn fragile architectures into resilient systems that protect revenue, reputation, and customer trust.

---

## 🧠 About Me

> I eliminate the million-dollar risks that keep CTOs awake at 3 AM: system failures that lose revenue, infrastructure that can't scale, and teams that can't deliver reliably.

I make reliability systematic — not reactive. I've turned fragile financial, e-commerce, and blockchain backends into systems that scale seamlessly and recover automatically — pairing deep engineering discipline with clean, simple design.

---

## 🧩 Open Source

### Ethereum Ecosystem Contributions

#### [Lighthouse Consensus Client](https://github.com/sigp/lighthouse)

Resolved critical `OOM` failures causing validator restarts **every 2 hours**, blocking institutional upgrades. Root cause analysis revealed architectural flaw in validator management code - `O(n)` decryption of all keys for single-validator updates. I **transformed O(n) to O(1)** behavior, boosting attestation success rates to **~99%** and unblocking scale operations for tier-1 staking providers. [Problem analysis](https://github.com/sigp/lighthouse/issues/4936) and [solution](https://github.com/sigp/lighthouse/pull/4126).

#### [MEV-Boost](https://github.com/flashbots/mev-boost)

Solved fundamental scaling limitation preventing multi-client operations. Introduced **per-validator relay architecture**, delivering **~26% latency improvement** with `OpenTelemetry` instrumentation. [Proposal](https://github.com/flashbots/mev-boost/issues/455) and [solution](https://github.com/flashbots/mev-boost/pull/470).

#### [Autonity Consensus Layer](https://github.com/autonity/autonity)

Integrated Tendermint Proof-of-Stake consensus into go-ethereum before Ethereum's PoS transition. Contributed to consensus engine architecture for decentralized risk markets with delegated PoS and 1-second block times.

### Databases & Event Sourcing

#### [KurrentDB Rust Client](https://github.com/kurrent-io/KurrentDB-Client-Rust)

Diagnosed a **~40ms latency penalty on every gRPC call** in the official Rust client (formerly EventStoreDB), silently slowing every read, append, and subscription on low-latency networks. Root cause: the transport never enabled `TCP_NODELAY`, leaving Nagle's algorithm and TCP delayed `ACK` to stall each request behind a kernel timer. My **one-line fix** dropped the median read from **42ms to 311µs (~135x)** and was merged upstream. [Problem analysis](https://github.com/kurrent-io/KurrentDB-Client-Rust/issues/232), [solution](https://github.com/kurrent-io/KurrentDB-Client-Rust/pull/233), and [write-up](https://github.com/screwyprof/screwyprof/blob/main/articles/kurrentdb-rust-nagle.md).

---

## 💼 Experience & Case Studies

### [Ethereum Staking at ConsenSys](https://consensys.io/staking)

Built backend infrastructure for self-custodial Ethereum Staking powering MetaMask and institutional clients — **$2+ billion** in assets across **33,000+ validators**. [Documentation](https://docs.staking.consensys.io/staking-help) and [API reference](https://docs.staking.consensys.io/docs/staking-api). Deep dive: [Building Financial Infrastructure That Must Not Fail](./articles/building-financial-infrastructure-that-must-not-fail.md).

### [Lazada (Alibaba Group) — Southeast Asia’s leading E-commerce marketplace](https://www.lazada.com/en/)

Migrated the product-catalog domain from a legacy PHP monolith to distributed Go microservices handling **16k+ RPS per instance** across six markets.
Advocated Clean Architecture and TDD practices across teams, improving reliability and maintainability at scale.

### [Competitive Auction House — Regulated Financial Platform](https://www.a-k-d.ru)

Transformed a fragile, monolithic trading system into an event-driven, **99.9%** uptime platform using **DDD, CQRS, and Event Sourcing**.
Introduced CI/CD pipelines and structured on-call processes, establishing reliability standards for the engineering organization.

### Distributed Randomness (R&D, DAO.Casino)

Pioneered on-chain random number generation — a cryptographic protocol using `El-Gamal` encryption and `Tendermint` validators.

---

## 💎 Notable Projects

*Personal projects and experiments showcasing systematic approaches to reliability, architecture, and testing patterns.*

### [delegator](https://github.com/screwyprof/delegator)

High-performance Tezos delegation service with CQRS architecture achieving **15s** full sync of **760k+** delegations and **<1ms** API responses. Demonstrates reliability patterns with **92% test coverage**.

### [payment](https://github.com/screwyprof/payment)

Complete `DDD/CQRS/Clean Architecture` implementation preventing costly financial errors. Features event-sourced aggregates, invariant enforcement, and full test coverage.

### [favkit](https://github.com/screwyprof/favkit)

Rust CLI tool for managing Finder favorites on macOS with ADR-based architectural reasoning and nix reproducibility.

### [cqrs](https://github.com/screwyprof/cqrs)

Event-sourced CQRS library featuring Given/When/Then DSL for business-readable domain specifications — a rare pattern in Go.

### [form3api](https://github.com/screwyprof/form3api)

Form3 API client with GitHub-style HATEOAS pagination and executable documentation as acceptance tests.

---

## 🎤 Writing & Talks

Currently writing about reliability, architecture, and building systems that must not fail.  
More to come — deep dives into scaling, process design, and the invisible layers of infrastructure.

📘 **Articles**
- [A Password Hash on Every API Call: The Bug That Restarted Our Validators Every Two Hours](./articles/lighthouse-web3signer-scrypt.md)
- [What Actually Moved the Score: A Rust Auth Server on highload.fun](./articles/highload-fun-auth-server.md)
- [TCP_NODELAY: One Line, 135x Faster in KurrentDB's Rust Client](./articles/kurrentdb-rust-nagle.md)
- [Why VS Code can't install extensions to a remote when your local ones are read-only](./articles/vscode-remote-readonly-extensions.md)
- [Building Financial Infrastructure That Must Not Fail](./articles/building-financial-infrastructure-that-must-not-fail.md)

💬 **Join the discussion on LinkedIn:** 
- [What Actually Moved the Score: A Rust Auth Server on highload.fun](https://www.linkedin.com/posts/maksim-shcherbo_rust-highload-performance-share-7479786319825600513-Z4W9/)
- [TCP_NODELAY: One Line, 135x Faster in KurrentDB's Rust Client](https://www.linkedin.com/posts/maksim-shcherbo_rust-grpc-eventsourcing-share-7475345091214258176-2qN1/)
- [Why VS Code can't install extensions to a remote when your local ones are read-only](https://www.linkedin.com/posts/maksim-shcherbo_softwareenginerring-vscode-nix-share-7474974533335101440-vH5q/)
- [Building Infrastructure That Must Not Fail](https://www.linkedin.com/pulse/building-infrastructure-must-fail-maksim-shcherbo-1d4se/)

---

## 🎯 What I'm Looking For

Open to **Senior+ / Staff / Principal** backend & distributed-systems roles — focused on reliability, architecture, and scale. **Remote-only**, hands-on IC, **no people management**.

I set technical direction through standards, RFCs/ADRs, and quality gates — as a hands-on engineer, not a manager — so systems and practices meet a high bar by default.

My passion is building infrastructure that must not fail and cultivating the engineering discipline that makes that possible.  

If you're scaling critical systems or modernizing legacy infrastructure, I'd love to discuss how I can help.

---

## 📫 Contact

<a href="https://linkedin.com/in/maksim-shcherbo"><img src="https://custom-icon-badges.demolab.com/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin-white&logoColor=fff" alt="LinkedIn profile"></a> <a href="mailto:max@happygopher.nl"><img src="https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white" alt="Email"></a>

Let's discuss how I can help transform your critical systems into reliable, scalable infrastructure.


---

<!--
<p align="center">
  <img src="https://komarev.com/ghpvc/?username=screwyprof&color=gray" alt="Profile views">
</p>
-->

<p align="center">
  <a href="https://go.dev"><img alt="Go" src="https://img.shields.io/badge/Go-1E293B?style=flat-square&logo=go&logoColor=white&labelColor=1E293B"></a>
  <a href="https://www.rust-lang.org/"><img alt="Rust" src="https://img.shields.io/badge/Rust-1E293B?style=flat-square&logo=rust&logoColor=white&labelColor=1E293B"></a>
  <a href="https://ethereum.org"><img alt="Ethereum" src="https://img.shields.io/badge/Ethereum-1E293B?style=flat-square&logo=ethereum&logoColor=white&labelColor=1E293B"></a>
  <a href="https://aws.amazon.com"><img alt="AWS" src="https://img.shields.io/badge/Cloud-Amazon_AWS-1E293B?style=flat-square&logo=amazon-aws&logoColor=white&labelColor=1E293B"></a>
  <a href="https://kubernetes.io"><img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-1E293B?style=flat-square&logo=kubernetes&logoColor=white&labelColor=1E293B"></a>
  <a href="https://www.docker.com"><img alt="Docker" src="https://img.shields.io/badge/Docker-1E293B?style=flat-square&logo=docker&logoColor=white&labelColor=1E293B"></a>
</p>

<!-- SEO keywords: Backend Engineer, Distributed Systems, Go, Golang, Rust, Ethereum, Reliability Engineering, Blockchain, DDD, CQRS, Event Sourcing, AWS, Kubernetes, Docker, CI/CD, PostgreSQL, gRPC, OpenAPI, REST, Scalability, System Architecture, Principal Software Engineer, Staff Software Engineer, Senior Software Engineer -->
