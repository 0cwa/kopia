# Distributed storage and durability

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `architecture-roadmap.md` sections 4.2–4.5.

## 4.2 Distributed blob layer

Proposed package boundary:

```text
repo/blob/distributed/
    storage.go       // blob.Storage implementation
    catalog.go       // logical blob records and checkpoints
    placement.go     // weighted deterministic placement
    stripe.go        // encoding, headers, and range mapping
    node_client.go   // storage-node protocol
    repair.go        // health evaluation and reconstruction
    audit.go         // inventory and sampled verification
    tombstone.go     // generations, deletion, and leases
```

Avoid `repo/blob/sharded` because Kopia already uses sharding terminology for path layout.

### Logical blob representation

A logical Kopia blob is divided into fixed-size stripes. Each stripe contains `k` systematic data fragments and `m` parity fragments.

```text
Logical blob B
  stripe 0 -> D0 D1 ... D(k-1) P0 ... P(m-1)
  stripe 1 -> D0 D1 ... D(k-1) P0 ... P(m-1)
  ...
```

Systematic coding permits a healthy range read to fetch the data fragment containing the requested range without decoding the whole stripe. A degraded range read fetches any `k` valid fragments for only the intersecting stripes.

### Shard header

Every shard is self-describing enough to audit and reconstruct the catalog:

```text
format_version
repository_id
data_namespace
logical_blob_id
logical_generation
codec_id
stripe_size
data_shards_k
parity_shards_m
stripe_number
fragment_number
logical_blob_length
placement_epoch
fragment_length
fragment_hash
optional stripe Merkle root
created_at
```

Headers are authenticated by a repository or coordinator signing key, or included in a signed descriptor. Fragment hashes are verified before decoding. Kopia's higher-level AEAD remains a final integrity check but is not the only physical corruption detector.

### Stripe descriptor and catalog record

A committed catalog record contains:

```text
blob ID and generation
logical length and timestamp
stripe/codec parameters
placement epoch and policy ID
committed shard locations or deterministic placement inputs
per-stripe roots or descriptor root
commit health summary
retention state
creation and tombstone generations
```

Bootstrap metadata receives stronger protection than bulk data:

- repository format records and catalog roots: multiple full replicas;
- catalog checkpoints and node maps: multiple full replicas plus offline recovery copies;
- content indexes and metadata packs: stronger replication or parity;
- bulk data packs: normal erasure-code policy.

### Storage node classes

Nodes advertise authenticated attributes:

```text
device ID
owner or administrative domain
host
power domain
network and site
capacity and weight
availability class
storage class
supported protocols
last inventory epoch
```

Node classes:

- **Core:** expected online; eligible for required data/parity placement.
- **Opportunistic:** laptops and intermittent devices; extra shards, caches, or soft claims.
- **Archive:** slow/cold but durable; eligible for full replicas or parity.
- **Transfer:** temporary staging or repair relay; never counted as durable.

### Placement

Use weighted rendezvous hashing or a CRUSH-like deterministic rule over repository, blob, stripe, fragment, node ID, and placement epoch. Selection is constrained by failure-domain policy.

Example rule:

```text
choose k+m nodes
require at least S sites or households
at most one fragment per host
at most F fragments per owner/admin domain
respect capacity, availability class, and drain state
```

A raw count of fragments is insufficient. Health must account for how many independent failure-domain combinations can satisfy the decode threshold, borrowing Tahoe-LAFS's “servers of happiness” lesson.

## 4.3 Catalog and coordinator

The catalog is the principal distributed-systems challenge. Kopia expects strong listing and read-after-write behavior, while arbitrary personal devices are often offline.

### Recommended initial model

- Immutable shard bytes live on many simple storage nodes.
- A small, stable set of three or five controllers runs a replicated state machine or strongly consistent database.
- Controllers commit node-map epochs, logical blob descriptors, tombstones, leases, and namespace heads.
- Signed catalog checkpoints are periodically exported to ordinary storage nodes and offline recovery media.
- Clients can read shards directly after obtaining an authorized descriptor, but controllers remain authoritative for current generations and listings.

This intentionally centralizes metadata consistency without centralizing bulk data.

### Alternatives and tradeoffs

| Model | Advantages | Costs and risks |
|---|---|---|
| Single gateway/catalog | Fastest prototype; easy Kopia semantics | Availability bottleneck; requires backups and failover |
| 3/5-node replicated catalog | Clear strong semantics; manageable complexity | Requires stable controller quorum; WAN write latency |
| Append-only log plus signed checkpoints | Recoverable and auditable; simple immutable history | Harder current-state queries and conflict handling |
| DHT/eventual catalog | Highly distributed | Does not naturally meet authoritative `ListBlobs`, atomic commit, GC, or deletion requirements |
| Fully deterministic placement without location records | Small metadata footprint | Still needs authoritative membership epochs, exceptions, migrations, and tombstones |

The first production design should use a small replicated catalog. A decentralized catalog is a later research project, not a prerequisite for distributed data placement.

## 4.4 Storage operations

### Put

1. Allocate a logical generation and placement epoch.
2. Stream the logical blob into fixed-size stripes.
3. Produce systematic data and parity fragments.
4. Upload fragments under temporary, generation-scoped names.
5. Verify node acknowledgements, fragment hashes, and failure domains.
6. Require the normal commit threshold (`k + safety_margin` and policy domains).
7. Quorum-commit the catalog descriptor.
8. Make the blob visible to `GetBlob`, metadata lookup, and `ListBlobs`.
9. Clean abandoned temporary fragments asynchronously.

Idempotency keys prevent retries from creating inconsistent generations.

### Get and range read

1. Resolve the current committed descriptor.
2. Map the requested byte range to stripes and systematic fragments.
3. Prefer nearby healthy systematic fragments.
4. Hedge slow requests when tail latency warrants it.
5. Verify fragment hashes.
6. Decode only degraded/intersecting stripes.
7. Verify the returned logical length and allow Kopia to verify higher-layer ciphertext/content integrity.

Research is needed to choose stripe size from actual Kopia range-read traces. Very small stripes increase metadata and request overhead; very large stripes amplify degraded reads and repairs.

### List

List from the catalog, never by polling all storage nodes. The catalog's committed generation is authoritative even when stale fragments remain on offline devices.

### Delete

1. Commit a generation-specific tombstone.
2. Remove the object from normal listings and namespace reachability.
3. Retain descriptor and reference evidence through a grace interval.
4. Propagate deletion leases to nodes.
5. Reclaim fragments only after retention and reference checks.
6. Reject inventory reports that attempt to reintroduce a tombstoned generation.

### Repair and rebalance

Repair is ciphertext-only in Track A. It needs no repository plaintext key if it reconstructs physical pack stripes.

The scheduler evaluates:

- decode health (`>= k` valid fragments);
- target health (`k+m` or policy target);
- independent failure-domain health;
- correlated-risk exposure;
- node drain and capacity pressure;
- soft-repair sources;
- estimated repair bandwidth and urgency.

It reconstructs missing fragments, verifies them, commits updated placement, and then releases obsolete copies. Rebalancing uses throttling and budgets so maintenance does not overwhelm foreground reads.

## 4.5 Auditing and inventory

Each node periodically publishes a signed inventory summary tied to a catalog epoch. The controller compares expected and observed placement.

Audit levels:

1. **Inventory presence:** node claims a fragment exists.
2. **Authenticated range sample:** controller requests unpredictable ranges and verifies hashes/Merkle paths.
3. **Full verification:** entire fragment or stripe is read and verified.
4. **End-to-end snapshot verification:** authorized client restores or checks logical contents.

Sampled auditing is inexpensive but probabilistic. Formal proofs of retrievability may be researched later if remote untrusted node economics justify their complexity.
