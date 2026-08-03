# Research backlog: cryptography, decentralization, and validation

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `research-backlog.md` sections 5–12.

## 5. Deduplication and cryptography research

## R8 — canonical object and deterministic encryption

**Question**

What exact format safely creates identical ciphertext for identical logical representations inside a dedup realm?

**Subproblems**

- segment boundaries and length encoding;
- canonical compression profile and cross-version determinism;
- associated-data contents;
- message-derived key security;
- deterministic/misuse-resistant authenticated encryption;
- domain separation among data, metadata, manifests, and adapters;
- padding and small-object policy;
- migration when format changes;
- test vectors and independent implementations.

**Required process**

- cryptographic design document;
- comparison of standardized or peer-reviewed candidate constructions;
- external cryptographer review;
- public vectors and negative tests;
- no production deployment before review findings are resolved.

This is the largest single research gate in Track B.

## R9 — duplicate-upload privacy protocol

**Question**

What does “not easily know another user has the same file” mean in measurable protocol terms?

**Candidate profiles**

1. **Full upload:** no explicit hit response; storage saves space only.
2. **Padded full upload:** adds timing/size buckets.
3. **Proof of ownership:** saves bandwidth but leaks likely existence.
4. **Trusted realm direct skip:** maximum efficiency, weak privacy.

**Experiment**

Build a red-team client that submits candidate files and observes:

- request count and bytes;
- timing distributions;
- error behavior;
- quota changes;
- physical-node contacts;
- retry and challenge patterns.

Define a privacy profile with an attack cost and observable leakage, rather than claiming perfect hiding.

## R12 — global dedup segmentation

**Question**

Should global dedup use whole files, fixed large segments, content-defined large segments, or small Kopia chunks?

**Tradeoffs**

- smaller chunks improve storage savings after edits;
- smaller chunks increase VOPRF calls, indexes, equality leakage, and guessability;
- whole files leak less partial similarity but miss near-duplicate savings;
- large content-defined segments may offer a useful middle ground.

**Dataset study**

Measure real corpora: software installers, photos/videos, documents, VM images, archives, and common public files. Report dedup savings versus segment size and estimate low-entropy/common-fragment exposure.

**Recommended starting hypothesis**

Private/team realms use normal small content-defined chunks. Global realm uses whole files or multi-megabyte segments and rejects or random-encrypts very small objects.

## R18 — quotas and billing side channels

**Question**

How should users be charged and informed without revealing marginal physical storage?

**Candidates**

- logical bytes only;
- logical bytes plus coarse account-wide savings, excluding cross-user detail;
- flat subscription;
- privacy-preserving accounting tokens (later).

Test whether quota deltas, upload speed, or invoices let a user distinguish hit/miss. The simplest safe rule is logical-byte billing regardless of deduplication.

## R19 — threshold VOPRF

**Question**

Does splitting the PRF key among independent operators produce meaningful risk reduction relative to operational complexity?

**Challenges**

- protocol/library maturity;
- distributed key generation and share refresh;
- quorum availability and latency;
- proof aggregation and batch evaluation;
- abuse-rate coordination;
- backups without reconstructing the key;
- operator collusion assumptions.

Proceed only after a single independently operated VOPRF service is measured and audited. Threshold operation is not automatically provided by RFC 9497.

## R22 — post-quantum profile

**Question**

Which long-lived keys need hybrid post-quantum protection, and what is the format migration strategy?

**Study**

- retention horizon and harvest-now-decrypt-later risk;
- envelope and signature sizes;
- browser/mobile/server library maturity;
- current NIST errata and implementation guidance;
- hybrid X25519+ML-KEM wrapping;
- identity signatures and whether hybrid signatures are necessary;
- backup/recovery of larger keys.

Keep bulk object encryption symmetric and algorithm-agile. Do not couple post-quantum rollout to global deduplication.

## 6. Semantic representation research

## R15 — raw versus canonical archives

**Question**

When can extracted files and a recipe reproduce an original archive bit-for-bit?

**Work plan**

- enumerate ZIP and tar metadata that affects bytes;
- test compressor/version/platform determinism;
- identify formats with embedded signatures, offsets, or nondeterministic ordering;
- define `bit_exact`, `semantically_equivalent`, and `not_reconstructible` outcomes;
- always retain raw bytes under archival policy unless explicitly discarded;
- fuzz and sandbox parsers.

**Success criterion**

Semantic adapters improve dedup or access without making a false preservation claim.

## R21 — virtual plaintext-backed shards

**Question**

Does allowing a plaintext owner to regenerate a deterministic encrypted systematic fragment materially reduce remote redundancy cost?

**Prerequisites**

- completed R8 deterministic object format;
- claim protocol from R5;
- reliability simulator from R2;
- key availability and revocation model;
- clear privacy treatment for possession claims.

**Compare**

- hard remote 6+3;
- hard remote 6+2 plus soft sources;
- virtual data fragments plus remote parity;
- delayed repair while verified sources are online.

Account for the information-theoretic floor: if all plaintext originals can disappear, autonomous remote storage must still contain a complete decode set for the promised recovery scenario.

## 7. Decentralization research

## R20 — decentralized catalog

**Question**

Can catalog authority be decentralized without losing atomic Put/List/Delete and safe GC?

Potential lines:

- per-namespace consensus groups;
- Byzantine or witness-signed append logs;
- CRDT catalog with monotonic tombstones and restricted operations;
- deterministic placement plus exception logs;
- client-maintained capability roots rather than global listing.

This is P2 because the system gains most storage decentralization benefits with a small metadata quorum. First measure whether controller availability or trust is actually the limiting concern.

## R23 — permissionless nodes and Sybil resistance

**Question**

How are independent failure domains established when anyone can create nodes?

Requires research into:

- identity and stake/reputation;
- proof of distinct administration/location;
- payment and repair incentives;
- availability commitments and penalties;
- legal/abuse handling;
- traffic analysis and privacy;
- denial-of-service and storage amplification.

Do not treat self-declared node IDs as independent durability. Initial deployments remain permissioned.

## 8. Benchmark and simulation program

### Workload corpus

- millions of small files;
- photo/video libraries;
- source trees and package caches;
- VM and disk images with incremental changes;
- database dumps;
- ZIP/tar/OCI archives;
- multi-user common public files;
- highly private random data;
- sparse and zero-heavy files.

### Network profiles

- local LAN;
- home broadband with asymmetric upload;
- mobile/intermittent laptop;
- cross-continent WAN;
- one slow or malicious node;
- controller quorum partition;
- prolonged site outage.

### Fault matrix

- crash before/after each catalog commit step;
- corrupt fragment/header/descriptor;
- missing systematic fragment;
- simultaneous host/site loss;
- stale node inventory;
- node clock skew;
- lost controller quorum;
- old catalog checkpoint restore;
- mass endpoint deletion;
- removed member offline commit;
- VOPRF outage/key epoch change.

### Required outputs

- storage overhead;
- recovery probability under the stated model;
- time below target health;
- repair and degraded-read bandwidth;
- foreground latency during repair;
- catalog availability and commit latency;
- restore seed hit rate;
- synchronization conflict rate;
- dedup savings by realm/segment size;
- duplicate-oracle classifier accuracy;
- GC duration and false-reclaim safety.

## 9. Formal methods targets

Model-check at least:

1. logical blob Put and catalog visibility;
2. placement migration and drain;
3. tombstone propagation and offline-node return;
4. namespace reference accounting and GC;
5. membership epochs and removed offline writers;
6. concurrent heads and rollback/fork detection;
7. VOPRF key epochs and object-key recovery.

The objective is not to prove every implementation detail. It is to find invalid state transitions before they become on-disk format commitments.

## 10. ADR backlog

Suggested Architecture Decision Records:

1. Four-plane architecture and Track A/Track B split.
2. `repo/blob/distributed` boundary versus gateway deployment.
3. Initial catalog consistency model.
4. Stripe header and descriptor format.
5. Placement epoch and failure-domain policy.
6. Reed–Solomon baseline parameters.
7. Hard versus soft durability semantics.
8. Local reseed behavior for current Kopia packs.
9. Trusted multi-user opaque reference and reachability model.
10. v2 identity, device, and recovery model.
11. Encrypted namespace and commit DAG format.
12. Capability types and endpoint ransomware policy.
13. Dedup privacy modes and user-visible guarantees.
14. Canonical object representation.
15. VOPRF/POPRF service context and key lifecycle.
16. Full-upload versus proof-of-ownership protocol.
17. Quota/reference accounting and GC epochs.
18. Multi-writer merge and conflict semantics.
19. Archive representation adapter API.
20. Cryptographic and format agility policy.

## 11. Research traps to avoid

- Selecting `k,m` from independent-node math without a real failure-domain model.
- Calling a live plaintext file a durable shard without considering historical versions and offline state.
- Assuming current Kopia pack ciphertext is reproducible from one plaintext file.
- Treating a VOPRF as eliminating equality leakage or online guessing.
- Treating proof of ownership as a privacy mechanism; it is an ownership/bandwidth mechanism.
- Using a content hash as both locator and authority.
- Freezing canonical compression based on unspecified library output.
- Building a permissionless network before safe permissioned operation exists.
- Introducing MLS, CRDTs, threshold crypto, LRCs, and post-quantum primitives simultaneously.
- Optimizing away full duplicate uploads before defining the desired existence-oracle resistance.
- Implementing deterministic encryption without a dedicated cryptographic review.

## 12. Recommended immediate research sprint

The first sprint should produce evidence for five decisions:

1. Capture and replay Kopia blob/range traces to choose stripe-size candidates.
2. Build a churn/failure-domain simulator and compare initial Reed–Solomon policies.
3. Specify and model-check Put, tombstone, and GC state machines.
4. Prototype restore-only seeds and logical reseeding with unmodified/current Kopia formats.
5. Write the trusted multi-user opaque-reference/reachability ADR before designing v2 cryptography.

This sequence keeps the project moving while isolating the highest-risk cryptographic work behind measured product and storage semantics.

## References

- Kopia architecture: <https://kopia.io/docs/advanced/architecture/>
- Kopia `blob.Storage` semantics: <https://github.com/kopia/kopia/blob/master/repo/blob/storage.go>
- Tahoe-LAFS servers of happiness: <https://tahoe-lafs.readthedocs.io/en/latest/specifications/servers-of-happiness.html>
- Mutagen synchronization: <https://mutagen.io/documentation/synchronization>
- desync seeds: <https://github.com/folbricht/desync>
- CRUSH publications: <https://ceph.io/en/news/publications/>
- Azure Local Reconstruction Codes: <https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang>
- Practical wide LRC considerations: <https://www.usenix.org/conference/fast23/presentation/kadekodi>
- RFC 9497: <https://www.rfc-editor.org/info/rfc9497/>
- DupLESS: <https://www.usenix.org/conference/usenixsecurity13/technical-sessions/presentation/bellare>
- Proofs of ownership: <https://research.ibm.com/publications/proofs-of-ownership-in-remote-storage-systems>
