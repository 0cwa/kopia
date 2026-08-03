# Decent Storage architecture pack

**Status:** Proposed architecture, August 2026  
**Target codebase:** `0cwa/kopia`, based on Kopia's current repository architecture  
**Purpose:** Synthesize the distributed-storage, working-copy, multi-user sharing, and privacy-preserving deduplication proposals into an implementable program.

## Documents

1. [Architectural roadmap](architecture-roadmap.md) — system boundaries, data flows, implementation phases, tradeoffs, and exit criteria.
2. [Security and privacy design](security-and-privacy.md) — threat model, capabilities, encrypted namespaces, key management, deduplication leakage, and revocation.
3. [Research backlog](research-backlog.md) — unresolved technical questions, experiments, decision gates, and validation plan.

## Core recommendation

Build the project as four cooperating planes:

1. **Immutable content plane:** Kopia-derived chunking, snapshots, compression, packing, indexing, retention, and restore.
2. **Distributed durability plane:** authenticated erasure-coded stripes placed across independent failure domains, with repair, auditing, rebalancing, and tombstones.
3. **Mutable collaboration plane:** local working folders, watcher-assisted scanning, three-way reconciliation, staging, and explicit conflict handling.
4. **Identity and namespace plane:** user/device identities, encrypted directory metadata, capabilities, key envelopes, signed commits, membership epochs, and optional server-aided deduplication.

The implementation should follow two tracks:

- **Track A — Kopia-compatible distributed durability.** Preserve the current Kopia repository and pack format. Add a distributed `blob.Storage` provider or S3-compatible gateway. Treat plaintext files as verified restore/reseed sources, not literal pack shards.
- **Track B — multi-principal format v2.** Introduce deterministic encrypted logical objects, encrypted per-user/per-share namespaces, opaque references, capability-based access, and optional VOPRF-assisted cross-user deduplication.

Track A is a practical product and learning vehicle. Track B is a new repository protocol and security model; it should not be represented as a small ACL extension to current Kopia.

## Non-negotiable invariants

- A content hash or storage identifier is a locator, never authorization.
- A committed object satisfies a hard durability policy without relying on mutable plaintext working copies.
- The catalog never exposes a logical blob before the required shards and failure domains are committed.
- Deletion uses generations and durable tombstones so reconnecting nodes cannot resurrect old data.
- Global deduplication is opt-in and its equality leakage is documented.
- Privacy mode must not expose a simple duplicate-existence oracle to ordinary clients.
- New cryptographic formats are versioned, domain-separated, algorithm-agile, and independently reviewed.

## Recommended first milestone

Implement a permissioned distributed blob gateway with:

- systematic Reed–Solomon stripes;
- range-aware reads;
- deterministic failure-domain placement;
- a small replicated catalog;
- signed shard headers and catalog checkpoints;
- repair and rebalance workers;
- failure-injection tests against Kopia's `blob.Storage` semantics.

In parallel, build a local seed agent that indexes ordinary files with the same logical chunking rules and can satisfy restores without weakening hard archival durability.
