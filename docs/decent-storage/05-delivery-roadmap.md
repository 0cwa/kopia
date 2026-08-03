# Delivery roadmap, tradeoffs, and validation

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `architecture-roadmap.md` sections 9–12.

## 9. Roadmap

## Phase 0 — specifications, simulator, and threat model

**Deliverables**

- storage-node, shard, descriptor, catalog, and tombstone format drafts;
- explicit threat model and trust profiles;
- workload collector for Kopia blob sizes and range reads;
- churn/failure-domain simulator;
- first ADRs for Track A versus Track B boundaries;
- conformance harness for `blob.Storage` semantics.

**Exit criteria**

- recoverability and commit invariants are executable tests;
- stripe-size and initial `k,m` parameters are justified by traces and simulation;
- catalog recovery and tombstone behavior are documented;
- security reviewers agree that no Track A feature is described as providing Track B confidentiality.

## Phase 1 — distributed gateway prototype

Prefer an S3-compatible gateway first so unmodified Kopia can exercise the backend.

**Deliverables**

- systematic Reed–Solomon stripes;
- authenticated shard headers;
- stable node identities and topology labels;
- single-leader catalog with replicated database/backups;
- Put/Get/range/List/Delete semantics;
- temporary-upload cleanup;
- initial CLI and metrics.

**Exit criteria**

- passes the blob storage conformance suite;
- survives injected node loss up to policy threshold;
- no partial blob is observable after crashes at any write step;
- catalog can be reconstructed from checkpoints plus shard metadata in a disaster drill;
- performance overhead is measured against direct S3/filesystem Kopia.

## Phase 2 — production distributed blob provider

Move the proven gateway logic into `repo/blob/distributed` or retain the gateway if operational separation is advantageous.

**Deliverables**

- 3/5-node strongly consistent catalog;
- placement epochs and deterministic weighted rules;
- direct client-to-node data path where safe;
- repair, drain, and rebalance;
- sampled audits and signed inventory;
- retention/tombstone propagation;
- stronger metadata policy classes;
- capacity and repair budgeting.

**Exit criteria**

- rolling upgrade and node-map changes preserve availability;
- correlated failure-domain tests behave as policy predicts;
- drain/rebalance never drops below hard floor;
- stale/offline nodes cannot resurrect deleted generations;
- sustained repair does not violate foreground latency targets.

## Phase 3 — local seed and assisted-recovery agent

**Deliverables**

- watcher-assisted scans with periodic full verification;
- chunk indexes compatible with logical snapshot contents;
- restore from local seeds and reflinks where available;
- signed expiring availability claims;
- logical reseed workflow through an authorized Kopia client;
- hard versus soft durability reporting.

**Exit criteria**

- changed files invalidate claims correctly;
- old snapshots never depend on missing current bytes;
- false/stale claims cannot reduce the configured hard floor;
- restore bandwidth reduction is measured on VM images, archives, and large edited files.

## Phase 4 — trusted multi-user server

**Deliverables**

- groups and shared-folder manifests;
- principal-scoped opaque references;
- content reachability and authorization index;
- append/write/admin/maintenance role separation;
- per-principal logical quotas and reference counts;
- hidden dedup outcome;
- shared-folder commit DAG and one-writer policy.

**Exit criteria**

- a user cannot retrieve content solely by knowing a global ID;
- reference/GC races are model-tested;
- endpoint credential compromise cannot delete protected history;
- sharing and revocation UX is validated before blind crypto is frozen.

## Phase 5 — v2 encrypted namespaces and object format

**Deliverables**

- versioned canonical object representation;
- client-side encryption and encrypted directory DAG;
- user/device identities and key recovery design;
- HPKE key envelopes;
- random `ObjectRef` API;
- signed namespace commits and rollback detection;
- read/append/write/admin/verify/repair capabilities;
- import/export bridge from Track A.

**Exit criteria**

- coordinator and storage nodes cannot decrypt user data or metadata beyond documented leakage;
- loss/revocation/recovery scenarios have tested procedures;
- deterministic-object mode has an external cryptographic review before use;
- namespace rollback and fork attacks are detectable.

## Phase 6 — multiple writers

**Deliverables**

- multi-device signing keys;
- concurrent heads and common-ancestor discovery;
- three-way directory and file reconciliation;
- staged atomic materialization;
- explicit conflict objects;
- optional MLS-based large-group epoch distribution.

**Exit criteria**

- concurrent non-conflicting updates converge;
- genuine conflicts preserve all versions;
- offline writers cannot silently overwrite newer history;
- membership changes cannot be confused with ordinary file commits.

## Phase 7 — optional global deduplication

**Deliverables**

- independent VOPRF/POPRF service;
- batching and rate limits;
- canonical large-segment profile;
- reviewed MLE construction;
- full-upload anti-oracle flow;
- random ref mapping and logical billing;
- optional proof-of-ownership mode;
- VOPRF key backup/refresh and epoch migration plan.

**Exit criteria**

- external protocol and cryptographic audit;
- duplicate/new uploads are indistinguishable within specified timing/traffic bounds in privacy mode;
- storage-only compromise does not enable practical offline dictionary attacks without the key service;
- online guessing controls and abuse telemetry are operational;
- restore does not depend on VOPRF availability because object keys are wrapped in namespaces.

## Phase 8 — semantic representation adapters

**Deliverables**

- sandboxed adapter API;
- raw plus canonical-tree linking;
- ZIP/tar proof of concept;
- bit-exact reconstruction reporting;
- parser resource limits and fuzzing.

**Exit criteria**

- adapters never replace preservation of raw bytes unless policy explicitly permits it;
- untrusted archives cannot escape sandbox or cause unbounded expansion;
- semantic dedup savings justify added complexity on representative datasets.

## 10. Tradeoff summary

| Decision | Preferred default | Alternative | Why the default wins initially |
|---|---|---|---|
| Backend entry point | S3 gateway, then native provider | immediate deep Kopia fork | proves semantics with lower coupling |
| Physical coding | systematic Reed–Solomon | LRC/MSR/regenerating code | mature, simple, range-friendly; optimize after measurement |
| Catalog | small strongly consistent controller set | DHT/eventual metadata | matches Kopia listing/atomicity and simplifies GC |
| Personal-device role | opportunistic/cache/soft source | hard required shard | intermittent availability should not break hard SLOs |
| Plaintext integration | restore seed and expiring claim | literal current-pack shard | current pack bytes are not reproducible from one file |
| Sharing first release | trusted server | blind E2EE immediately | validates semantics and reachability before freezing crypto format |
| Namespace locator | random ObjectRef | global content hash | avoids locator-as-authority and reduces equality exposure |
| Global dedup privacy | full ciphertext upload | client-side skip with PoW | removes the clearest duplicate-existence signal |
| Global dedup granularity | file/large segment | small content chunk | fewer VOPRF calls and less fragment-equality leakage |
| Group keys | HPKE per member for small groups | MLS always | less state and audit surface until group churn warrants MLS |
| Post-quantum | algorithm agility; hybrid wrapping later | PQ on every content operation | bulk symmetric crypto is already efficient and the hot path should remain simple |

## 11. Cross-cutting validation

### Correctness

- property tests for stripe/range mapping and reconstruction;
- fault injection at every Put/Delete commit boundary;
- model checking for catalog generations, tombstones, and GC;
- fuzzing of shard descriptors, namespace commits, and representation adapters;
- deterministic test vectors for every v2 cryptographic format.

### Security

- independent threat-model review before each trust-boundary change;
- external cryptographic review before deterministic encryption or VOPRF production use;
- key compromise, device loss, rollback, malicious-node, and collusion exercises;
- constant-response and timing tests for duplicate/new upload flows;
- audit log integrity and key transparency for identity/device changes.

### Performance

Measure separately:

- healthy and degraded range-read latency;
- write amplification and storage overhead;
- repair bandwidth, I/O, and time-to-safe-state;
- catalog quorum latency and outage tolerance;
- seed hit rate and restore bandwidth reduction;
- synchronization scan/stage/apply costs;
- VOPRF batching, canonicalization, and upload overhead;
- GC and reference traversal at repository scale.

### Operations

- bootstrap and disaster recovery without a surviving controller;
- controller quorum replacement;
- node enrollment, drain, loss, and ownership transfer;
- repository/key backup and recovery;
- rolling format and algorithm upgrades;
- observability that does not leak plaintext names or sharing graph.

## 12. Definition of an acceptable end state

The architecture is successful when:

- encrypted backup blobs can survive configured independent device/site failures;
- hard durability can be explained without counting live working folders;
- local files materially accelerate restore and repair when available;
- users can share a folder without obtaining repository-wide keys or global deletion authority;
- multi-writer conflicts preserve data and immutable history;
- private mode reveals no cross-user equality;
- global dedup mode clearly documents equality leakage and does not expose an easy hit oracle to clients;
- storage and repair nodes can operate without plaintext keys;
- every format and trust transition has a migration and recovery story.

## References

- Kopia architecture: <https://kopia.io/docs/advanced/architecture/>
- Kopia blob storage interface: <https://github.com/kopia/kopia/blob/master/repo/blob/storage.go>
- Kopia repository server: <https://kopia.io/docs/repository-server/>
- Tahoe-LAFS file encoding: <https://tahoe-lafs.readthedocs.io/en/stable/specifications/file-encoding.html>
- Tahoe-LAFS servers of happiness: <https://tahoe-lafs.readthedocs.io/en/latest/specifications/servers-of-happiness.html>
- Tahoe-LAFS write coordination: <https://tahoe-lafs.readthedocs.io/en/latest/write_coordination.html>
- Mutagen synchronization architecture: <https://mutagen.io/documentation/synchronization>
- desync seeds and chunk indexes: <https://github.com/folbricht/desync>
- Ceph/CRUSH publications: <https://ceph.io/en/news/publications/>
- Local Reconstruction Codes: <https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang>
- RFC 9180, HPKE: <https://www.rfc-editor.org/info/rfc9180/>
- RFC 9420, MLS: <https://www.rfc-editor.org/info/rfc9420/>
- RFC 9497, OPRF/VOPRF/POPRF: <https://www.rfc-editor.org/info/rfc9497/>
- DupLESS: <https://www.usenix.org/conference/usenixsecurity13/technical-sessions/presentation/bellare>
- Proofs of ownership: <https://research.ibm.com/publications/proofs-of-ownership-in-remote-storage-systems>
