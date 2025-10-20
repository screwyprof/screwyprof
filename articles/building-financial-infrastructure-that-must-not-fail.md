<!--
---
title: "Building Financial Infrastructure That Must Not Fail"
author: "Maksim Shcherbo"
description: "Lessons from building institutional-grade reliability in blockchain and financial systems — entirely from public information and general principles."
tags: [Reliability, Blockchain, Ethereum, Infrastructure, Engineering, Case Study]
---
-->

# Building Financial Infrastructure That Must Not Fail

> I build financial and blockchain infrastructure that **must not fail** — identifying and preventing the catastrophic patterns that cause million-dollar losses, and creating the missing capabilities institutional operators need.

I've built systems handling **billions in assets** without failure. During my time at [ConsenSys](https://consensys.io), I contributed fixes to [Lighthouse](https://github.com/sigp/lighthouse) and enhancements to [MEV-Boost](https://github.com/flashbots/mev-boost) that demonstrate this expertise through **publicly verifiable code**.

## Why It Matters

In this retrospective, you'll discover how I bridge traditional finance to Ethereum — the expertise required for any organization building institutional-grade digital asset infrastructure.

- **Preventing million-dollar losses in institutional systems** - From regulated trading platforms to billion-dollar e-commerce using proven failure-prevention patterns

- **Bridging Traditional Finance to Ethereum at scale** - $2B+ infrastructure managing 33,000+ validators with institutional-grade safeguards

- **Building what's missing when tools hit institutional limits** - Lighthouse fixes and MEV-Boost enhancements with publicly verifiable impact

I've built financial infrastructure in both worlds - traditional finance and blockchain - with the specific expertise to bridge institutional-grade patterns to on-chain operations.

## Many Ways to Lose Millions And How I Prevented Each

Through building high-stakes systems across different domains, I've encountered specific ways that millions can disappear instantly. I once witnessed someone accidentally remove `Akamai's CDN Cache` with a misclick - **$20,000** gone in seconds. In institutional finance, similar human errors can mean millions. Systems must be ready.

Each pattern below stopped a real loss from happening. These examples demonstrate patterns essential to any institutional-grade financial infrastructure.

### Duplicate Operations from Failed Requests

#### Business Problem

Imagine submitting a **$10M** procurement bid for nuclear sector equipment when the system freezes. Click again and you might commit to **$20M**. Every financial system faces this risk, but the consequences vary from angry customers to regulatory violations.

#### How I Solved It

Operations that can't be safely repeated need [`idempotency`](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent) - a guarantee that multiple identical requests produce the same response.

I've implemented this pattern across multiple financial systems. While the core concept remains constant, every implementation differs based on business needs and technical constraints. Major payment providers like [Stripe](https://stripe.com/docs/api/idempotent_requests) and [PayPal](https://developer.paypal.com/api/rest/reference/idempotency/) each implement their own variations, adapting the pattern to their specific requirements.

In my implementations, I typically use a unique idempotency key per request, following the `Idempotency-Key` header from the [IETF draft](https://www.ietf.org/archive/id/draft-ietf-httpapi-idempotency-key-header-01.html):

```
Request arrives → Check idempotency key
  ├─ Found & Complete → Return cached response
  ├─ Found & In Progress → 409 Conflict
  ├─ Found but payload changed → 422 Error
  └─ Not Found → Process new request
```

Failed operations aren't cached, allowing clients to retry with the same key - crucial for recovering from transient failures.

While idempotency prevents duplicate operations, there's another way millions disappear: unauthorized data access.

### Access Controls & Data Isolation

#### Business Problem

In financial systems, seeing another client's data ends everything. One leak = lawsuits, regulatory penalties, and permanent reputation damage. Whether it's competitors' bids in regulated procurement systems, customer financial data in e-commerce platforms, or validator keys in staking platforms, data isolation isn't optional - it's fundamental.

#### How I Solved It

Multi-tenant systems require both access control (who can do what) and data isolation (keeping client data separate). A single bug could expose sensitive data across tenant boundaries.

##### Access Control

Every system implements differently, but I've worked with various patterns:

- Some use third-party identity services, others build in-house due to regulatory constraints
- Authentication varies: JWTs, session IDs, or proprietary tokens
- Different layers handle different concerns:
  - Gateway: Token transformation (session ID → JWT), request routing
  - Application: Token validation, fetching permissions/roles, then authorization decisions
  - Database: Final enforcement through Row-Level Security (PostgreSQL/Oracle/SQL Server) or application-enforced tenant filtering
- Each layer provides independent protection

##### Data Isolation Strategies

- Infrastructure level: Separate databases in different regions for different clients
- Storage level: Encrypted S3 buckets with tenant-specific access policies
- Query level: True database-level isolation (RLS) where available, otherwise application-enforced tenant filtering

Multiple layers mean if one fails, others still protect the data - essential for institutional trust.

This multi-layered approach has protected sensitive data across different systems and financial platforms. Protection requires proof - audit trails resolve disputes when millions are at stake.

### Audit Trails for million-dollar Disputes

#### Business Problem

When million-dollar transactions go wrong, "he said, she said" isn't acceptable. Customers claim they never authorized trades. Auditors demand transaction histories. Regulators require proof of compliance. Without immutable audit trails, you're defenseless against disputes that can destroy reputations and trigger massive penalties.

#### How I Solved It

Audit trails must balance completeness with privacy - capturing everything needed for disputes while protecting sensitive information.

I've implemented audit trails using various approaches:

- Writing to a dedicated database audit table enforcing updates/inserts via triggers
- Using cloud storage providers (S3, Azure Blob) with object lock/retention policies
- CQRS/Event Sourcing for complex flows requiring full state reconstruction

What matters is capturing who did what, when, and why - with enough context to resolve disputes months later - and ideally ensuring it can't be altered after the fact:

```json
{
  "trace_id": "f47ac10b-58cc-4372",
  "timestamp": "2023-11-15T10:32:41.837Z",
  "user_id": "7842",
  "action": "bid.submit", 
  "resource": "auction:nuclear-reactor-parts-Q4",
  "amount": 10500000,
  "currency": "USD",
  "result": "success"
}
```

These three patterns - idempotency, access control, and audit trails - became even more essential at `ConsenSys Staking` — each directly preventing the kinds of silent failures that can destroy trust in financial infrastructure.

## The Stakes

> All details below are based on publicly available documentation from ConsenSys Staking.

[ConsenSys Staking](https://consensys.io/staking) provides self-custodial staking on [Ethereum](https://ethereum.foundation/ethereum) - clients retain control of their keys while ConsenSys manages the technical infrastructure. The platform powers staking for [MetaMask](https://portfolio.metamask.io/stake)'s millions of users and institutional clients, operating **33,000+** validators managing **$2+ billion** in assets.

On `Ethereum`, each validator requires exactly **32 ETH** locked to participate in network validation, earning rewards but facing penalties for downtime. **"Slashing"** (permanent loss of staked `ETH` for protocol violations) is the ultimate risk. Unlike traditional finance with its settlement periods and dispute resolution, every staking operation is cryptographically final.

Based on publicly available information from [ConsenSys](https://consensys.io) and [ConsenSys Staking](https://consensys.io/staking):

```ascii-art
┌────────────────────────────────────────────────────────────────────┐
│              CONSENSYS STAKING AT INSTITUTIONAL SCALE              │
│                 (Public metrics from consensys.io)                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Scale & Performance:                    Trust & Security:         │
│  • 33,000+ validators                    • Non-custodial design    │
│  • $2 Billion+ staked ETH                • SOC2 Type II certified  │
│  • 99.9%+ participation rate             • Multi-region infra      │
│  • 0 slashing incidents                  • 24/7 monitoring         │
│                                                                    │
│  The Engineering Challenge:                                        │
│                                                                    │
│  Build APIs that institutional clients trust while ensuring        │
│  every operation translates correctly to irreversible blockchain   │
│  transactions. One bug = permanent loss of client assets.          │
└────────────────────────────────────────────────────────────────────┘
```

## From API to Blockchain

During my time on the `ConsenSys Staking` team, I applied institutional-grade expertise from many years of building financial systems. While I cannot discuss internal implementation details, the publicly available documentation reveals some patterns worth examining:

The [Staking Help Center](https://docs.staking.consensys.io) provides:

- [Staking Guide](https://docs.staking.consensys.io/staking-help) - general usage guide
- [Programmatic Usage](https://docs.staking.consensys.io/programmatic-usage) - how to use the `Staking API`
- [API Documentation](https://docs.staking.consensys.io/docs/staking-api) - a rendered `OpenAPI Schema`

Together, these resources shed light on how this black box transforms HTTP requests into irreversible blockchain operations.

### Staking API Primary Endpoints

The documentation describes two primary endpoints:

#### `POST /stake` - Initiating Staking

This [endpoint](https://docs.staking.consensys.io/docs/staking-api#tag/Staking/operation/reserveStakesV2) initiates staking by **idempotently** reserving validators and returns the payload required for sending the transaction to the **deposit contract**.

```yaml
POST /stake
Headers:
  Authorization: Bearer <access_token> # an encoded JWT
  Idempotency-Key: f1c971b4-7114-4c4b-8ad2-a6f9e33e2cd0 # A UUID
Body:
  withdrawal_pubkey: # BLS Public Key or an Ethereum Address
  amount: 32000000000  # 32 ETH in Gwei
Returns: Deposit contract payload
```

```json
{
  "eth1_contract_address": "0x00000000219ab540356cBB839Cbe05303d7705Fa",
  "stakes": [{
    "pubkey": "0x93247...",
    "withdrawal_credentials": "0x010000...",
    "amount": "32000000000",
    "signature": "0x1b66ac...",
    "deposit_root": "0xdef456..."
  }]
}
```

Each object in the `stakes` array represents the ethereum **deposit data** structure.

#### `GET /voluntary-exit-message/{validatorID}` - Exiting Validators

This [endpoint](https://docs.staking.consensys.io/docs/staking-api#tag/Staking/operation/validatorVoluntaryExit) requests a **signed voluntary exit message**.

```yaml
GET /voluntary-exit-message/{validatorID}
Headers:
  Authorization: Bearer <access_token> # an encoded JWT
Path:
  ValidatorID: <bls_pubkey or validator_index>
Returns: Signed voluntary exit message
```

```json
{
  "exit_transaction": {
    "message": {
      "epoch": 0,
      "validator_index": 0
    },
    "signature": "0x2126140d542c..."
  },
  "fork_version": "0x00000001"
}
```

The response contains a **BLS-Signed Voluntary Exit Message**.

### Failure Prevention Patterns I Recognize

These public endpoints reveal failure prevention patterns essential for institutional-grade systems. Someone with infrastructure experience notices when examining this black box:

#### Idempotency Where It Matters

The `POST /stake` endpoint mandates an `Idempotency-Key` header - implementing the pattern for preventing duplicate financial operations. This is vital: accidentally creating duplicate validators means locking **64 ETH** instead of **32 ETH**.

This prevents the exact duplicate operation problem I've solved in regulated environments, where double-clicking could submit duplicate high-value transactions.

Notably, the `GET /voluntary-exit-message/{validatorID}` endpoint doesn't require an idempotency key. This follows [HTTP semantics](https://httpwg.org/specs/rfc9110.html) - `GET` is a [safe method](https://httpwg.org/specs/rfc9110.html#safe.methods) and therefore idempotent by definition (when properly implemented).

#### Granular Access Control

Error responses reveal sophisticated permission management:

- `401: unauthorized: missing authentication header`
- `403: forbidden: missing permission: read:validator`

These granular permissions (`read:validator`) implement the access control patterns essential for multi-tenant systems where different clients need different access levels. Such permissions often work hand-in-hand with data isolation - ensuring clients can only access their own validators.

#### Production-Grade Error Handling

The `API` demonstrates careful information management:

- Client errors (4xx): Specific, actionable messages
- Server errors (500): Sanitized to "please contact support"

This separation prevents internal state leakage while maintaining usability - exactly what institutional clients expect. In high-scale production environments, I've implemented similar error sanitization to protect internal system details - including backend services handling **16,000+** requests per second after all caching layers.

These patterns - idempotency, granular access control, and production-grade error handling - demonstrate institutional-grade design. What makes this API truly special is how it bridges financial infrastructure with Ethereum's blockchain protocol.

## Bridging Traditional Finance to Ethereum

With protocol-level understanding of `Ethereum` from years of study, I can recognize how the `Staking API` translates institutional patterns into blockchain operations.

### How the API Connects Two Worlds

The API inputs and outputs aren't arbitrary - they're `Ethereum` protocol data structures:

```ascii-art
┌────────────────────────────────────────────────────────────────────┐
│              ConsenSys Staking API → Ethereum Staking              │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│ POST /stake              →  Deposit Data   →  Deposit Contract     │
│ (Idempotency required)      (+ 32 ETH tx)     (0x00000000219ab...) │
│                                                                    │
│ GET /voluntary-exit      →  Exit Message   →  Beacon Chain         │
│ (Naturally idempotent)      (Signed VEM)      (Exit processing)    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Understanding Ethereum at Protocol Level

These patterns emerged from mastering the [consensus specs](https://github.com/ethereum/consensus-specs) before [The Merge](https://ethereum.org/en/roadmap/merge/) in September 2022. At this transition, the [Beacon Chain](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md) became `Ethereum`'s current consensus layer, replacing `Proof of Work` with `Proof of Stake`.

#### Deposit Flow

The API returns [DepositData](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#depositdata) - the exact structure required by the [Deposit Contract](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/deposit-contract.md) which is [deployed on mainnet](https://etherscan.io/address/0x00000000219ab540356cBB839Cbe05303d7705Fa). To activate a validator, users send a transaction with exactly **32 ETH** plus this `DepositData` to the contract. Each field (`pubkey`, `withdrawal_credentials`, `signature`, `deposit_root`) must match the protocol specification or the transaction fails.

#### Exit Flow

The API returns a [Signed](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#signedvoluntaryexit) [`VoluntaryExit`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#voluntaryexit) message containing `epoch` and `validator_index`. Broadcasting this to the beacon chain initiates the exit process, eventually returning staked ETH plus rewards to the withdrawal address.

## Improving Ethereum Ecosystem Tools

```ascii-art
┌──────────────────────────────────────────────────────────────────┐
│                      ETHEREUM STAKING TOOLS                      │
├──────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────┐   ┌───────────────────────────┐  │
│  │      Lighthouse            │──►│      MEV-Boost            │  │
│  │                            │   │                           │  │
│  │  Runs validators,          │   │  Connects validators to   │  │
│  │  proposes blocks,          │   │  specialized builders     │  │
│  │  attests to chain          │   │  for higher rewards       │  │
│  │                            │   │                           │  │
│  │  My fix: O(n)→O(1)         │   │  My enhancement:          │  │
│  │  → Unblocked upgrades      │   │  → Per-validator config   │  │
│  │  → Attestations are at 99% │   │  → 26% faster             │  │
│  └────────────────────────────┘   └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

These tools hit limits at institutional scale. I identified the bottlenecks and delivered solutions that kept thousands of validators running.

### Lighthouse Performance Regression

#### Business Problem

[Lighthouse](https://github.com/sigp/lighthouse) is an `Ethereum Consensus Client` written in `Rust` that runs validators - the entities that propose blocks and `attest` to the blockchain's state. It's the critical software that keeps validators online and earning rewards.

Testing [Lighthouse v3.4.0](https://github.com/sigp/lighthouse/releases/tag/v3.4.0) caught a serious regression before it could reach production: validator restarts every **2 hours**. Had this hit our production validators, we would have lost revenue immediately - thousands of validators missing attestations means hemorrhaging `ETH`. We caught it in testing, but the regression made immediate upgrading impossible for our configuration.

#### How I Solved It

The root cause lay in [Lighthouse's](https://github.com/sigp/lighthouse) validator management code. Every `PATCH /lighthouse/validators/{pubkey}` request attempted to decrypt the key cache for **ALL** validators - even [`web3signer`](https://github.com/ConsenSys/web3signer) validators (remote signing service) that store no local keys. With thousands of validators, this `O(n)` behavior caused `CPU` throttling and memory exhaustion. The `OOM killer` terminated the process, prompting `Kubernetes` to restart the service **every 2 hours**.

The [initial report](https://github.com/sigp/lighthouse/issues/3795#issuecomment-1422257924) sparked discussion but no resolution. With the upgrade blocked, I investigated the issue. Traditional debugging misled me: CPU profiling pointed to logging, but removing logger calls at hot paths didn't help. Memory profiling tools failed to capture the issue, though `top` showed gradual but steady memory growth. I documented these [failed attempts](https://github.com/sigp/lighthouse/issues/4936) in detail.

When the maintainer [couldn't reproduce it](https://github.com/sigp/lighthouse/issues/4936#issuecomment-1888483983) and suggested optimizations that didn't address the root cause, I analyzed the validator management code line by line. The architectural flaw was hidden in the validator management logic - every update unconditionally called `decrypt_key_cache()` on all validators.

[The Solution](https://github.com/sigp/lighthouse/pull/4126/files): check if local `keystores` exist before attempting decryption. For pure `web3signer` deployments, this transforms `O(n)` to `O(1)`. A 20-line fix that took months of persistence. The maintainer's response: ["Fantastic, thank you for doing the work to dig into this!"](https://github.com/sigp/lighthouse/pull/4126#pullrequestreview-1355774494)

This fix unblocked the upgrades and prevented what would have been significant revenue loss from validator downtime. Merged into `Lighthouse v4.1.0`, it now protects all large-scale `web3signer` deployments from the same issue — exactly the kind of failure-proofing that defines financial infrastructure that must not fail.

### MEV-Boost Per-Validator Configuration

#### Business Problem

[MEV-Boost](https://github.com/flashbots/mev-boost) is middleware written in `Golang` that enables validator clients to access specialized block builders who compete to create the most profitable blocks. It connects through "relays" - trusted intermediaries that aggregate blocks from multiple builders and ensure fair payment distribution.

At institutional scale, [a major operator found a fundamental limitation](https://github.com/flashbots/mev-boost/issues/154#issue-1274788549): no way to configure different `MEV sources` for different clients' validators. When asked for specifics, they [explained](https://github.com/flashbots/mev-boost/issues/154#issuecomment-1160393131): "One customer 'trusts' relays 3,4,5 as sources of MEV. Another customer 'trusts' relays 1,2,3." The problem? `MEV-Boost` applies the same relay configuration to all validators - no way to customize per client.

Imagine if a restaurant's ordering system forced the kitchen to use the same suppliers for every table. Some tables want only organic ingredients, others insist on local sources, but the system can't track preferences per table. The workaround? Run entirely separate kitchens - vastly more complex than simply noting "table 5 wants organic."

<img alt="MEV-Boost before - multiple instances needed" src="https://user-images.githubusercontent.com/1536274/220412386-d13661e3-81b4-4e3d-ae0a-99a505d045ad.png" height="250"/>

#### How I Solved It

`MEV-Boost` only accepts a global list of relay `URL`s at startup via `-relay` flag. No configuration mechanism exists to specify which validators should use which relays. Every validator connected through the same `MEV-Boost` instance must query all configured relays.

A workaround? Run separate consensus clients (like `Lighthouse`) each with its own `MEV-Boost` instance. Need validators using relay set `A` and others using relay set `B`? That's two `Lighthouse` instances, two `MEV-Boost` instances, double the resources, double the operational overhead.

I first proposed the solution in [issue #455](https://github.com/flashbots/mev-boost/issues/455), outlining how per-validator relay configuration could work:

<img alt="MEV-Boost after - single instance with per-validator config" src="https://user-images.githubusercontent.com/1536274/220412381-2e71c192-ef20-4c04-8beb-4a3f8e1fa7d9.png" height="250"/>

After months of discussion and refinement, I implemented the [complete solution](https://github.com/flashbots/mev-boost/pull/470/files). The architecture centers on two components: a `Relay Configuration Manager (RCM)` that holds validator-relay mappings, and a pluggable `Relay Configuration Provider (RCP)` that loads configs from files or APIs. Configuration updates happen without service restarts (using lock-free atomics for safety), and the optimized implementation achieved **26%** latency improvement.

After [months of collaboration](https://github.com/flashbots/mev-boost/issues/455), when upstream integration stalled, I maintained a production fork with additional performance improvements and `OpenTelemetry` instrumentation the original lacks. This pragmatic decision delivered immediate value to institutional clients while keeping the door open for future convergence — ensuring validators could operate reliably at scale instead of depending on brittle workarounds that invite failure.

## Proven at Scale

Preventing failures before they cost millions, recognizing what breaks at institutional scale, and building what's missing when it matters - this is how I approach every system. The code is publicly verifiable. The expertise is proven. 

These principles extend beyond staking — they apply to any financial or blockchain system where failure means real loss.

---

### About the Author

I'm a Principal Software Engineer who specializes in designing fault-tolerant, high-stakes infrastructure — systems that move billions without breaking.  
If you're building infrastructure that **must not fail**, I'd love to connect.

- [LinkedIn](https://linkedin.com/in/maksim-shcherbo)  
- [GitHub](https://github.com/screwyprof)
