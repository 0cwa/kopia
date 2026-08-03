# Research backlog: storage, durability, and synchronization

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `research-backlog.md` sections 1–4.

**Status:** Proposed research program  
**Purpose:** Identify the questions that must be answered before formats and security promises are frozen.

## 1. Prioritization

- **P0 — blocks the next implementation phase.** Must be answered with experiments, models, or expert review.
- **P1 — important for production quality.** Prototype can proceed with conservative defaults.
- **P2 — optimization or future trust model.** Defer until measurements justify complexity.

Each item should produce an Architecture Decision Record (ADR), test artifacts, and reproducible data where possible.

## 2. Research matrix

| ID | Priority | Question | Blocks |
|---|---:|---|---|
| R1 | P0 | What stripe size and range layout fit actual Kopia access patterns? | distributed gateway format |
| R2 | P0 | What failure/churn model represents target devices and correlated domains? | `k,m`, commit, placement policy |
| R3 | P0 | Which catalog consistency architecture meets Kopia semantics under outages? | production backend |
| R4 | P0 | How are tombstones, offline nodes, and GC made monotonic? | safe deletion |
| R5 | P0 | What exact hard/soft durability policy and claim protocol is defensible? | plaintext integration |
| R6 | P0 | Can current Kopia logical contents be efficiently reseeded without recreating packs? | seed agent |
| R7 | P0 | What authorization/reachability index scales for trusted multi-user service? | sharing and GC |
| R8 | P0 | What v2 canonical object and deterministic encryption construction is safe? | global dedup/virtual shards |
| R9 | P0 | What duplicate-upload protocol meets the claimed privacy profile? | global dedup release |
| R10 | P0 | How are namespace rollback, forks, and offline writers detected? | blind multi-user format |
| R11 | P1 | Reed–Solomon versus LRC/regenerating codes under measured repair workload? | code optimization |
| R12 | P1 | Which large-segment boundaries maximize global savings with acceptable leakage? | dedup efficiency |
| R13 | P1 | HPKE-per-member versus MLS at realistic group sizes/churn? | group key distribution |
| R14 | P1 | What filesystem merge semantics are portable and data-preserving? | multi-writer sync |
| R15 | P1 | How should archive/raw/canonical representations be linked and verified? | semantic adapters |
| R16 | P1 | What audit scheme detects withholding/corruption at acceptable cost? | untrusted storage operation |
| R17 | P1 | What key recovery model balances usability and true end-to-end control? | user identity/recovery |
| R18 | P1 | How should logical quotas and billing avoid dedup side channels? | multi-tenant service |
| R19 | P2 | Does threshold VOPRF materially improve the deployed trust model? | advanced global dedup |
| R20 | P2 | Is a decentralized/eventual catalog worth the consistency complexity? | controller decentralization |
| R21 | P2 | Do virtual plaintext-backed shards provide enough savings to justify a new codeword format? | advanced assisted durability |
| R22 | P2 | What post-quantum hybrid envelope/signature profile should become default? | long-term crypto agility |
| R23 | P2 | Can a permissionless node network resist Sybil and economic abuse? | open public network |

## 3. Storage and durability research

## R1 — Kopia I/O trace and stripe sizing

**Question**

How large should a physical stripe be so healthy range reads remain efficient while metadata, request counts, and degraded reconstruction stay acceptable?

**Why unresolved**

Kopia packs are tens of megabytes and content reads are ranges identified by pack offset and length. The optimal stripe depends on the distribution of packed object sizes, range locality, cache behavior, network RTT, node throughput, and degraded-read rate.

**Experiment**

- Instrument a Kopia blob backend to record blob sizes, range offsets/lengths, concurrency, read-after-write timing, and maintenance scans.
- Replay traces through a simulator for stripe sizes such as 256 KiB, 512 KiB, 1 MiB, 2 MiB, 4 MiB, and 8 MiB.
- Test healthy systematic reads and one-/multi-fragment degradation over LAN, residential WAN, and high-latency links.
- Include metadata packs and indexes separately from bulk packs.

**Metrics**

- bytes fetched per requested byte;
- number of node requests;
- p50/p95/p99 latency;
- decode CPU and memory;
- repair amplification;
- descriptor/catalog size.

**Decision rule**

Choose a conservative stripe size that keeps healthy read amplification near one for common ranges and bounds degraded-read amplification. Permit future codec/stripe versions per blob policy.

## R2 — target failure and churn model

**Question**

What independent and correlated failure probabilities should drive placement and `k,m`?

**Why unresolved**

Disk independence assumptions are not credible for personal devices. Host failure, power loss, ISP outage, household disaster, owner departure, software bugs, and long offline periods are correlated.

**Experiment/model**

- Define target deployments: single household, family across households, small organization, geo-distributed cooperative.
- Collect node online/offline histories and repair bandwidth constraints.
- Model failure domains explicitly: device, host, owner, power/network, site, administrative key.
- Simulate permanent loss, transient outage, and malicious withholding.
- Compare policies such as 4+2, 6+3, 8+4, full metadata replicas, and mixed core/opportunistic nodes.

**Output**

A service-level policy language:

```text
bulk: decode from any 6 of 9, across >=3 households and >=2 sites
metadata: 3 full replicas across 3 sites plus 6+3 coded copy
working-copy claims: informational, never below hard floor
```

Do not publish reliability numbers based only on independent identical node probability.

## R3 — catalog consistency and availability

**Question**

Can a three/five-controller replicated catalog provide acceptable availability for the target home/cooperative network, and what operations remain available without quorum?

**Options**

- Raft-like replicated state machine;
- transactional SQL/KV database with consensus;
- append-only signed log plus materialized index;
- single leader with asynchronous replicas for prototype;
- per-namespace optimistic commits plus strong blob catalog.

**Technical challenges**

- Kopia requires authoritative `ListBlobs` and immediate visibility after Put.
- Offline personal devices cannot form a reliable consensus quorum.
- Cross-site synchronous writes add latency.
- Disaster recovery must avoid accepting two divergent catalog generations.

**Experiment**

- Model Put/Delete/repair under controller partitions and crash/restart.
- Measure WAN commit latency and read availability.
- Define read-only/degraded behavior without quorum.
- Prototype checkpoint export and catalog reconstruction.
- Specify controller replacement and epoch fencing.

**Recommended hypothesis**

Stable controllers are a distinct role from arbitrary storage devices. A small quorum controls metadata while data shards remain broadly distributed.

## R4 — tombstones, generations, and GC

**Question**

How can offline nodes and clients never resurrect a deleted object or reference?

**Risks**

- node reconnects with old fragment inventory;
- offline writer commits a namespace parent that references expired objects;
- GC races with sharing/reference creation;
- catalog restoration from an old checkpoint loses tombstones;
- object ID reuse confuses generations.

**Method**

Create a formal state-machine model, preferably TLA+ or an equivalent executable model, covering:

```text
upload -> stage -> commit -> reference -> share -> unshare
-> retention expiry -> tombstone -> grace -> reclaim
```

Include crashes and duplicated/reordered messages. Require generation fencing and monotonic tombstone/checkpoint epochs.

**Success criterion**

No reachable retained commit loses data; no tombstoned generation reappears after any allowed recovery sequence.

## R5 — hard and soft durability semantics

**Question**

How should availability claims affect repair without overstating safety?

**Research areas**

- claim granularity: file, logical object, chunk ranges, or whole snapshot;
- lease duration and invalidation latency;
- proof/challenge frequency;
- malicious or buggy claimant behavior;
- authorization needed to use plaintext as a repair source;
- privacy leakage from publishing file/chunk possession;
- whether claims are visible only to the owning namespace.

**Recommended policy hypothesis**

Claims influence repair scheduling and assisted-recovery reporting, never the archival hard floor. An explicitly cheaper user-selected tier may delay repair based on claims, but must label the resulting reduced guarantee.

## R6 — logical reseeding with current Kopia

**Question**

What is the lowest-coupling way for an authorized device with plaintext chunks to replace lost logical content when existing physical pack stripes cannot be regenerated exactly?

**Candidate approaches**

1. Restore-only seed: no repository mutation.
2. Re-upload logical contents through normal Kopia writer, creating new packs/index entries.
3. Add a maintenance API that validates content ID and writes replacement content without creating a new snapshot.
4. Preserve enough pack construction metadata to reproduce selected packs—not recommended unless proven simple.

**Experiment**

Induce loss above physical repair capability while retaining plaintext sources. Measure whether normal content rewrite can repair references safely and how indexes/GC react. Verify that content ID and compression/encryption rules remain valid.

**Likely result**

Logical reseeding should create valid new physical storage and update indexes, not attempt exact old-pack reproduction.

## R11 — erasure-code family

**Question**

When do Local Reconstruction Codes or regenerating/MSR codes outperform systematic Reed–Solomon for this workload?

**Baseline**

Use systematic Reed–Solomon first.

**Measure**

- common single-fragment repair bandwidth and I/O;
- degraded range-read amplification;
- code width and correlated outage exposure;
- implementation maturity and vectorization;
- memory on low-power devices;
- format complexity and future interoperability.

LRCs are promising when repair traffic dominates. Wide or regenerating codes can worsen partial-read behavior and operational reliability if codewords span too many failure events. Adopt only after measured benefit.

## R16 — audit scheme

**Question**

What level of possession checking is required for storage nodes?

**Candidates**

- signed inventory only;
- random authenticated range reads;
- Merkle block challenges;
- proof-of-retrievability scheme;
- periodic full scrub.

**Decision factors**

- node trust and incentives;
- storage and network cost;
- acceptable detection delay;
- whether nodes are merely unreliable or adversarial;
- fragment size and Merkle metadata overhead.

Begin with random authenticated ranges plus scheduled full scrubs. Formal PoR is P2 unless storage is purchased from mutually untrusted peers.

## 4. Namespace, access-control, and synchronization research

## R7 — trusted-server reachability index

**Question**

How can a server efficiently prove that a requested content object is reachable from a principal's authorized roots?

**Options**

- materialized principal → object reference table;
- per-snapshot transitive closure;
- capability-scoped random references created during upload/commit;
- namespace DAG with refcounted edges;
- lazy traversal plus signed authorization cache.

**Challenges**

- snapshots share large subgraphs;
- group membership changes;
- object references are immutable but roots and retention evolve;
- GC needs global reference safety;
- malicious clients may submit manifests referencing objects they never uploaded or owned.

**Prototype**

Use opaque references and validate every commit edge. Materialize reference ownership/refcounts with generation epochs. Benchmark repositories with millions/billions of chunks and many shared snapshots.

## R10 — rollback, fork, and offline writers

**Question**

How do devices distinguish legitimate concurrent history from malicious rollback/equivocation?

**Candidate mechanisms**

- signed parent-linked commit DAG;
- per-device monotonic sequence;
- membership epoch binding;
- retained last-seen heads on clients;
- cross-device gossip;
- external witness or transparency log;
- account recovery rules for lost last-seen state.

**Test scenarios**

- coordinator shows two devices different membership histories;
- removed writer submits an offline commit under old epoch;
- user restores an old local device backup;
- account recovery intentionally rolls state forward from partial evidence;
- concurrent folder changes under same and different parents.

**Success criterion**

The system either proves a valid descendant/merge or surfaces a fork requiring explicit resolution; it never silently accepts rollback.

## R13 — HPKE versus MLS group management

**Question**

At what group size and churn does MLS reduce cost enough to justify its state machine and audit surface?

**Benchmark**

- 2, 5, 20, 100, and 1,000 members;
- per-user versus per-device recipients;
- offline device catch-up;
- frequent device revocation;
- many shared folders with overlapping memberships;
- envelope size, update size, and client state.

**Likely policy**

HPKE per recipient for small/static groups. MLS only for large/high-churn groups, and only as epoch-secret distribution.

## R14 — portable multi-writer semantics

**Question**

Which file and metadata conflicts can be merged automatically without data loss across Linux, macOS, Windows, mobile, and network filesystems?

**Research dimensions**

- file identity across rename/move;
- case-folding collisions;
- Unicode normalization;
- permission/xattr semantics;
- atomic application and crash recovery;
- symlink/special-file handling;
- partially written files and application locks;
- deletion versus modification;
- large binary files and sparse extents.

**Recommended staged scope**

1. one writer, many readers;
2. one user, multiple devices;
3. multiple writers, explicit conflict copies;
4. format-specific CRDT adapters.

Use a remembered common ancestor and staging. Preserve all conflicting bytes before optimizing conflict UX.

## R17 — account and key recovery

**Question**

How can users recover without covertly granting the service universal decryption power?

**Options**

- printed/offline recovery key;
- encrypted recovery bundle under password-hard KDF;
- threshold social/device recovery;
- enterprise escrow;
- service reset that intentionally loses old encrypted data.

**Research**

Usability tests, device-loss drills, compromise analysis, and clear product messaging. Recovery must include identity and namespace fork protection, not only decryption keys.
