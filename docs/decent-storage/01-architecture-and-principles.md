# Architecture, goals, and principles

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `architecture-roadmap.md` sections 1–4.1.

**Status:** Proposed  
**Date:** 2026-08-03  
**Baseline:** `0cwa/kopia` master, with Kopia's BLOB → content-addressable block storage → object storage → manifest layering.

## 1. Executive decision

The requested system combines four problems that have different consistency and security requirements:

1. immutable backup and historical retention;
2. distributed storage across heterogeneous devices;
3. mutable multi-device and multi-user collaboration;
4. encrypted cross-user deduplication.

They should share content primitives, but they should not be collapsed into one mutable repository abstraction.

The recommended design has four planes:

```text
Local files and folders
        |
        v
+----------------------------+
| Client/device agent        |
| - scan and watch           |
| - content-defined chunking |
| - sync and merge           |
| - client-side crypto (v2)  |
| - availability claims      |
+-------------+--------------+
              |
              | immutable commits / encrypted objects
              v
+----------------------------+
| Namespace/control plane    |
| - identities and devices   |
| - encrypted directory DAG  |
| - share membership epochs  |
| - opaque object references |
| - signed commit heads      |
+-------------+--------------+
              |
              | logical encrypted blobs
              v
+----------------------------+
| Distributed durability     |
| - replicated catalog       |
| - stripe descriptors       |
| - erasure coding           |
| - topology-aware placement |
| - audit, repair, rebalance |
+----+---------+---------+----+
     |         |         |
  Device A  Device B  Device C ...

Optional independent service:
VOPRF/POPRF key service for opt-in global deduplication
```

The implementation is divided into two compatibility tracks.

### Track A: Kopia-compatible distributed durability

Track A preserves current Kopia content IDs, encryption, packs, indexes, snapshots, and maintenance. It adds distribution below the content layer through a new `blob.Storage` implementation or an S3-compatible gateway.

This track can provide:

- erasure-coded storage across devices;
- failure-domain-aware placement;
- ciphertext-only repair;
- local plaintext files as restore seeds or logical reseed sources;
- a trusted multi-user server with stronger authorization than current Kopia.

It cannot safely provide exact plaintext-backed regeneration of arbitrary existing pack stripes because current pack bytes depend on randomized encryption, repository secrets, packing order, offsets, and unrelated contents in the same pack.

### Track B: multi-principal format v2

Track B introduces a new logical object and namespace format designed for:

- client-side encryption from an untrusted coordinator;
- per-user and per-shared-folder namespaces;
- random user-visible object references;
- capability-based read, append, write, verify, repair, and administration;
- deterministic encrypted objects where deduplication is enabled;
- optional VOPRF-assisted message-locked encryption;
- possible plaintext-backed virtual data shards.

Track B is a new repository protocol. Existing repositories require export/import or a clearly specified migration; they must not be silently upgraded.

## 2. Goals and non-goals

### Goals

- Survive device loss and corruption with independently verifiable redundancy.
- Support stable servers, home NAS devices, laptops, and intermittently online nodes without treating them as equally reliable.
- Preserve point-in-time snapshots even when live working folders change or disappear.
- Support shared folders with explicit read/write/admin authority and multi-writer conflict handling.
- Deduplicate identical data within a user, shared folder, organization, or optionally globally.
- Prevent ordinary users from learning that another user owns a candidate file through a cheap API probe.
- Keep bulk processing limited to rolling hashes, compression, hashing, symmetric encryption, and erasure coding.
- Make the storage, crypto, and synchronization formats independently testable and versioned.

### Non-goals for early releases

- A permissionless public storage market.
- Byzantine consensus across every storage node.
- General-purpose filesystem CRDT semantics for all file types.
- Retroactively making recipients forget plaintext or keys already received.
- Hiding ciphertext equality from every operator while retaining inexpensive physical global deduplication.
- ORAM, generic multiparty computation, homomorphic encryption, or blockchain-based placement on the normal data path.

## 3. Architectural principles and invariants

### 3.1 Locator is not authority

Blob IDs, content IDs, ciphertext hashes, pack IDs, stripe IDs, and physical node addresses identify data. None of them grants permission to read or attach a reference. Authorization requires namespace reachability, an opaque reference, an access token, or possession of the relevant cryptographic capability.

### 3.2 Hard and soft durability are separate

The system reports at least two values:

- **Autonomous durability:** the object can be reconstructed from authenticated encrypted storage without any user's plaintext file being online.
- **Assisted recoverability:** authorized devices currently claim they can regenerate logical content from a working copy, cache, or archive representation.

Historical retention policies are evaluated against autonomous durability. Soft sources may reduce repair urgency and bandwidth, but they do not silently satisfy an archival hard floor.

### 3.3 Commit is a catalog operation, not a successful shard upload

A logical blob becomes visible only after:

- its required shard set has been uploaded and verified;
- the minimum number of independent failure domains is satisfied;
- the catalog has durably committed the descriptor and generation.

A write with only the decoding threshold `k` is degraded and should not normally be acknowledged as healthy. Normal commit requires `k + safety_margin` and the configured failure-domain rule.

### 3.4 Data is immutable; placement is versioned

Logical object generations and ciphertext bytes are immutable. Node membership and placement rules evolve through explicit epochs. Rebalancing writes new shard locations and then commits a new placement state; it does not reinterpret old data under an unversioned current node map.

### 3.5 Deletion is monotonic

Logical deletion creates a signed or quorum-committed tombstone naming object generation and namespace state. Offline nodes cannot resurrect an old generation after reconnecting. Physical reclamation is delayed by retention, grace, and reference checks.

### 3.6 Equality leakage is a product policy

Deduplication domains are explicit:

- private user/folder;
- trusted team or organization;
- server-aided global.

The user interface and API explain that broader deduplication reveals broader equality information to at least the deduplication coordinator.

## 4. Components

## 4.1 Kopia content and snapshot layer

Kopia already supplies the strongest parts of the backup system:

- content-defined splitting using a rolling hash;
- repository-wide content deduplication;
- compression and authenticated encryption;
- aggregation into larger pack blobs;
- indexes mapping content IDs to pack, offset, and length;
- immutable object trees and snapshot manifests;
- retention, maintenance, verification, and restore.

The current `repo/blob.Storage` contract is an appropriate physical-storage boundary, but its semantics are strict: atomic writes, immediate read-after-write visibility through reads and listings, authoritative listing, monotonic timestamps, and range reads. The distributed backend must emulate those semantics rather than expose an eventually consistent union of devices.
