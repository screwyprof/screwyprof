<!---
title: 'A Password Hash on Every API Call: The Bug That Restarted Our Validators Every Two Hours'
seoTitle: 'A Password Hash on Every API Call Restarted Our Validators'
description: 'Updating validators via Lighthouse restarted our pods every two hours. The cause: scrypt hashing on every update. A 20-line guard, merged upstream in v4.1.0.'
date: 2026-07-06
tags: [debugging, performance, ethereum]
linkedin: 'https://www.linkedin.com/posts/maksim-shcherbo_reliability-ethereum-rust-share-7480964307736973312-z_ME/'
related:
  - building-financial-infrastructure-that-must-not-fail
  - kurrentdb-rust-nagle
  - highload-fun-auth-server
--->
# A Password Hash on Every API Call: The Bug That Restarted Our Validators Every Two Hours

> 📍 **Canonical version: [happygopher.nl/writing/lighthouse-web3signer-scrypt](https://happygopher.nl/writing/lighthouse-web3signer-scrypt/).** This copy is kept for existing links.

On ConsenSys's Ethereum staking infrastructure, a Lighthouse upgrade kept failing pre-production testing: the validator clients restarted roughly **every two hours**. Lighthouse is the Ethereum consensus client, and its validator client is the process that keeps validators signing and earning. The trigger was a batch job that toggled builder proposals by sending one `PATCH /lighthouse/validators/{pubkey}` per validator. Under that load the process spiked CPU, stopped answering its `/metrics` liveness probe inside the five-second timeout, and Kubernetes restarted the pod. With thousands of keys the restarts were relentless, and the upgrade stayed blocked from production.

**TL;DR.** Updating one validator over the HTTP API decrypted Lighthouse's on-disk key cache, and that decryption runs `scrypt`, a password-hashing function built to be slow and memory-hungry on purpose. These validators signed through a remote signer and had no local keys, so the work was pure waste, but the code decrypted before checking whether it had anything to decrypt. Many of those calls in parallel spiked CPU and memory until the liveness probe timed out. The fix is a guard: skip the decryption when there are no local keys. It merged into Lighthouse `v4.1.0`.

## A fix that shipped, and didn't hold

The first version of this was reported months earlier: a single `PATCH` re-initialized every validator on the node, not just the one being patched. A maintainer traced it to a missing check. Local validators had a fast path that skipped re-initialization when the validator was already loaded, and the remote-signer branch had no equivalent, so it rebuilt all of them on every call. A fix added the missing check and shipped in `v3.4.0`, and the issue was closed.

It did not hold. On `v3.4.0` the restarts came back: the batch job still spiked CPU, the metrics probe still timed out, and Kubernetes still recycled the pod every couple of hours. Whatever the real cost was, the re-initialization check had not removed it.

## The path that looked free

When the performance issue was reopened, the leading theory was reasonable, and it came from people who know the codebase far better than I do: the update path decrypts a key cache, but for an all-remote-signer setup that decryption should be close to a no-op, because there are no local keys to decrypt. The logs seemed to agree. They showed `Key cache not modified` on every request, which reads like the cache work was being skipped.

So I traced the update path instead of reasoning about it. The `v3.4.0` fix had removed the redundant re-initialization, but the key-cache decryption sat on a separate line, above it and independent of it, and that line was not being skipped. The `Key cache not modified` log only reported that the cache had nothing new to write, near the end. The decryption ran much earlier, unconditionally, before anything checked whether a single local key existed:

```rust
let mut key_cache = self.decrypt_key_cache(cache, &mut key_stores).await?;
```

And `decrypt_key_cache` calls `scrypt`. That is the whole bug. `scrypt` is a key-derivation function engineered to be slow and memory-hard on purpose, so that guessing the password on an Ethereum keystore is prohibitively expensive. That is exactly what you want protecting a key at rest, and exactly what you never want on the hot path of an API. For a remote-signer node it guards nothing, because there are no local keys, and it ran on every `PATCH` anyway. One call was survivable. The batch job made many at once, and because `scrypt` is deliberately memory-hard, the parallel calls spiked memory as well as CPU, enough to starve the process past the five-second probe window. In a profile it would not even look suspicious: it shows up as legitimate crypto time, a function that is supposed to be slow being slow.

## The fix

Once the cause is named, the fix is boring, which is how you know it is the right one. Check whether any local keys exist, and only decrypt when they do. The core is one guard:

```rust
// Check if there is at least one local definition.
let has_local_definitions = self.definitions.as_slice().iter().any(|def| {
    matches!(def.signing_definition, SigningDefinition::LocalKeystore { .. })
});

// Decrypting the cache calls scrypt and is very expensive; it is never
// used for web3signer. Skip it entirely when nothing local needs it.
let mut key_cache = if has_local_definitions {
    self.decrypt_key_cache(cache, &mut key_stores).await?
} else {
    KeyCache::new()
};
```

The same guard wraps the two other places that touched the cache. For a pure remote-signer node the whole `scrypt` path is skipped, which is what everyone had assumed it already did. Twenty-odd lines, one file. Another operator on the thread had already suspected it was costing them missed attestations. It merged into `v4.1.0`, and the maintainer's review was the nicest kind of anticlimax: _"Fantastic, thank you for doing the work to dig into this!"_

## What it took

Nothing here was exotic. It was `scrypt` doing exactly its job, one layer from where it belonged, on a path that a fast-path check and a reassuring log line both made look free. The reported symptom, Kubernetes restarts, was two hops from the cause, and a shipped fix had already closed the issue once. Getting to the real one meant not trusting that a closed issue was closed, and not trusting that a path everyone had reasoned about matched the path the code actually ran. When a hot path is "obviously" free, that is the one to trace by hand.

## References

- [Lighthouse issue #3795](https://github.com/sigp/lighthouse/issues/3795): the original report, a `PATCH` re-initializing the whole validator set
- [Lighthouse issue #3968](https://github.com/sigp/lighthouse/issues/3968): the reopened performance issue this fix resolved
- [Lighthouse PR #4126](https://github.com/sigp/lighthouse/pull/4126): the fix, merged into `v4.1.0`
