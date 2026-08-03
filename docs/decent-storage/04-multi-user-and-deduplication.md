# Multi-user namespaces and deduplication

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `architecture-roadmap.md` sections 7–8.

## 7. Multi-user architecture

## 7.1 Trusted-server path

A Trusted Server v2 is the fastest path to usable folder sharing.

The server can retain repository keys and plaintext visibility but must improve authorization:

- user and group identities;
- namespace/share manifests;
- reachability-based object authorization;
- random per-principal object references;
- server-side reference accounting;
- append-only endpoint roles;
- separate maintenance/deletion authority;
- responses that do not reveal duplicate status.

Current Kopia server logic grants content access by a broad content access level and applies finer checks to manifests. The replacement invariant is that an authenticated principal may read a content object only when the server can prove authorized reachability from a namespace or share root, or when a separately scoped repair capability permits ciphertext access.

Advantages:

- conventional cryptography and operations;
- easier migration from existing Kopia repositories;
- fastest validation of sharing, quotas, references, and GC.

Cost:

- server compromise exposes plaintext and repository keys.

## 7.2 Blind multi-principal path

In Track B, clients perform chunking, canonical compression, object encryption, metadata encryption, and commit signing before upload.

The data plane stores opaque encrypted objects. The namespace plane stores encrypted directory DAGs, key envelopes, member rosters, and signed heads.

Use two identifiers:

```text
GlobalObjectID = hash(ciphertext)     // internal to dedup/storage service
ObjectRef      = random 256-bit value // visible in an authorized namespace
```

The coordinator maps an authorized `ObjectRef` to a global object and never exposes global equality information, physical locations, owner counts, or first-upload timestamps to ordinary clients.

### Key hierarchy

```text
User identity root
  -> device signing and HPKE keys
  -> private namespace root key
  -> shared-folder epoch key
       -> directory metadata keys
       -> child-folder keys
       -> wrapped object/data keys
```

For small groups, HPKE-wrap the folder epoch key to each member/device. For large or high-churn groups, MLS may distribute epoch secrets, but MLS does not define filesystem authorization, retained-history revocation, or merge semantics.

### Capabilities

Separate:

- read;
- append/new-version;
- write/modify;
- admin/membership;
- verify/audit;
- repair/ciphertext reconstruction;
- retention/deletion/maintenance.

Endpoint devices should normally receive append or folder-scoped write authority, not global deletion or maintenance keys.

### Multi-writer commits

Represent shared-folder state as immutable signed commits:

```text
Commit {
  folder_id
  epoch
  parent_commit_ids[]
  encrypted_root
  author_device_key
  logical_clock or sequence
  policy/version
  signature
}
```

The service preserves concurrent heads. Clients find a common ancestor and perform a three-way merge. Genuine conflicts retain both versions. A general CRDT is deferred; selected structured file formats may later use CRDT adapters.

### Revocation

Revocation is forward-looking by default:

1. remove the member/device;
2. commit a new roster and folder epoch;
3. wrap the new epoch secret to remaining members;
4. use new keys for future metadata and objects;
5. lazily rewrap descendant object keys on modification.

A recipient cannot be made to forget plaintext or keys already obtained. Retroactive revocation requires re-encrypting old data or accepting server-enforced denial without cryptographic erasure of prior access.

## 8. Cross-user deduplication

## 8.1 Modes

| Mode | Scope | Encryption behavior | Main leakage |
|---|---|---|---|
| Private | user or folder | random object keys | no cross-tenant equality |
| Shared/team realm | explicit trusted group | shared realm context or key service | equality within realm; members can test candidates more easily |
| Organization realm | managed organization | organization VOPRF/POPRF context | organization-wide equality and online guessing surface |
| Global privacy mode | unrelated tenants | VOPRF-assisted message-locked encryption; full upload | coordinator sees ciphertext equality; users should not see hit status |
| Global bandwidth mode | unrelated tenants, explicit opt-in | VOPRF plus proof of ownership/upload skip | stronger existence-oracle and traffic leakage |

Private or shared-folder deduplication should be the default. Global deduplication should initially operate on whole files or large multi-megabyte segments, not tiny chunks.

## 8.2 VOPRF-assisted object derivation

For canonical representation `R`:

```text
x = Hash("decent-storage-dedup-v1" || realm || format || size_class || R)
z = VOPRF.Evaluate(key_service, x)
K_object = HKDF(z, "object-key" || context)
C = ReviewedMessageLockedEncrypt(K_object, R, associated_data)
```

The client blinds `x`; the key service does not learn the plaintext hash or PRF result. The VOPRF proof lets the client verify the configured service key. POPRF public input may bind realm and format version.

The deterministic encryption construction is a research and audit item. Do not emulate it by fixing an AES-GCM nonce.

## 8.3 Privacy-preserving upload path

Default global privacy mode:

1. client derives and encrypts the complete canonical object;
2. client uploads the complete ciphertext to a random staging identifier;
3. coordinator validates and hashes it;
4. coordinator deduplicates internally;
5. coordinator creates a random namespace `ObjectRef` and authorization record;
6. response shape, timing bucket, and logical quota behavior are the same for new and duplicate objects;
7. client stores a namespace-wrapped object key.

This preserves storage savings but intentionally gives up duplicate-upload bandwidth savings.

Do not expose:

- global ciphertext/content hashes;
- dedup hit/miss status;
- marginal physical-byte billing;
- global reference counts;
- first uploader or first-upload time;
- physical shard addresses.

## 8.4 Proof of ownership mode

Proof of ownership can permit upload skipping after a client answers unpredictable challenges over the object or a Merkle encoding. It prevents a hash-only attacker from claiming a large object.

It does not hide existence. Receiving a challenge, different timing, or a skipped upload can confirm that the candidate object already exists. Therefore it is an explicit bandwidth-optimized mode for trusted domains or large objects, not the global privacy default.

## 8.5 Split trust

Recommended role separation:

| Party | Learns |
|---|---|
| Storage nodes | encrypted fragments, sizes, access timing, limited equality within their shard view |
| Coordinator/catalog | accounts, authorization, random refs, global ciphertext equality, logical references |
| VOPRF service | authenticated request timing and blinded inputs, not plaintext hashes or stored objects |
| Client | its plaintext, keys, namespace metadata, and refs; not other owners or global counts |

A threshold VOPRF may reduce single-service trust, but it is a later protocol and operations project rather than a property supplied automatically by RFC 9497.
