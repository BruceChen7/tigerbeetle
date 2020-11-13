# Design Document

This is a living document that keeps a (best effort) record of the design decisions behind TigerBeetle.

## Performance

TigerBeetle provides more performance than a general-purpose relational database such as MySQL or an in-memory database such as Redis:

* TigerBeetle **supports batching by design**. You can batch all the transfer prepares or commits you receive in a 50ms window and then send them all in a single network query to the database. This enables low-overhead networking, large sequential disk write patterns and amortized fsync across hundreds and thousands of account transfers.

* TigerBeetle **performs all balance tracking logic in the database**. This is a paradigm shift where we move the code to the data, not the data to the code, and eliminates the need for complex caching logic outside the database. You can keep your application layer simple, and completely stateless.

* TigerBeetle **masks transient gray failure performance problems**. For example, if a disk write that typically takes 4ms suddenly starts taking 4 seconds because the disk is slowly failing, TigerBeetle will use redundancy to mask the gray failure automatically without the user seeing any 4 second latency spike. This is a relatively new performance technique known as "tail tolerance" in the literature and something not provided by most existing databases.

> "The major availability breakdowns and performance anomalies we see in cloud environments tend to be caused by subtle underlying faults, i.e. gray failure (slowly failing hardware) rather than fail-stop failure."

## Safety

TigerBeetle provides more safety than a general-purpose relational database such as MySQL or an in-memory database such as Redis:

* Strict consistency, CRCs and crash safety are not enough.

* TigerBeetle **detects disk corruption** (3.45% per 32 months, PER DISK) and **detects misdirected writes**, where the disk firmware writes to the wrong sector (0.042% per 17 months, PER DISK), and **prevents data tampering** using cryptographic checksums.

* TigerBeetle **does not trust the fsync of a single disk**, or the hardware of a single node. Disk firmware and operating systems frequently have broken fsync, and single node systems fail all the time. TigerBeetle **performs synchronous replication** to a quorum of nodes.

* TigerBeetle **does not truncate the journal at the first corrupt disk sector**, but **recovers from local corruption without violating the global consensus protocol**, already providing more safety than replicated state machines such as ZooKeeper and LogCabin.

## Developer-Friendly

TigerBeetle does all the typical ledger validation, balance tracking, persistence and replication for you, all you have to do is:

1. send in a batch of prepares to TigerBeetle (in a single network hop), and then
2. send in a batch of commits to TigerBeetle (in a single network hop).

## Architecture

In theory, TigerBeetle is a replicated state machine, that **takes an initial starting state** (account opening balances), and **applies a set of input events** (transfers) in deterministic order, after first replicating these input events safely, to **arrive at a final state** (account closing balances).

In practice, TigerBeetle is based on the [LMAX Exchange Architecture](https://skillsmatter.com/skillscasts/5247-the-lmax-exchange-architecture-high-throughput-low-latency-and-plain-old-java) and makes a few improvements.

We take the same three classic LMAX steps:

1. journal incoming events safely to disk, and replicate to backup nodes, then
2. apply these events to in-memory state, then
3. ACK

And then we introduce something new:

4. delete the local journalling step entirely, and
5. replace it with parallel 3-out-of-5 quorum replication to 5 distributed journal nodes.

Our architecture then becomes three easy steps:

1. replicate incoming events safely to a quorum of distributed journal nodes, then
2. apply these events to in-memory state, then
3. ACK

That's how TigerBeetle **eliminates gray failure in the leader's local disk**, and how TigerBeetle **eliminates gray failure in the network links to the replication nodes**.

Like LMAX, TigerBeetle uses separate threads for server networking, replication and in-memory business logic, with strict single-threading to enforce the single writer principle and prevent multi-threaded access to data. Again, like LMAX, TigerBeetle uses ring buffers to move event batches from networking to replication to logic threads for lockless communication.

### Leader Modes

While the TigerBeetle leader can be run as a networked client-server database engine,
TigerBeetle takes inspiration from SQLite by enabling the TigerBeetle leader itself to be embedded as an in-process library within the application to eliminate gray failure in the network link to the leader:

* Applications like Mojaloop that must transform individual HTTP requests into batches of transfers may find that HTTP is a bottleneck, and may want to operate the TigerBeetle leader in **networked mode** to have multiple stateless client processes drive high levels of load to saturate TigerBeetle.

* Applications that can drive high levels of load from a single client process may want to operate the TigerBeetle leader in **embedded mode** to eliminate gray failure completely.

## Data Structures

The best way to understand TigerBeetle is through the data structures it provides. All data structures are **fixed-size** for performance and simplicity, and there are two main kinds of data structures, **events** and **states**.

### Events

Events are **immutable data structures** that **instantiate or mutate state data structures**:

* Events cannot be changed, not even by other events.
* Events cannot be derived, and must therefore be recorded before being executed.
* Events must be executed one after another in deterministic order to ensure replayability.
* Events may depend on past events (should they choose).
* Events cannot depend on future events.
* Events may depend on states being at an exact version (should they choose).
* Events may succeed or fail but the result of an event is never stored in the event, it is stored in the state instantiated or mutated by the event.
* Events only ever have one immutable version, which can be referenced directly by the event's id.
* Events should be retained for auditing purposes. However, events may be drained into a separate cold storage system, once their effect has been captured in a state snapshot, to compact the journal and improve startup times.

**create_transfer**: Create a transfer between accounts (maps to a "prepare"). We use 64-bit words at a minimum to avoid unaligned operations on less than a machine word and to simplify struct packing. We group fields in descending order of size to avoid unnecessary struct padding in C implementations.

```
          create_transfer {
                      id: 16 bytes (128-bit)
        debit_account_id: 16 bytes (128-bit)
       credit_account_id: 16 bytes (128-bit)
                custom_1: 16 bytes (128-bit) [optional, e.g. enforce a foreign key relation on a pre-existing `transfer` state]
                custom_2: 16 bytes (128-bit) [optional, e.g. a short description for the transfer]
                custom_3: 16 bytes (128-bit) [optional, e.g. an ILPv4 condition to validate a subsequent `commit-transfer` event]
                   flags:  8 bytes ( 64-bit) [optional, to modify the usage of custom slots, and for future feature expansion]
                  amount:  8 bytes ( 64-bit) [required, an unsigned integer in the unit of value of the debit and credit accounts, which must be the same for both accounts]
                 timeout:  8 bytes ( 64-bit) [optional, a quantity of time, i.e. an offset in nanoseconds from timestamp]
               timestamp:  8 bytes ( 64-bit) [reserved, assigned by the leader before journalling]
} = 128 bytes (2 CPU cache lines)
```

**commit_transfer**: Commit a transfer between accounts (maps to a "fulfill"). A transfer can be accepted or rejected by toggling a bit in the `flags` field.

```
          commit_transfer {
                      id: 16 bytes (128-bit)
                custom_1: 16 bytes (128-bit) [optional, e.g. enforce a foreign key relation on a pre-existing `transfer` state]
                custom_2: 16 bytes (128-bit) [optional, e.g. a short description for the accept or reject]
                custom_3: 16 bytes (128-bit) [optional, e.g. an ILPv4 preimage to validate against the condition of a previous `commit-transfer` event]
                   flags:  8 bytes ( 64-bit) [optional, used to indicate transfer success/failure, to modify usage of custom slots, and for future feature expansion]
               timestamp:  8 bytes ( 64-bit) [reserved, assigned by the leader before journalling]
} = 80 bytes (2 CPU cache lines)
```

**create_account**: Create an account.

* We use the terms `credit` and `debit` instead of "payable" or "receivable" since the meaning of a credit balance depends on whether the account is an asset or liability, income or expense.
* An `accepted` amount refers to an amount posted by a committed transfer.
* A `reserved` amount refers to an inflight amount posted by a created transfer only, where the commit is still outstanding, and where the transfer timeout has not yet fired.
* The total debit balance of an account is given by adding `debit_accepted` plus `debit_reserved`, and both these individual amounts must be less than their respective limits. Likewise for the total credit balance of an account.
* The total balance of an account can be derived by subtracting the total credit balance from the total debit balance.
* We keep both sides of the ledger (debit and credit) separate to avoid dealing with signed numbers, and to preserve more information about the nature of an account. For example, two accounts could have the same balance of 0, but one account could have 1,000,000 units on both sides of the ledger, whereas another account could have 1 unit on both sides, both balancing out to 0.
* Once created, an account may be changed only through transfer events, for auditing purposes and to avoid complications from changing account invariants. Thus, limits may be changed only by creating a new account.

```
           create_account {
                      id: 16 bytes (128-bit)
                  custom: 16 bytes (128-bit) [optional, opaque to TigerBeetle]
                   flags:  8 bytes ( 64-bit) [optional, used to modify usage of custom slot, and for future feature expansion]
                    unit:  8 bytes ( 64-bit) [optional, opaque to TigerBeetle, a unit of value, e.g. gold bars or marbles]
          debit_reserved:  8 bytes ( 64-bit)
          debit_accepted:  8 bytes ( 64-bit)
         credit_reserved:  8 bytes ( 64-bit)
         credit_accepted:  8 bytes ( 64-bit)
    debit_reserved_limit:  8 bytes ( 64-bit) [optional, a non-zero limit]
    debit_accepted_limit:  8 bytes ( 64-bit) [optional, a non-zero limit]
   credit_reserved_limit:  8 bytes ( 64-bit) [optional, a non-zero limit]
   credit_accepted_limit:  8 bytes ( 64-bit) [optional, a non-zero limit]
                 padding:  8 bytes ( 64-bit) [reserved]
               timestamp:  8 bytes ( 64-bit) [reserved]
} = 128 bytes (2 CPU cache lines)
```

### States

States are **data structures** that capture the results of events:

* States can always be derived by replaying all events.

TigerBeetle provides two state data structures:

* **Transfer**: A transfer (and whether created/accepted/rejected).
* **Account**: An account showing the effect of all transfers.

However:

* To simplify client-side implementations, to reduce memory copies and to reuse the wire format of event data structures as much as possible, we reuse our `create_transfer`, `create_account` event data structures to instantiate the corresponding state data structures, with the caveat that the accept/reject bit flag in any `flags` field is reserved for the state equivalent only.
* We use the `flags` field to track the created/accepted/rejected state of a transfer state.

To give you an idea of how this works in practice:

* The `create_transfer` event is immutable and we persist this in the log.
* We also reuse and apply this as a separate `transfer` state (as yet uncommitted, neither accepted or rejected) to our in-memory hash table and any subsequent state snapshot.
* When we receive a corresponding `commit_transfer` event, we then modify the `transfer` state (not the original `create_transfer` event which remains immutable, "what-you-sent-is-what-we-persist") to indicate whether the transfer was accepted or rejected.
* We do not support custom commit results beyond accepted or rejected. A participant cannot indicate to TigerBeetle why they are rejecting the transfer. However, they may include an opaque reason in any of the custom fields.
* The creation timestamp of a transfer is stored in the `create_transfer` state and in the `transfer` state. However, the commit timestamp is stored in the `commit_transfer` event only.

## Timestamps

**All timestamps in TigerBeetle are strictly reserved** to prevent unexpected distributed system failures due to clock skew. For example, a payer could have all their transfers fail, simply because their clock was behind. This means that only the TigerBeetle leader may assign timestamps to data structures.

Timestamps are assigned at the moment a batch of events is received by the TigerBeetle leader from the application. Within a batch of events, each event timestamp must be **strictly increasing**, so that no two events will ever share the same timestamp and so that timestamps may **serve as sequence numbers**. Timestamps are therefore stored as 64-bit **nanoseconds**, not only for optimal resolution and compatibility with other systems, but to ensure that there are enough distinct timestamps when processing hundreds of events within the same millisecond window.

Should the user need to specify an expiration timestamp, this can be done in terms of **a relative quantity of time**, i.e. an offset relative to the event timestamp that will be assigned by the TigerBeetle leader, rather than in terms of **an absolute moment in time** that is set according to another system's clock.

For example, the user may say to the TigerBeetle leader: "please expire this transfer 7 seconds after the timestamp you assign to it when you receive it from me".

The intention of this design decision is to reduce the blast radius in the worst case to the clock skew experienced by the network link between the client and the leader, instead of the clock skew experienced by all distributed systems participating in two-phase transfers. Where the leader is embedded within a local process, the blast radius is even less.

## Protocol

While the TigerBeetle leader can be embedded in-process with an `execute(command: integer, batch: buffer)` function signature to execute commands and event batches, there is also a TCP wire protocol to use the TigerBeetle as a remote network server database.

The TCP wire protocol is:

* a fixed-size header that can be used for requests or responses,
* followed by variable-length data.

```
HEADER (64 bytes)
16 bytes CHECKSUM META (remaining HEADER)
16 bytes CHECKSUM DATA
16 bytes ID (to match responses to requests and enable multiplexing in future)
 8 bytes MAGIC (for protocol versioning)
 4 bytes COMMAND
 4 bytes SIZE (of DATA if any)
DATA (multiple of 64 bytes)
................................................................................
................................................................................
................................................................................
```

The `DATA` in **the request** for a `create_transfer` command looks like this:

```
{ create_transfer event struct }, { create_transfer event struct } etc.
```

* All event structures are simply appended one after the other in the `DATA`.

The `DATA` in **the response** to a `create_transfer` command looks like this:

```
{ index: integer, error: integer }, { index: integer, error: integer }, etc.
```

* Only failed `create_transfer` events emit an `error` struct in the response. We do this to optimize the common case where most `create_transfer` events succeed.
* The `error` struct includes the `index` into the batch of the `create_transfer` event that failed and a TigerBeetle `error` return code indicating why.
* All other `create_transfer` events succeeded.
* This `error` struct response strategy is the same for `create_account` events.

### Protocol Design Decisions

The header is a multiple of 64 bytes because we want to keep the subsequent data
aligned to 64-byte cache line boundaries. We don't want any structure to
straddle multiple cache lines unnecessarily.

We order the header struct as we do to keep any future C implementations
padding-free.

The ID means we can switch to any reliable, unordered protocol in future. This
is useful where you want to multiplex messages with different priorities. For
example, huge batches would cause head-of-line blocking on a TCP connection,
blocking critical control-plane messages.

The MAGIC means we can do backwards-incompatible upgrades incrementally, without
shutting down the whole TigerBeetle system. The MAGIC also lets us discard any
obviously bad traffic without doing any checksum calculations.

We use BLAKE3 as our checksum, truncating the checksum to 128 bits.

The reason we use two checksums instead of only a single checksum across header
and data is that we need a reliable way to know the size of the data to expect,
before we start receiving the data.

Here is an example showing the risk of a single checksum for the recipient:

1. We receive a header with a single checksum protecting both header and data.
2. We extract the SIZE of the data from the header (4 GB in this case).
3. We cannot tell if this SIZE value is corrupt until we receive the data.
4. We wait for 4 GB of data to arrive before calculating/comparing checksums.
5. Except the SIZE was corrupted in transit from 16 MB to 4 GB (2 bit flips).
6. We never detect the corruption, the connection times out and we miss our SLA.

## Why C/Zig?

We want:

* **C ABI compatibility** to embed the TigerBeetle leader library or TigerBeetle network client directly into any language, to match the portability and ease of use of the [SQLite library](https://www.sqlite.org/index.html), the most used database engine in the world.
* **Control of the memory layout, alignment, and padding of data structures** to avoid cache misses and unaligned accesses, and to allow zero-copy parsing of data structures from off the network.
* **Explicit static memory allocation** from the network all the way to the disk with **no hidden memory allocations**.
* **OOM safety** as the TigerBeetle leader library needs to manage GBs of in-memory state without crashing the embedding process.
* Direct access to **io_uring** for fast, simple networking and file system operations.
* Direct access to **fast CPU instructions** such as `POPCNT`, which are essential for the hash table implementation we want to use.
* Direct access to **existing C libraries** without the overhead of FFI.
* **No artificial bottlenecks**. For example, Node has a [2x TCP throughput bottleneck](https://github.com/nodejs/node/pull/6923) because of surplus memory copies in the design of the socket stream interface, and libuv [forces unnecessary syscalls and context switches when offloading I/O to the thread pool](https://www.scylladb.com/2020/05/05/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/) wasting thousands of NVMe IOPS.
* **Strict single-threaded control flow** to eliminate data races by design and to enforce the single writer principle.
* **Compiler support for error sets** to enforce [fine-grained error handling](https://www.eecg.utoronto.ca/~yuan/papers/failure_analysis_osdi14.pdf).
* A developer-friendly and fast build system.

C is a natural choice, however Zig retains C ABI interoperability, offers relief from undefined behavior and makefiles, and provides an order of magnitude improvement in runtime safety and fine-grained error handling. Zig is a good fit with its emphasis on explicit memory allocation and OOM safety. Since Zig is pre-1.0.0 we plan to use only stable language features. It's a great time for TigerBeetle to adopt Zig since our stable roadmaps will probably coincide, and we can catch and surf the swell as it breaks.

## What is a tiger beetle?

Two things got us interested in tiger beetles as a species:

1. Tiger beetles are ridiculously fast... a tiger beetle can run at a speed of 9 km/h, about 125 body lengths per second. That’s 20 times faster than an Olympic sprinter when you scale speed to body length, **a fantastic speed-to-size ratio**.

2. Tiger beetles thrive in different environments, from trees and woodland paths, to sea and lake shores, with the largest of tiger beetles living primarily in the dry regions of Southern Africa... and that's what we want for TigerBeetle, **something that's fast and safe to deploy everywhere**.

## References

The collection of papers behind TigerBeetle:

* [LMAX - How to Do 100K TPS at Less than 1ms Latency - 2010](https://www.infoq.com/presentations/LMAX/) - Martin Thompson on mechanical sympathy, and why a relational database is not the right solution.

* [The LMAX Exchange Architecture - High Throughput, Low Latency and Plain Old Java - 2014](https://skillsmatter.com/skillscasts/5247-the-lmax-exchange-architecture-high-throughput-low-latency-and-plain-old-java) - Sam Adams on the high-level design of LMAX.

* [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf) - A high performance alternative to bounded queues for
exchanging data between concurrent threads.

* [Evolution of Financial Exchange Architectures - 2020](https://www.youtube.com/watch?v=qDhTjE0XmkE) - Martin Thompson looks at the evolution of financial exchanges and explores the state of the art today.

* [Gray Failure](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/06/paper-1.pdf) - "The major availability breakdowns and performance anomalies we see in cloud environments tend to be caused by subtle underlying faults, i.e. gray failure rather than fail-stop failure."

* [The Tail at Store: A Revelation from Millions
of Hours of Disk and SSD Deployments](https://www.usenix.org/system/files/conference/fast16/fast16-papers-hao.pdf) - "We find that storage performance instability is not uncommon: 0.2% of the time, a disk is more than 2x slower than its peer drives in the same RAID group (and 0.6% for SSD). As a consequence, disk and SSD-based RAIDs experience at least one slow drive (i.e., storage tail) 1.5% and 2.2% of the time."

* [The Tail at Scale](https://www2.cs.duke.edu/courses/cps296.4/fall13/838-CloudPapers/dean_longtail.pdf) - "A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first."

* [Werner Vogels on Amazon Aurora](https://www.allthingsdistributed.com/2019/03/Amazon-Aurora-design-cloud-native-relational-database.html) - *Bringing it all back home*... choice quotes:

  > Everything fails all the time. The larger the system, the larger the probability that something is broken somewhere: a network link, an SSD, an entire instance, or a software component.

  > And then, there's the truly insidious problem of "gray failures." These occur when components do not fail completely, but become slow. If the system design does not anticipate the lag, the slow cog can degrade the performance of the overall system.

  > ... you might have three physical writes to perform with a write quorum of 2. You don't have to wait for all three to complete before the logical write operation is declared a success. It's OK if one write fails, or is slow, because the overall operation outcome and latency aren't impacted by the outlier. This is a big deal: A write can be successful and fast even when something is broken.

* [Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases](https://www.allthingsdistributed.com/files/p1041-verbitski.pdf)

* [Amazon Aurora: On Avoiding Distributed Consensus for I/Os, Commits, and Membership Changes](https://dl.acm.org/doi/pdf/10.1145/3183713.3196937?download=true)

* [Amazon Aurora Under The Hood - Quorum and Correlated Failure](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-quorum-and-correlated-failure/)

* [Amazon Aurora Under The Hood - Quorum Reads and Mutating State](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-quorum-reads-and-mutating-state/)

* [Amazon Aurora Under The Hood - Reducing Costs using Quorum Sets](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-reducing-costs-using-quorum-sets/)

* [Amazon Aurora Under The Hood - Quorum Membership](https://aws.amazon.com/blogs/database/amazon-aurora-under-the-hood-quorum-membership/)

* [Viewstamped Replication: The Less-Famous Consensus Protocol](https://brooker.co.za/blog/2014/05/19/vr.html)

* [Viewstamped Replication Explained](https://blog.brunobonacci.com/2018/07/15/viewstamped-replication-explained/)

* [ZFS: The Last Word in File Systems (Jeff Bonwick and Bill Moore)](https://www.youtube.com/watch?v=NRoUC9P1PmA) - On disk failure and corruption, the need for checksums... and checksums to check the checksums, and the power of copy-on-write for crash-safety.

* [SDC 2018 - Protocol-Aware Recovery for Consensus-Based Storage](https://www.youtube.com/watch?v=fDY6Wi0GcPs) - Why replicated state machines need to distinguish between a crash and corruption, and why it would be disastrous to truncate the journal when encountering a checksum mismatch.

* [Can Applications Recover from fsync Failures?](https://www.usenix.org/system/files/atc20-rebello.pdf) - Why we use Direct I/O in TigerBeetle and why the kernel page cache is a dangerous way to recover the journal, even when restarting from an fsync() failure panic.