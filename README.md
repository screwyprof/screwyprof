<!-- Maksim Shcherbo GitHub profile (screwyprof) -->
# Maksim Shcherbo — Principal Engineer | Reliability Architect | Distributed Systems Expert

*(GitHub handle: [@screwyprof](https://github.com/screwyprof))*

**Specialties:** distributed systems, reliability engineering, system scalability, blockchain infrastructure, and fault-tolerant architecture.

### “I stop outages before they happen”

> Senior distributed systems and reliability engineer with 20+ years of experience building resilient platforms in finance, blockchain, and high-traffic e-commerce.

<!--
<div align="center">
<img src="social-preview.png" width="100%" alt="Maksim Shcherbo — Reliability Engineering & Distributed Systems">
</div>
-->

---

## 🧠 About Me

> I eliminate the million-dollar risks that keep CTOs awake at 3 AM: system failures that lose revenue, infrastructure that can't scale, and teams that can't deliver reliably.

I build platforms that recover invisibly — turning fragile systems into resilient infrastructure that protects revenue, reputation, and business continuity. My expertise spans both technical leadership and hands-on implementation, with 20+ years of systematic approaches to reliability that scale across financial systems, e-commerce platforms, and institutional-grade blockchain technology.

---

## 📈 Key Transformations

**Consistent pattern of turning fragile systems into resilient infrastructure:**

- ✅ **ConsenSys Staking platform ($2+ billion assets)** → Built zero-failure backend with institutional-grade safeguards
- ✅ **E-commerce marketplace** → Migrated monolith to microservices at Southeast Asia's #1 platform, built backend services handling **16,000+ RPS**
- ✅ **Regulated trading system** → Transformed chaotic operations into **99.99% uptime** platform while leading teams through complete system modernization
- ✅ **Distributed RNG protocol** → Pioneered on-chain random number generation using cryptographic protocol with `El-Gamal` encryption and `Tendermint` validators.

<details>
<summary>📊 Key metrics summary</summary>

| Area | Metric | Description |
|-------|---------|-------------|
| Reliability | 99.99% uptime | Transformed chaotic system to HA platform |
| Scale | 16,000+ RPS | Designed backend services at major e-commerce platform |
| Assets | $2+ billion | Safeguarded assets on ConsenSys staking backend |
</details>

---

## 🧩 Open Source (Public)

### Ethereum Ecosystem Contributions

#### [Lighthouse Consensus Client](https://github.com/sigp/lighthouse)

Resolved critical `OOM` failures causing validator restarts **every 2 hours**, blocking institutional upgrades. Root cause analysis revealed architectural flaw in validator management code - `O(n)` decryption of all keys for single-validator updates. I **transformed O(n) to O(1)** behavior, boosting attestation success rates to **~99%** and unblocking scale operations for tier-1 staking providers. [Problem analysis](https://github.com/sigp/lighthouse/issues/4936) and [solution](https://github.com/sigp/lighthouse/pull/4126).

#### [MEV-Boost](https://github.com/flashbots/mev-boost)

Solved fundamental scaling limitation preventing multi-client operations. Introduced **per-validator relay architecture**, delivering **~26% latency improvement** with `OpenTelemetry` instrumentation. [Proposal](https://github.com/flashbots/mev-boost/issues/455) and [solution](https://github.com/flashbots/mev-boost/pull/470).

#### [Autonity Consensus Layer](https://github.com/autonity/autonity)

Integrated Tendermint Proof-of-Stake consensus into go-ethereum before Ethereum's PoS transition. Contributed to consensus engine architecture for decentralized risk markets with delegated PoS and 1-second block times.

---

## 🏢 Enterprise Systems (Proprietary)

### [Ethereum Staking at ConsenSys](https://consensys.io/staking)

Built backend infrastructure for self-custodial Ethereum Staking powering MetaMask and institutional clients — **$2+ billion** in assets across **33,000+ validators**. [Documentation](https://docs.staking.consensys.io/staking-help) and [API reference](https://docs.staking.consensys.io/docs/staking-api).

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

<!--
## 🎤 Talks & Writing

> Coming soon: Reliability engineering deep-dives, architecture walkthroughs, and blockchain consensus insights.

---
-->

## 🎯 What I'm Looking For

**Principal/Staff Engineer roles** tackling complex architecture while mentoring teams.  

**Technical Leadership positions** bridging technical and organizational challenges.  

**Systems architecture challenges** in distributed, financial, or blockchain systems.  

**Available for consulting and fractional CTO engagements.**


---

## 📫 Contact

<a href="https://linkedin.com/in/maksim-shcherbo"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn profile"></a> <a href="mailto:max@happygopher.nl"><img src="https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white" alt="Email"></a>

Let's discuss how I can help transform your critical systems into reliable, scalable infrastructure.


---

![Visitors](https://komarev.com/ghpvc/?username=screwyprof&color=blue) <a href="https://golang.org"><img src="https://img.shields.io/badge/Go-00ADD8?style=flat-square&logo=go" alt="Go programming language"></a> <a href="https://www.rust-lang.org"><img src="https://img.shields.io/badge/Rust-000000?style=flat-square&logo=rust" alt="Rust programming language"></a> <a href="https://ethereum.org"><img src="https://img.shields.io/badge/Ethereum-3C3C3D?style=flat-square&logo=ethereum" alt="Ethereum blockchain"></a> <a href="https://aws.amazon.com"><img src="https://img.shields.io/badge/AWS-FF9900?style=flat-square&logo=amazon-aws" alt="AWS cloud"></a> <a href="https://kubernetes.io"><img src="https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes" alt="Kubernetes container orchestration"></a> <a href="https://www.docker.com"><img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker" alt="Docker containerization"></a>

<picture>
  <source
    srcset="https://github-readme-stats.vercel.app/api/top-langs?username=screwyprof&show_icons=true&layout=compact&theme=dark"
    media="(prefers-color-scheme: dark)"
  />
  <source
    srcset="https://github-readme-stats.vercel.app/api/top-langs?username=screwyprof&show_icons=true&layout=compact"
    media="(prefers-color-scheme: light), (prefers-color-scheme: no-preference)"
  />
  <img src="https://github-readme-stats.vercel.app/api/top-langs?username=screwyprof&show_icons=true&layout=compact" alt="Top languages"/>
</picture>


<!-- SEO keywords: Golang, Rust, Ethereum, Reliability Engineering, Distributed Systems, Blockchain, DDD, CQRS, AWS, Kubernetes, Docker, CI/CD, PostgreSQL, gRPC, OpenAPI, REST, Scalability, System Architecture, Technical Leadership, Principal Engineer, Staff Engineer -->
