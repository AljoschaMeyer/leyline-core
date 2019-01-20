# Leyline Core

Leyline-core defines a data format for signed append-only logs that can be replicated in a peer-to-peer fashion. It supports partial replication of logs: To verify the integrity of the `x`-th entry, it is not necessary to have knowledge of all previous entries, only `log_2(x)` other entries need to be known. This ability comes at a moderate cost: The metadata size per entry grows logarithmically with the log length, bringing the total log size to `O(n * log_2(n))` (where `n` is the total number of entries).

## Concepts and Properties

Each append-only *log* is identified by a public key of a cryptographic signature scheme. Conceptually, an *entry* in the log is a tuple of:

- the *sequence number* of the entry (i.e. the offset in the log)
- *backlinks* to some previous entries, in the form of cryptographically secure hashes
- the hash of the actual *payload* of the entry
- a digital signature of all the previous data, issued with the log's public key

Since all entries are signed, only the holder of the log's private key (called the *author*) can create new entries. Thus logs can be replicated via potentially untrusted peers - any attempt to alter data or create new entries can be detected. The author however could *fork* a log by giving different entries of the same sequence number to different peers, resulting in a directed tree or even a directed acyclic graph rather than a log (aka a path in graph-theory parlance). This is where the backlinks come in: By iteratively traversing backlinks, any consumer of the log can verify that no fork occured. Forked logs are considered invalid from the point of the earliest fork, so there is no incentive for the author to deliberately fork their log. Additionally, backlinks establish a causal order on the existence of entries: Each entry guarantees that all previous entries have already existed prior in time, else their hash could not have been (transitively) included.

As a result of these properties, replication of a full log between two peers becomes very simple: The peers compare the newest sequence numbers they are aware of, and the peer with newer entries sends them and the corresponding payloads to the other peer. The receiver verifies the integrity by checking signature, sequence numbers and backlinks, and then stores the entries in their local copy of the log. In this mode of replication, all peers are equally capable, it does not matter where the data originated. And the worst a malicious peer can do is to deliberately withhold a suffix of the log.

This simple replication of full logs is inspired by [secure scuttlebutt](https://www.scuttlebutt.nz/) (ssb), and it only requires each entry to include a backlink to the previous entry. The core distinction between ssb and leyline-core lies in the ability to partially replicate logs. By including additional backlinks, leyline-core gains two crucial properties: First, the shortest backlink-paths between two entries have length logarithmic in the absolute difference of their sequence numbers. So to verify the created-before relation between two entries, only a small number of additional entries is needed.

While this property alone allows a peer to efficiently validate parts of a log, it does not automatically lead to transitive replication. Suppose a peer only has two entries `x` and `y`, as well as all entries needed to show that there is a path of backlinks from `y` to `x` (we call those entries a *certificate for `x` and `y`*). Now if another peer has entry `z` and wants to get entry `y`, the first peer may not have the entries necessary to show that there's a path of backlinks from `z` to `y`, and thus they might not be able to replicate, even though entry `y` itself is available. Leyline-core solves this problem by assigning a (logarithmically sized) set of entries `cert_pool(x)` (the *certificate pool of `x`*) to each entry `x`, such that for all entries `x` and `y` the union of `cert_pool(x)` and `cert_pool(y)` contains a path of backlinks from `y` to `x` (or the other way around). This allows fully transitive partial replication where peers only need to store logarithmically more entries than they are directly interested in.

## Encoding

Since signatures and hashes are computed over concrete bytes rather than abstract descriptions, leyline-core defines a precise binary encoding for the log entries. It uses multiformats so that new cryptographic primitives can be introduced without having to change the specification. The encoding is defined as the concatenation of the following byte strings:

- `tag`, a byte of zeros. This tag allows future extensions of the protocol in a backwards-compatible way, by preceding new kinds of entries with a different byte. This also allows building more specialized protocols on top of leyline-core. Leyline-core reserves all even tags (those whose least-significant bit is zero), non-core extensions and experiments can use odd tags without fear of conflict with potential updates to leyline-core. Examples of possible extensions include support for key rotation and log compaction.
- `payload_hash`, the hash of the payload, encoded as a canonical, binary [yamf-hash](https://github.com/AljoschaMeyer/yamf-hash).
- `seqnum`, the sequence number of the entry, encoded as a canonical [VarU64](https://github.com/AljoschaMeyer/varu64). Note that this limits the maximum size of logs to 2^64 - 1.
- `backlinks`, the concatenation of all backlinks, encoded as canonical, binary [yamf-hashes](https://github.com/AljoschaMeyer/yamf-hash). The number of backlinks is the number of ones in the binary representation of the seqnum. For details on which backlinks to include, see the next section. Backlinks are sorted from the oldest entry to the newest.
- `sig`, the signature obtained from signing the previous data with the author's private key, encoded as a canonical [VarU64](https://github.com/AljoschaMeyer/varu64) followed by that many bytes (the raw signature).

Note that peers don't necessarily have to adhere to this encoding when persisting or exchanging data. In some cases, tag, seqnum, backlinks and payload_hash can be reconstructed without them having to be transmitted, additionally having to parse all backlinks to find out where the sig starts would be cumbersome. This encoding is only binding for signature verification and hash computation, nothing more.

In particular, a fully replicated log can be stored in `O(n)` space by dropping backlinks and reconstructing them on-the-fly. Add in a cache, and you get both (heuristically) fast retrieval and linear storage overhead, at the cost of additional implementation effort and a larger state space of the program. This may not be worth it for most settings, but at least the option to go back to `O(n)` space complexity is available.

The authenticity of an entry can be verified by checking whether the signature is correct. Further validity checks on sequence numbers and backlinks are necessary to guarantee absence of forks and verify the claimed entry creation order, as described in the next section.

## Backlinks and Entry Verification

An entry contains backlinks to previous entries. Which entries to link to depends on the sequence number of the entry. To compute the set of backlinks, first find the unique decomposition of the sequence number into a sum of powers of two (aka its binary representation). Initialize an accumulator to zero, then for each of those powers of two in descending order: Add the power of two to the accumulator and link to the entry whose sequence number is one less than the accumulator. A few properties of this mechanism: Each entry except the zeroth entry links to the preceding one, each entry links to as many previous entries as the binary representation of its sequence number has ones, and the zeroth entry has no backlinks (as expected).

Whether the created-before relation claimed by the sequence numbers `x` and `y` of two entries is indeed valid can be verified by finding a path along the backlinks from the (claimed) newer entry to the older one. By verifying the created-before relation over all known entries of a log, a peer can check that the entries form indeed a single log (rather than disparate logs, trees or dags). By construction, each path along backlinks from `y` to `x` contains the shortest possible such path. Thus it is sufficient to follow backlinks along the shortest possible path, rather than having to explore across all backlinks. The shortest path is obtained by iteratively following backlinks starting at `y`, going to the smallest entry greater or equal to `x`. In addition to verifying the created-before relation in this way, a peer should also verify that all backlinks point to the entry of the expected sequence number.

An entry is considered *verified* if and only if:

- authenticity has been checked via the signature
- there is a backlink path from the entry to the zeroth entry

## Partial Replication and Log Verification

When partially replicating a log, a peer is only interested in a subset of the log's entries. Verifying all elements of a subset of a log independently does not yet verify the subset as a whole. For a subset to be considered verified, there must also be a backlink path from the newest entry to the zeroth entry that contains all other entries of the subset. Otherwise, the created-before relation claimed by the entry's sequence numbers isn't verified.

To traverse that path, the peer may have to store other entries it isn't really interested in. It doesn't need to fully verify those, but it should still check authenticity and follow backlinks if their target is available, to check for forks and invalid sequence numbers.

For some entry `x` the peer is interested in, the *certificate pool of `x`* is the (logarithmically sized) set of further entries the peer needs to store. It is defined as the union of the shortest backlink paths from:

- `x` to `prev_pow`, where `prev_pow` is the largest power of two less than or equal to `x`
- `prev_pow` to `0`
- `next_pow` to `x`, where `next_pow` is the smallest power of two strictly greater than `x`

Note that this is a superset of the entries needed to verify `x` (the first two backlink paths). The path from `next_pow` to `x` is needed so that the union of two certificate pools for two entries `x` and `y` (`x` < `y`) always contains a path from `y` to `0` via `x`. By requesting full certificate pools from its peers, a peer can then always verify the full log subset it is interested in.

Note also that the entries of the path from `next_pow` to `x` do not necessarily exist yet. The log subset can still be fully verified, since the non-existent entries are only needed to allow later extensions of the subset (at which point the new entries do exist). Finally note that peers can efficiently request full certificate pools by just specifying the single sequence number they are interested in. They can also tell their peers about subsets of the certificate pool they already have in the same way, resulting in very low communication overhead.

An example implementation of backlink computation, certificate pool computation and certificate checking in [rust](https://www.rust-lang.org/) can be found [here](https://gist.github.com/AljoschaMeyer/aa730547d1aec4649e6d569cc857a35a).

## Related Work

[Secure scuttlebutt](https://www.scuttlebutt.nz/) inspired leyline-core. Leyline-core can be seen as a generalization of ssb's signed linked lists to a signed skip-list to allow partial replication (at the cost of quasilinear log size rather than ssb's linear log size). To mitigate the lack of partial replication, ssb supports retrieval of log entries via hash, but forfeits the security guarantees of regularly replicated entries. As of writing, ssb also signs the payloads directly rather than their hash, making it impossible to locally delete log payloads without losing the ability to replicate the log to other peers.

[Hypercore](https://github.com/mafintosh/hypercore) is a distributed, sparsely-replicatable append-only log like leyline-core. It uses merkle trees whereas leyline-core only uses backlinks. The two approaches are related though, the backlink distribution of leyline-core entries corresponds directly to the merkle forests of hypercore. Leyline-core's focus on transitive sparse replication via certificate pools can also be implemented on top of hypercore. The largest difference is that hypercore does not provide guarantees about the created-before order of log entries. Those weaker guarantees allow hypercore to keep the metadata overhead linear.

Leyline-core is effectively a combination of the strong guarantees of ssb with the partial replication of hypercore, at the cost of quasilinear log metadata size.

## Implementation Notes

Some parts of leyline-core are fairly obvious to implement, others less so. The obvious ones include defining an in-memory representation of entries, encoding and decoding them, computing hashes, computing the set of backlinks an entry of a given sequence number includes, computing certificate pools. Beyond those, there are persistent storage and log verification. These can get trickier because of statefulness, asynchronicity and conflicting performance goals.

To verify an entry, you need access to previous entries. An verification implementation must thus decide on an interface to the log store. This interface should probably be nonblocking, since it might be backed up by persistent (i.e. slow) storage. It might be a good idea to keep this interface generic, allowing to transparently substitute different storage backends (for example in-memory storage, a local database, a cloud provider). The backends might also differ in how much data they want to persist, and how much data to recompute on-the-fly. Ideally, this would be hidden behind the interface (as usual, performance and abstraction might end up at odds, so feel free to disregard this patronizing advice).

Authenticity verification of entries can be done once upon receiving them, so there's not much complexity there. But for backlink verification, you probably want to somehow cache results. Entries can be in different states regarding backlink verification:

- fully verified (only those should be handed to the application layer)
- verified relative to some other entry
- parts of the backlinks have been verified, others haven't been (maybe because the backlink target isn't available in the database)
  - among those, it makes a difference whether the entry is intended for the application layer, or whether it is just part of some certificate pools

Keeping track of partial verification can be quite fiddly, so better plan this in advance when designing a storage backend. How much of this logic should be backend-specific and how much can be implemented via the generic interface between the leyline-core library and the backends is likely a matter of taste.

Some simplifying assumptions can be made, such as entries never being deleted, or entries being added only in ascending order of sequence numbers. Whether those assumptions are realistic and/or overly restrictive is another question altogether, depending on the backend and use case.

In general, a backend will likely track which entries are interesting for the application layer, and which entries are just available because they are needed for verification. There could be two interfaces to higher-level code, one that only provides explicitly requested data (that has been fully verified), and one that provides all data that happens to be available. The latter interface would need to distinguish between different levels of verification. A general-purpose application platform built on leyline-core probably only wants to expose the former interface, since the latter provides less simple guarantees regarding created-before order and forklessness.

Aside from metadata management, there's also the question of storing actual payloads. This can be fairly specialized, depending on the application(s) built on top of leyline-core. Details of this choice might also leak into a more specialized leyline-core implementation, particularly well-suited for a specific use-case.

The separation of payload and metadata via the payload_hash allows locally deleting payloads while keeping the log intact, verifiable and replicatable. Implementations should support this, and may want to store a set of "blocked" payload_hashes so that a deleted payload doesn't accidentally get stored again because of e.g. an overly eager replication mechanism. Or they might make sure that such retroactive fetching can't accidentally happen, thus mitigating the need for a block set. Either is fine, an implementation should just design the desired approach in advance.
