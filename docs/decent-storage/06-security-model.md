# Security model, identity, and sharing

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `security-and-privacy.md` sections 1–10.

**Status:** Proposed security architecture  
**Scope:** Distributed durability, multi-user namespaces, shared folders, deduplication, device agents, and coordinator/storage trust.

## 1. Security posture

This system cannot offer one undifferentiated “secure” mode. It has several trust profiles and explicit leakage choices.

### Profile A: trusted personal repository

- Repository password/keys are shared among the user's devices.
- Coordinator or Kopia server may decrypt content.
- Main threats are storage-node compromise, device loss, corruption, and ransomware.
- Deduplication can span the repository.

### Profile B: trusted multi-user service

- Users distrust one another but trust the server operator.
- Server can decrypt content and enforce per-user/share reachability.
- Conventional ACLs and opaque references are sufficient.
- This is the lowest-complexity shared-folder product.

### Profile C: blind coordinator

- Coordinator and storage nodes must not learn plaintext content, filenames, directory structure, or share keys.
- Clients encrypt and sign before upload.
- Coordinator still learns account relationships, access timing, object sizes unless padded, and internal ciphertext equality when global deduplication is enabled.

### Profile D: split-trust global deduplication

- Same as Profile C, plus an independently operated VOPRF/POPRF service.
- Storage-only compromise lacks message-derived keys.
- Key-service-only compromise lacks ciphertext, object ownership, and namespace metadata.
- Coordinator/key-service collusion remains a stronger threat and must be documented.

## 2. Threat actors

| Actor | Capabilities considered |
|---|---|
| Curious storage node | observes its shards, sizes, timing, client/node addresses; may return corrupt or stale bytes |
| Malicious storage node | deletes data, lies about inventory, withholds reads, replays old fragments |
| Malicious user | uploads chosen files, guesses candidate content, probes APIs, attempts unauthorized references or downloads |
| Compromised endpoint | has that device's plaintext, cached keys, and scoped capabilities; may attempt mass deletion/encryption |
| Malicious coordinator | rewrites metadata, withholds commits, serves stale heads, observes global equality and access patterns |
| Compromised VOPRF service | sees authenticated requests and blinded elements; may deny service, equivocate keys, or rate-limit selectively |
| Colluding services | combine coordinator ownership/equality information with key-service observations |
| Network adversary | observes or modifies traffic where transport encryption/authentication is absent |
| Sybil node operator | creates many apparent storage nodes/failure domains in an open network |
| Accidental operator | misconfigures placement, retention, membership, clocks, or key recovery |

Early deployments should be permissioned and mutually authenticated. Permissionless node admission requires a separate Sybil-resistance, accounting, abuse, and incentive design.

## 3. Protected assets

- plaintext file and metadata contents;
- filenames, paths, directory topology, file types, sizes, and modification times where the selected privacy profile promises protection;
- user identities, device relationships, and sharing graph;
- repository and namespace keys;
- historical snapshots and retention state;
- availability, integrity, and rollback resistance;
- deduplication equality and ownership information;
- quotas and billing information that could reveal dedup hits;
- catalog generations, tombstones, and object references.

## 4. Core security invariants

1. **Identifier separation:** global physical IDs are never client authorization tokens.
2. **Reference authorization:** creating or resolving an `ObjectRef` requires authenticated namespace authority or proof that the principal uploaded/possesses the object under the selected protocol.
3. **Reachability authorization:** reads are allowed only through an authorized namespace/share root or a narrowly scoped repair capability.
4. **Client-side integrity:** clients verify signed namespace commits and authenticated object bytes; storage acknowledgement is not integrity proof.
5. **Rollback detection:** clients retain or gossip the last accepted namespace epoch/head and reject unjustified rollback.
6. **Deletion separation:** ordinary write devices do not hold global retention deletion or maintenance authority.
7. **Hard durability independence:** soft plaintext claims cannot reduce required autonomous shard policy.
8. **No silent privacy expansion:** moving from private to team/global dedup requires explicit policy and new object creation/migration.
9. **Canonicalization binding:** dedup keys bind representation, compression, realm, and format version.
10. **Restore independence from VOPRF:** authorized namespaces retain wrapped object keys; reads do not require the online key service.

## 5. Identity and device model

### User identity

A user identity root should be long-lived and primarily sign device and recovery-key changes. It should not sign every file update.

### Device identity

Each device has:

- a signing key for commits and claims;
- an encryption/KEM key for key envelopes;
- a device ID derived from or bound to its public keys;
- scoped authorization and expiration/revocation state;
- optional hardware-backed key protection.

Device enrollment is a high-value event and should be represented in an append-only identity log or transparency structure. Other devices should detect unexpected key additions.

### Recovery

Recovery options must be explicit:

- offline recovery key;
- threshold recovery among trusted people/devices;
- service-assisted encrypted recovery bundle;
- no recovery.

Recovery authority can decrypt or rewrap broad namespace keys and is therefore equivalent to powerful account access. It should not be hidden behind ordinary password reset semantics.

## 6. Key hierarchy

A possible hierarchy:

```text
User Identity Signing Key
  signs:
    Device Registry
    Recovery Policy
    Namespace Root Descriptor

Device keys
  - device signing key
  - HPKE/KEM key

Private Namespace Root Key
  derives/wraps:
    directory metadata keys
    random object keys
    search/index keys

Shared Folder Descriptor
  - FolderID
  - membership epoch
  - roster and roles
  - folder root key / epoch secret
  - signed commit heads

Object
  - object/data-encryption key
  - canonical representation parameters
  - ciphertext
  - one or more namespace key envelopes
```

Directory and object keys should be independent enough that sharing a subtree does not reveal siblings or ancestors. Read-only grants must not reveal signing or write-key material.

## 7. Capability model

Capabilities combine authority and, in blind mode, the keys necessary to exercise it.

| Capability | Permits | Must not permit |
|---|---|---|
| Read | resolve refs and decrypt a subtree/object | create accepted commits or delete history |
| Append | add new commits/versions | rewrite existing retained history or membership |
| Write | modify designated subtree and create commits | administer unrelated subtrees or global retention |
| Admin | change roster/roles and rotate epoch | decrypt content not included in grant unless admin also has read keys |
| Verify | enumerate expected ciphertext/shards and validate integrity | decrypt plaintext |
| Repair | reconstruct and replace ciphertext fragments | resolve arbitrary user namespace refs or decrypt |
| Maintenance | compact indexes, run GC under policy | bypass retention or content authorization |
| Retention delete | authorize expiry/destruction after policy | ordinary content writes |

Capabilities should include scope, epoch, issuer, audience/device, expiry, and policy version. The server may revoke tokens immediately; cryptographic epoch rotation limits future access without trusting token revocation alone.

## 8. Encrypted namespace design

### Directory object

A directory object contains encrypted entries such as:

```text
entry random ID
padded/encrypted name
entry type
child ObjectRef or folder capability
wrapped child key
metadata (padded/encrypted)
version/tombstone state
```

Directory ciphertext should avoid deterministic names or labels. Entry order may be randomized or canonicalized inside encryption depending on merge requirements.

### Namespace head

The current state is a signed commit DAG, not a mutable unsigned pointer.

```text
NamespaceCommit {
  namespace_or_folder_id
  membership_epoch
  parent_commit_ids[]
  encrypted_root_ref
  author_device_id
  logical time / author sequence
  policy and format version
  signature
}
```

The coordinator stores a set of accepted heads. Compare-and-swap may optimize the no-conflict case but must not discard a valid concurrent head.

### Rollback and fork detection

A malicious coordinator can serve different histories to different devices unless clients retain evidence.

Minimum measures:

- each device stores last accepted epoch/head;
- commits link to parents and membership epoch;
- device synchronization compares known heads;
- unexpected non-descendant state is surfaced as a fork;
- optional witness/gossip service or transparency log records identity, roster, and head checkpoints.

A full global transparency service may be deferred, but the format must support signed checkpoints and witness gossip.

## 9. Sharing and membership

### Small groups

For each new folder epoch, HPKE-wrap the epoch secret to each authorized member/device. Complexity is linear in recipient count but simple to implement and audit.

### Large or dynamic groups

MLS can establish asynchronous group epochs with forward secrecy and post-compromise security. It should be used as a key-distribution component, not as the filesystem authorization or storage format.

Research questions include:

- whether membership is per user or per device;
- how offline devices receive epoch transitions;
- how long old epoch secrets remain available for retained history;
- whether removing one device requires rotating every folder shared with its user;
- how coordinator-enforced ACL removal interacts with cryptographic history access.

### Read-only and write grants

A read grant includes subtree location/reference and read/decryption keys. A write grant additionally includes authority to create accepted signed commits for that subtree. Do not derive a private signing key directly from a symmetric read/write key without a reviewed construction and clear rotation model.

## 10. Revocation

Revocation has three layers:

1. **Service authorization:** stop resolving refs or accepting commits from the removed principal/device.
2. **Future cryptographic access:** rotate the folder epoch and encrypt new data/metadata under new keys.
3. **Historical cryptographic access:** re-encrypt old data or rewrap keys where the removed principal did not already retain them.

No mechanism can revoke plaintext already downloaded. Product wording should say “prevent future service access” and “exclude from future epochs,” not claim to erase knowledge.

Lazy rewrapping is usually sufficient for forward revocation. Immediate historical revocation can be extremely expensive and should be an explicit policy with measurable re-encryption work.
