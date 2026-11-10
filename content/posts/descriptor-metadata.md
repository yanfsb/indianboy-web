---
title: "Descriptor metadata"
description: "Solving the descriptor wallets stop-gap loading problems with an metadata approach"
publishedAt: "2026-10-22"
tags: ["bitcoin", "wallets", "encoding"]
author: "Yan Fernandes"
summary: "Solving the descriptor wallets stop-gap loading problems with an metadata approach"
---

## What is the problem?

Output script descriptors define how a wallet derives its SPKs (scriptPubKeys), but they don't necessarily tell us which derivation indexes have already been revealed or used. When recovering a wallet without its previous derivation state, a common approach is to derive addresses sequentially and scan for transaction activity until a predefined number of consecutive unused addresses is found.

This heuristic is known as the _gap limit_, and it introduces an important limitation: the wallet must rediscover relevant derivation indexes instead of restoring them directly.

### The merchant example

Let's think about a merchant using [BTCPay Server](https://btcpayserver.org/). Every time a customer creates an invoice, BTCPay Server generates a new Bitcoin address, whether the customer actually pays or not.

Now imagine this merchant has generated thousands of invoices, with many of them left unpaid. If they open the same wallet in an external application with a gap limit of 20, it might stop scanning after finding 20 consecutive unused addresses, even if there are payments at higher indexes.

This isn't just a hypothetical problem. [BTCPay Server documents this exact issue](https://docs.btcpayserver.org/FAQ/Wallet/) with external wallets:

> If you're using an existing software or a hardware wallet with your BTCPay Server, you may experience a discrepancy between balance in your BTCPay wallet and the external wallet's web, desktop or mobile app. This discrepancy is usually related to a gap-limit issue.

### Trade-offs and workarounds

One way to deal with this is to increase the gap limit. But how high should it be? 100? 1,000? 100,000? At some point, we're just making a bigger guess.

The larger the gap limit, the more addresses the wallet may need to derive and check. And even then, there's no guarantee it will find every address that has been used.

But now we run into another problem: scanning the blockchain can tell us whether a derived address has been used, but it can't tell us whether an unused address has already been **revealed** to someone.

In our merchant example, an unpaid invoice still has a revealed address associated with it. If the wallet doesn't recover its previously revealed state, it might end up revealing the same address to another customer, leading to address reuse. Even worse, if recovery stops before discovering an address that has received a payment, those funds might not appear in the recovered wallet.

### Full scan vs sync

There's an important distinction here: a full scan and a regular wallet sync aren't the same Pokémon.

When recovering a wallet without its previous derivation state, a full scan tries to discover which SPKs have historical activity. In BDK, it scans the already known revealed range (if any), then continues until the stop-gap is reached.

But what if we already know which SPKs have been revealed?

In that case, we can sync those SPKs directly, without relying on the stop-gap to discover them. However, this doesn't necessarily mean less work. A wallet with thousands of revealed indexes might still need to derive and check thousands of SPKs.

This brings us to another question: what if we could reconstruct the wallet's previous derivation state without having to materialize every intermediate SPK?

## What if we didn't have to rediscover it?

We've been asking the wallet to rediscover information that the original wallet already knew: which derivation indexes were **revealed**, which ones were **used**, and how far the wallet had progressed.

What if we could export that information as metadata alongside the descriptor?

Instead of relying entirely on the gap limit to infer the previous derivation state, we could restore it directly using explicit metadata.

### SPK Metadata

The idea is pretty simple. Instead of making the wallet rediscover its previous derivation state, what if we could store that information as metadata associated with each descriptor?

For this research, I'm exploring three pieces of information:

```text
last_revealed
revealed_spk[]
used_spk[]
```

`last_revealed` represents the highest derivation index reached by the wallet. `revealed_spk[]` contains the indexes that have been revealed, while `used_spk[]` contains the indexes with known historical activity.

Why keep them separate? Because a revealed address isn't necessarily a used address. Going back to our merchant example, an address associated with an unpaid invoice has already been revealed, even if it has never received a transaction.

Let's imagine the wallet has revealed 1,000 addresses, but only four of them have actually been used. We could represent its derivation state like this:

```text
last_revealed = 999
revealed_spk  = [0, 1, 2, ..., 999]
used_spk      = [4, 39, 250, 900]
```

Now, what would happen if we tried to recover this wallet using only its descriptor and a gap limit of 20, without any previously persisted derivation state?

The wallet would start deriving addresses from index 0, eventually finding a payment at index 4. But after checking another 20 consecutive unused addresses (indexes 5 through 24), it would stop. We wouldn't even reach index 39, let alone 250 or 900.

Apparently, index 900 doesn't exist anymore because the wallet got bored at index 24.

Of course, we could increase the gap limit, but we're back to the same question: how high should it be?

Now imagine we exported this metadata alongside the descriptor. We already know that the wallet has reached index 999, which indexes have been revealed, and which ones have known historical activity.

Instead of trying to rediscover those used indexes, we could reconstruct the wallet's previous derivation state and derive the relevant SPKs directly. We would also know where to continue revealing new addresses, without accidentally reusing previously revealed ones.

There's an important distinction here: knowing the derivation state doesn't mean we've recovered the transaction history. We still need to synchronize with the blockchain, and addresses that were unused when the metadata was exported might have received payments since then.

The goal isn't to replace synchronization, but to avoid unnecessary work when reconstructing what the wallet already knew.

## Encoding the metadata

There's one more problem with this approach: how do we store all these indexes without making the metadata too large?

In our previous example, we only had four used indexes. That's easy enough to store, but what happens when a wallet has thousands of revealed and used addresses? Storing every derivation index as a plain integer could make our metadata unnecessarily large.

Fortunately, both `revealed_spk[]` and `used_spk[]` are sorted sequences of integers. We can take advantage of that to represent them more efficiently.

One of the encoding techniques I've been exploring is **Elias–Fano**, which is designed to represent monotonically increasing integer sequences in a compact way, especially when the values are sparsely distributed.

### How does Elias–Fano work?

The idea behind Elias–Fano is pretty simple: instead of storing every integer individually, we can take advantage of the fact that our indexes are already sorted.

Elias–Fano splits each integer into two parts: **high bits** and **low bits**. The low bits are stored directly, while the high bits are represented using a bitvector.

Let's take the `used_spk[]` from our merchant example:

```text
used_spk = [4, 39, 250, 900]
```

We have four indexes, with values ranging from 0 to 999. For this example, we can use 7 bits for the lower part of each number, leaving the rest for the upper part.

```text
Index    High    Low
--------------------
4         0       4
39        0      39
250       1     122
900       7       4
```

Since we're using 7 low bits, each number is split using `2^7 = 128`:

```text
high = index / 128
low  = index % 128
```

Now we can store the four low values using 28 bits in total, and encode the high values in an 11-bit vector.

To construct the high bitvector, we place a `1` at each position `high[i] + i`:

```text
high      = [0, 0, 1, 7]
positions = [0, 1, 3, 10]

bitvector = 11010000001
```

That's **39 bits**, compared to **128 bits** if we simply stored four 32-bit integers.

Not bad for a list of numbers, right?

Of course, this is just the core representation. An actual serialized format needs additional information to decode it correctly. And a smaller representation doesn't automatically mean faster wallet recovery. That's something we'll need to measure.

### Is Elias–Fano really the best option?

Well, that's what we're trying to find out.

Elias–Fano looks promising, especially for sparse sequences, but that doesn't necessarily mean it's the best choice for every wallet.

For this research, we also want to explore a few alternatives:

- **Raw integers:** storing each derivation index directly, without compression.
- **Delta encoding with varints:** storing the differences between consecutive indexes instead of their absolute values.
- **Bitmaps:** representing whether each index is used with a single bit.
- **Elias–Fano:** encoding the sorted indexes, either as absolute values or as positions within the revealed set.

There's also an interesting detail: since `used_spk[]` is a subset of `revealed_spk[]`, we might be able to represent the used indexes more efficiently by storing their positions within the revealed sequence.

And we still need to figure out how to turn the resulting binary data into something we can actually export alongside a descriptor. That's where formats like Base64 and Bech32 come into play.

But these are two different problems: **how we compress the metadata and how we encode it for transport**.

The goal isn't to prove that Elias–Fano is the winner. It's to measure these alternatives and understand which ones work better for different wallet states.

### What about BIP-393?

There's also the question of how to export this metadata alongside a descriptor. [BIP-393](https://bips.dev/393/) introduces descriptor annotations, which could be an interesting place for it.

However, the current specification only accepts non-negative decimal integers as annotation values, so we can't simply attach a Base64 or Bech32 payload without changing the format.

That's something worth exploring later. For now, let's see if the idea actually works.

## Implementing it in BDK

It's one thing to describe SPK metadata on paper. It's another thing to make a real wallet use it without doing all the work we're trying to avoid in the first place.

For this research, I'm using [BDK](https://bitcoindevkit.org/) as the first reference implementation, with experimental changes in both `bdk` and `bdk_wallet`.

The first challenge wasn't actually encoding the metadata.
It was understanding how BDK handles derivation indexes internally, and what happens when we try to restore a wallet with a large derivation frontier.

### The problem with KeychainTxOutIndex

In BDK, the `KeychainTxOutIndex` is responsible for tracking the derivation state of each keychain and indexing its SPKs.

One of the things it tracks is `last_revealed`, the highest derivation index revealed by the wallet. BDK treats the revealed range as contiguous, meaning that if index 999 has been revealed, every index from 0 through 999 is logically considered revealed as well.

The problem is that, in the traditional restoration path, this logical state is closely tied to the physical materialization of SPKs.

For example, when we use `reveal_to_target()` to restore a frontier of 50,000, the indexer materializes the intermediate SPKs up to that index, along with its configured lookahead. But do we actually need to materialize every single one of them just to restore the wallet's previous derivation state?

Not necessarily.

A wallet knowing that it has revealed 50,000 indexes doesn't mean it needs all 50,001 corresponding scripts materialized in memory immediately. This distinction became the starting point of our experimental changes.

### Sparse keychain restoration

We introduced a way to restore the logical derivation frontier independently from the SPKs that are physically materialized.

The main change lives in `KeychainTxOutIndex`, where we added state to distinguish the last revealed index, whether a keychain was restored from sparse metadata, and which indexes are known to have historical activity.

Here's a simplified excerpt of the internal fields:

```rust
last_revealed: HashMap<DescriptorId, u32>,
sparse_restored: BTreeSet<DescriptorId>,
known_used: BTreeMap<DescriptorId, BTreeSet<u32>>,
```

The `sparse_restored` marker is particularly important.
It allows the indexer to distinguish an ordinary persisted wallet state from an explicit sparse snapshot, even when there are no known-used indexes.

We also introduced `restore_keychain_state()`, which lets us restore the frontier and the known-used indexes without filling every gap between them.

Let's take a more extreme version of our merchant example:

```text
last_revealed = 50_000
known_used    = [0, 50_000]
lookahead     = 25
```

With the descriptor already registered in the indexer, the experimental restoration API can be called like this:

```rust
index.restore_keychain_state(
    TestKeychain::External,
    Some(50_000),
    [0, 50_000],
)?;
```

Instead of filling the entire revealed range, the indexer materializes the known-used SPKs and the scripts required by its bounded lookahead.

In this particular example, that means:

```text
Initial lookahead: 0..24
Known-used:        0, 50_000

Materialized:      0..24, 50_000
Total:             26 SPKs
```

The logical frontier is still 50,000, and the next derivation index remains 50,001. But the indexer only needs 26 materialized SPKs in this scenario.

This is not just an expected behavior. We added [tests for this exact case](https://github.com/yanfsb/bdk/blob/03df61edfe430ca4520d480a7896210e0bdf7004/crates/chain/tests/test_keychain_txout_index.rs#L102-L119), including checks that intermediate indexes remain unmaterialized.

Of course, there's a reason we can't simply skip SPK materialization everywhere. The indexer needs concrete scripts to recognize outputs belonging to the wallet when processing transactions.

That's why our implementation materializes the known-used indexes before applying the corresponding transaction data.

In other words, we're not eliminating the SPKs required for transaction recognition. We're trying to avoid materializing the ones we don't need for that operation.

### Bringing it to bdk_wallet

With the sparse restoration behavior implemented in `bdk`, the next step was exposing it through `bdk_wallet`.

For now, our experimental `SpkMetadata` is a BDK-specific representation containing the keychain, the highest revealed index, and the known-used indexes:

```rust
pub struct SpkMetadata {
    keychain: KeychainKind,
    last_revealed: Option<u32>,
    used_indexes: Vec<u32>,
}
```

You might notice that `revealed_spk[]` isn't represented explicitly here.

That's because BDK's revealed indexes are logically contiguous. We can reconstruct that set from `last_revealed`, although we shouldn't assume the same behavior for every wallet implementation. Our broader research model still keeps `revealed_spk[]` as a separate concept.

At the wallet level, we now have two different approaches to applying metadata.

The original `apply_spk_metadata()` uses the ordinary address revelation path, which can materialize the entire range up to `last_revealed`.

The experimental `apply_spk_metadata_sparse()` instead calls our new indexer API:

```rust
let changeset = self.tx_graph.index.restore_keychain_state(
    keychain,
    metadata.last_revealed(),
    metadata.used_indexes().iter().copied(),
)?;

self.stage.indexer.merge(changeset);
```

Notice that we also merge the resulting `ChangeSet` into the wallet's staged state. Restoring the frontier only in memory wouldn't be enough if the wallet forgot about it after being closed and reopened.

We also added a separate request builder, `start_sync_with_spk_metadata_at()`, which uses the materialized known-used SPKs instead of requesting the complete revealed range.

That gives us an experimental path from metadata to wallet restoration and targeted transaction-history lookup, without necessarily deriving every intermediate SPK.

### Did we actually eliminate the work?

Well, not so fast.

There's an important detail in the implementation: existing APIs such as `revealed_keychain_spks()` still preserve their logical meaning. They represent the entire revealed range, even when some SPKs haven't been materialized yet.

If an operation explicitly iterates through that complete range, the missing scripts can still be derived lazily. So we might avoid materializing thousands of SPKs during restoration, only to derive them later during another operation.

There's also the question of synchronization. Requesting only the known-used SPKs from a metadata snapshot is not equivalent to discovering every possible payment. An address that was unused when the metadata was exported could have received funds afterward.

We still need a discovery strategy for that case.

So the question isn't simply whether sparse restoration can materialize fewer SPKs. We've already implemented a path that does that. The interesting question is whether this approach reduces the overall work required to restore and synchronize a wallet, without changing the behavior we expect from it.

And that's what we're trying to measure.

## Acknowledgements

The original idea behind this research came from [Leonardo Lima](https://github.com/oleonardolima) and [Edil Medeiros](https://github.com/edilmedeiros). I want to thank both of them for bringing up the idea and for the discussions that led me to explore it further.

I also want to thank [Vinteum](https://vinteum.org/) for supporting my work on Bitcoin open source and giving me the opportunity to dedicate time to this research.
