# Working copies and collaboration

> Part of the [Decent Storage architecture pack](README.md). Synthesized from the two design proposals; source baseline: `architecture-roadmap.md` sections 5–6.

## 5. Working copies and uncompressed representations

## 5.1 Three integration levels

### Level 1: restore seeds

A device indexes ordinary files using compatible content-defined chunking. During restore, matching chunks are copied, reflinked, or read from local files before remote storage is consulted. This borrows the seed/index pattern from casync/desync.

Benefits:

- lower restore bandwidth;
- faster local reconstruction;
- effective reuse of prior VM images, archives, and large files;
- no repository-format change.

This level does not count toward hard durability.

### Level 2: signed availability claims

A device publishes an expiring statement:

```text
AvailabilityClaim {
  device_id
  namespace/repository
  logical_object_or_chunk_root
  chunk ranges
  source representation
  filesystem identity or version
  failure domain
  last full verification
  expiration
  signature
}
```

Claims are invalidated when a scan observes a changed file and expire if the device is offline. Random range challenges can reduce accidental or dishonest overclaiming.

Claims may:

- reduce repair urgency;
- select a cheaper repair source;
- count toward an explicitly weaker “assisted recovery” service objective.

They do not satisfy historical hard-retention policy.

### Level 3: virtual systematic shards

A future Track B format could let plaintext owners regenerate deterministic encrypted data fragments and therefore act as virtual data shards. The safe order is:

```text
plaintext
 -> canonical/versioned representation
 -> deterministic, reviewed encrypted object
 -> erasure-code data/parity fragments
```

Never compute parity directly over raw plaintext as a confidentiality mechanism; erasure coding is not secret sharing.

This level requires new format research because current Kopia pack bytes cannot generally be reproduced from one local file. It should only be pursued if simulations show substantial durability or cost gains beyond Level 2.

## 5.2 Historical-version limitation

A current file only contains chunks still present in that version. Deleted or overwritten historical chunks require autonomous storage unless another retained copy still contains them. Content-defined chunking improves reuse after small edits but does not reconstruct bytes that no longer exist.

The agent therefore creates immutable commits:

1. scan and reconcile the working tree;
2. reuse unchanged chunk references;
3. write new logical contents;
4. publish the new snapshot/commit;
5. expire claims for replaced data;
6. retain old objects until policy permits GC.

## 5.3 Archives and semantic representations

An archive byte stream and its extracted tree are not equivalent representations. Order, timestamps, permissions, compression settings, compressor versions, extra fields, and nondeterministic metadata can prevent bit-exact reconstruction.

Store three related objects where useful:

- **Raw representation:** exact original ZIP, tar, image, or container bytes.
- **Canonical logical tree:** members and metadata for semantic deduplication.
- **Representation recipe:** packaging information sufficient for reconstruction when possible.

Adapters should be sandboxed:

```text
RepresentationAdapter {
  Detect
  Enumerate
  Canonicalize
  Reconstruct
  VerifyBitExactness
}
```

ZIP, tar, OCI images, VM disk formats, and media containers need separate adapters. Protect parsers against malformed inputs, decompression bombs, path traversal, and resource exhaustion.

## 6. Mutable collaboration plane

The archive is not the live collaboration filesystem. Device agents synchronize materialized working folders and commit stable states to the immutable plane.

Borrow Mutagen's operational sequence:

```text
watch hint -> scan both sides -> compare with last agreed baseline
 -> reconcile -> stage bytes -> atomically apply -> record new baseline
```

Initial modes:

- one-way replica;
- one-way safe;
- two-way safe with explicit conflicts;
- read-only materialization;
- sparse/cache-only materialization.

The system must account for filesystem differences:

- case sensitivity and Unicode normalization;
- permissions and ownership;
- symlinks and special files;
- hard links;
- sparse files and reflinks;
- xattrs and alternate data streams;
- atomic rename and file-lock behavior;
- clocks and timestamp granularity.

Every successful reconciliation may create an immutable snapshot, but sync frequency and snapshot retention are independent policies.
