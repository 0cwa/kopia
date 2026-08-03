# Deduplication privacy and security operations

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `security-and-privacy.md` sections 11–20.

## 11. Deduplication privacy model

## 11.1 Fundamental limitation

Physical deduplication means some component determines that two objects are equal. Equality cannot be hidden from every component while retaining cheap global physical deduplication. The design goal is to:

- constrain which party learns equality;
- prevent ordinary users from querying equality cheaply;
- prevent storage-only attackers from testing low-entropy candidate plaintexts offline;
- prevent equality from becoming download authorization;
- document remaining size, timing, and access-pattern leakage.

## 11.2 Private mode

- Generate random object keys.
- Encrypt independently per user or folder.
- Deduplicate only where the namespace intentionally reuses the same ciphertext/key.
- Best default for sensitive data.

Potential optimization: within one private namespace, retain a keyed content index so the client can reuse an existing encrypted object without revealing a global hash.

## 11.3 Team-realm mode

A family or organization establishes a dedup realm. It may use a shared convergence secret or a realm-specific VOPRF/POPRF context.

Tradeoff:

- good storage savings inside a trusted group;
- every member with observable tags/ciphertext and access to the derivation service has a larger candidate-testing surface;
- realm compromise does not automatically affect private/global realms if domain separation is correct.

## 11.4 VOPRF-assisted global mode

Protocol sketch:

```text
R = CanonicalEncode(object, format_version)
x = Hash(domain || realm || format_version || size_class || R)
(blind, blinded) = VOPRF.Blind(x)
evaluated, proof = KeyService.Evaluate(blinded, public_context)
z = VOPRF.Finalize(x, blind, evaluated, proof)
K = HKDF(z, object_context)
C = MLE_Encrypt(K, R, associated_data)
```

Security benefits:

- storage-only compromise does not give the message-derived key;
- key service does not see the unblinded object hash in a correct OPRF execution;
- verifiability detects service key substitution/equivocation relative to the pinned public key.

Remaining risks:

- authorized clients can make online guesses;
- coordinator sees equal ciphertext/global IDs;
- sizes and timing may reveal candidates;
- coordinator/key-service collusion is stronger;
- compromise of the effective PRF key threatens the dedup domain;
- deterministic encryption security depends on the exact MLE construction and canonicalization.

Controls:

- authenticated accounts and per-realm authorization;
- rate limits and abuse analytics;
- stricter handling of small/low-entropy objects;
- batch-size caps;
- independent key-service operator;
- HSM or threshold protection of the effective PRF key;
- pinned VOPRF public key and signed epoch metadata.

## 11.5 Canonicalization

Deduplication only works when clients produce identical `R`. Canonicalization must bind:

- object or segment boundaries;
- compression algorithm and exact version/profile;
- metadata inclusion/exclusion;
- padding and length encoding;
- representation type;
- dedup realm;
- cryptographic format version.

Any change creates a new dedup domain or requires migration. “Canonical compression” must be tested across architectures and versions; relying on unspecified library output is unsafe.

## 11.6 Deterministic encryption

The deterministic/MLE object scheme is the highest-risk cryptographic element. Requirements include:

- deterministic ciphertext for identical canonical representation in one realm;
- integrity and strong misuse resistance;
- domain separation between object types and versions;
- no fixed-nonce misuse of an ordinary nonce-based AEAD;
- security analysis for message-derived keys and associated data;
- public test vectors and independent review.

A candidate construction must be selected through a dedicated cryptographic design review. This roadmap intentionally does not name an improvised cipher suite.

## 12. Deduplication upload protocols

### Privacy-preserving full-upload mode

1. Derive and encrypt locally.
2. Upload complete ciphertext to a random staging reference.
3. Server validates the complete object.
4. Server maps to an existing or new internal global object.
5. Server creates a random authorized `ObjectRef`.
6. Server returns the same protocol shape and logical accounting either way.

Benefits:

- no explicit hit/miss signal;
- no challenge that only occurs for existing objects;
- possession is demonstrated by sending the complete ciphertext.

Costs:

- duplicate upload bandwidth and ingress processing remain;
- sophisticated timing/traffic analysis still needs padding/bucketing if in scope.

### Proof-of-ownership/upload-skip mode

1. Client supplies an opaque dedup token or initiates candidate upload.
2. Server sends unpredictable challenges derived after commitment.
3. Client proves possession of sampled encoded blocks/Merkle leaves.
4. Server attaches a new authorized reference without complete upload.

Benefits:

- saves duplicate upload bandwidth;
- prevents hash-only ownership claims when correctly constructed.

Costs:

- reveals or strongly suggests object existence;
- small objects provide weak sampling value;
- challenge state and replay resistance add complexity;
- it does not prevent candidate guessing by a client that actually has the candidate file.

Use only as explicit optimization for large objects or trusted realms.

## 13. Opaque reference service

The coordinator maintains:

```text
ObjectRef -> GlobalObjectID
             namespace/principal scope
             role and operations
             creation/reference epoch
             logical byte accounting
             retention and expiry
```

`ObjectRef` values are random and unguessable. They can be rotated without moving physical data. A namespace manifest stores only authorized refs and encrypted key envelopes.

Reference creation rules:

- allowed after a full upload;
- allowed after valid proof of ownership in the weaker mode;
- allowed by sharing an existing authorized namespace capability;
- never allowed merely because a client supplies a global hash.

## 14. Quotas and side channels

To avoid exposing dedup hits:

- charge logical bytes, not incremental physical bytes;
- do not show per-user dedup savings in global mode;
- bucket or normalize completion responses;
- avoid different errors for new versus existing objects;
- do not reveal global owner/reference counts;
- route storage through the coordinator if physical-node selection would reveal equality;
- consider padding objects into size classes where metadata privacy warrants the overhead.

Perfect traffic-analysis resistance is not an initial goal. The selected leakage profile should explicitly state whether the server learns exact encrypted lengths, access frequency, and online status.

## 15. Storage-node integrity and availability

Storage nodes receive no plaintext key in blind mode. They store authenticated fragments and may receive capability-scoped requests.

Controls:

- mutually authenticated transport;
- signed node enrollment and topology labels;
- per-fragment hash/Merkle verification;
- unpredictable audit challenges;
- signed inventories bound to catalog epoch;
- repair from any valid decode set;
- controller skepticism of node-reported failure domains;
- administrative attestation or external verification for owner/site labels.

In a permissioned deployment, identity vetting may be enough. An open deployment needs Sybil resistance; self-asserted “different household” labels are not security boundaries.

## 16. Ransomware and deletion safety

Endpoint compromise is assumed.

Recommended controls:

- endpoint capabilities are append-only or folder-scoped write by default;
- retention deletion requires a separate offline/admin key or quorum;
- immutable object generations and delayed GC;
- object lock/WORM integration where backend supports it;
- anomaly detection for mass changes/deletes;
- recovery UI that restores prior namespace heads;
- no automatic propagation of suspicious bulk deletion without policy confirmation;
- signed audit trail of membership, retention, and deletion actions.

Synchronization safety and archival retention are independent. A sync engine may mirror a deletion while the archive retains prior commits.

## 17. Garbage collection and reference safety

Global deduplication makes GC security-sensitive. A physical object is reclaimable only when:

- no live authorized namespace reference exists;
- no retained commit/snapshot reaches it;
- no in-flight transaction may publish a reference;
- the reference epoch and grace period have elapsed;
- catalog checkpoints and offline writers cannot legally resurrect the reference;
- tombstone policy permits deletion.

Use generation/epoch-based reference accounting rather than a simple mutable integer alone. Model-check races among upload, share, unshare, retention expiry, offline commit, and GC.

## 18. Cryptographic algorithm guidance

### Appropriate now

- modern AEAD for randomly keyed private objects;
- HKDF and explicit domain separation;
- Ed25519 or another well-supported signature for early commit formats;
- HPKE for recipient key envelopes;
- Merkle trees for object/shard verification;
- RFC 9497 VOPRF/POPRF for the key-service primitive;
- systematic Reed–Solomon for physical redundancy.

### Requires dedicated review

- deterministic/message-locked encryption construction;
- threshold VOPRF and distributed key generation/refresh;
- private dedup tags and proof-of-ownership integration;
- cryptographic subtree/key derivation and attenuation;
- post-quantum hybrid envelope formats;
- anonymous or privacy-preserving quota mechanisms.

### Defer

- ORAM/PIR for ordinary reads;
- attribute-based encryption;
- generic proxy re-encryption;
- homomorphic encryption;
- blockchain consensus;
- post-quantum operations for every data block.

## 19. Post-quantum posture

Bulk stored data should remain protected primarily with strong symmetric cryptography. Public-key operations are concentrated in:

- device enrollment;
- folder/share key wrapping;
- identity and commit signatures;
- recovery and long-lived archive key protection.

The format should include algorithm identifiers and support hybrid classical/post-quantum envelopes. NIST ML-KEM and ML-DSA are candidates for long-lived key establishment/signatures, but implementation choices should follow mature libraries, current errata, interoperability needs, and independent review. Do not make experimental post-quantum OPRFs a dependency of the first global-dedup release.

## 20. Required security reviews

Before production milestones:

1. **Track A distributed storage review:** commit atomicity, shard integrity, catalog recovery, tombstones, node authentication.
2. **Trusted multi-user review:** reachability authorization, reference attachment, quotas, GC races, endpoint roles.
3. **v2 namespace review:** key hierarchy, capability attenuation, commit signatures, rollback/fork detection, recovery.
4. **Deterministic object review:** canonicalization and MLE construction.
5. **VOPRF protocol review:** context binding, key lifecycle, batching, abuse controls, full-upload indistinguishability.
6. **Multi-writer review:** membership epochs, offline commits, conflicts, and malicious writer behavior.
7. **Representation adapter review:** sandbox boundaries and parser security.

## References

- Kopia Repository Server security model: <https://kopia.io/docs/repository-server/>
- Tahoe-LAFS encoding and capabilities: <https://tahoe-lafs.readthedocs.io/en/stable/specifications/file-encoding.html>
- RFC 9180, HPKE: <https://www.rfc-editor.org/info/rfc9180/>
- RFC 9420, MLS: <https://www.rfc-editor.org/info/rfc9420/>
- RFC 9497, OPRF/VOPRF/POPRF: <https://www.rfc-editor.org/info/rfc9497/>
- DupLESS: <https://www.usenix.org/conference/usenixsecurity13/technical-sessions/presentation/bellare>
- Proofs of ownership: <https://research.ibm.com/publications/proofs-of-ownership-in-remote-storage-systems>
- Peergos architecture examples: <https://peergos.org/posts/dev-update>
- NIST FIPS 203, ML-KEM: <https://csrc.nist.gov/pubs/fips/203/final>
- NIST FIPS 204, ML-DSA: <https://csrc.nist.gov/pubs/fips/204/final>
