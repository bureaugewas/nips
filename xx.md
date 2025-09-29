# NIP-XX: Generic Data Packages for Nostr

## NIP: XX

## Title: Generic Data Packages for Nostr

## Author: Jurjen van Genugten

## Status: Draft

## Type: Standard

## Created: 2025-09-29

---

## Abstract

Today’s data sharing, analytics, and reporting rely heavily on centralized platforms and cloud storage. Applications that need to use third-party data are forced to maintain servers, databases, and cloud infrastructure. This introduces both **technical and organizational burdens**: developers must manage complex storage systems, hire cloud specialists, and design robust security architectures. The centralized model also creates a **weakest-link problem**, where a single company’s security failure can compromise sensitive data for all participants.

This NIP proposes a decentralized alternative where **data does not need to be stored on traditional servers at all**. Instead, producers publish data directly to Nostr relays as portable “data packages,” and consumers retrieve and store the data locally (e.g., on a phone or workstation). Relays serve only as **transient or archival distribution channels**, not as custodians of sensitive data. For applications that require persistence or backups, private or paid relays can provide extended storage without undermining decentralization.

This model is particularly well-suited to domains such as **medical records**, where patient data must be exchanged securely across institutions without central data silos, and **home energy management**, where smart meters and devices can share time-series data with analytics apps without requiring each vendor to build and secure its own cloud infrastructure. By standardizing data packaging on Nostr, applications can interoperate seamlessly, avoid redundant cloud systems, and give users direct control over where and how their data is stored and shared.

This NIP defines a standardized format for **generic data packages** on Nostr. Data packages may be public or private, encrypted or unencrypted, and are designed to support **interoperable structured data exchange** across applications (e.g., medical records, IoT streams, energy usage, research data).

For now it introduces a namespace of `dp:*` tags for describing metadata (e.g., type, size, integrity, encryption) and **reuses the `p` tag** for indexing by user/owner pubkey. More tags may be reused later if possible, or may be moved to a seperate metadata note to reduce the number of tags.

---

## Motivation

* Existing Nostr event formats are optimized for text notes and simple messages, not for structured, multi-part datasets required in data-heavy applications.
* There is currently no standard way to describe metadata such as size, type, schema, compression, or encryption of shared data.
* Many real-world use cases demand privacy-preserving mechanisms: the ability to hide the true producer, obfuscate application or source identifiers, or delegate publishing through a designated key.
* A generic, application-independent container format enables reliable discovery, grouping, and reconstruction of data packages across different applications and domains.
* By standardizing this layer, Nostr can support interoperable data exchange in sensitive domains such as medical records and energy management, without requiring centralized data platforms.

---

## Specification

### Event Kind

* Proposed new kind: `5000` (“Data Package”).

### Event Content

* Raw payload:

  * **Unencrypted**: UTF-8 or base64-encoded binary.
  * **Encrypted**: ciphertext (NIP-04, NIP-44, or other).

---

### Tags

| Tag              | Value              | Parameters             | Purpose                                             |
| ---------------- | ------------------ | ---------------------- | --------------------------------------------------- |
| `p`              | pubkey (hex)       | —                      | Owner/recipient pubkey (for indexing).              |
| `dp:producer`    | pubkey (hex)       | —                      | Designated producer (may be pseudonymous).          |
| `dp:app`         | string             | version (optional)     | Application identifier (may be obfuscated).         |
| `dp:source`      | string             | —                      | Source identifier (may be obfuscated).              |
| `dp:id`          | string             | —                      | Dataset/package identifier (grouping).              |
| `dp:type`        | MIME type / format | —                      | Data type (e.g., `application/json`).               |
| `dp:ts`          | unix timestamp     | —                      | Dataset creation timestamp.                         |
| `dp:part`        | integer            | total parts (optional) | Current part index / chunking.                      |
| `dp:prev`        | event id (hex)     | —                      | Previous event id (for chaining).                   |
| `dp:size`        | integer (bytes)    | —                      | Payload size.                                       |
| `dp:checksum`    | hash string        | —                      | Integrity check (e.g., `sha256:...`).               |
| `dp:compression` | string             | —                      | Compression algorithm (`gzip`, `zstd`).             |
| `dp:schema`      | URL or schema ID   | —                      | Schema reference.                                   |
| `dp:expires`     | unix timestamp     | —                      | Expiration/retention hint.                          |
| `dp:acl`         | string             | —                      | Access control hint (`public`, `private`, `group`). |
| `dp:enc`         | algorithm          | keyref (optional)      | Encryption method & key reference.                  |
| `dp:keyreq`      | pubkey (hex)       | —                      | Pubkey to request decryption key from.              |
| `dp:version`     | string or integer  | —                      | Protocol/data version.                              |

---

### Example Event

```json
{
  "kind": 5000,
  "pubkey": "npub1producer...",
  "created_at": 1750000000,
  "tags": [
    ["p", "npub1owner..."],
    ["dp:producer", "npub1producer..."],
    ["dp:app", "app-xyz", "v1.2"],
    ["dp:source", "src-987"],
    ["dp:id", "dataset-123"],
    ["dp:type", "application/json"],
    ["dp:ts", "1749999999"],
    ["dp:part", "1", "3"],
    ["dp:prev", "prev-event-id"],
    ["dp:size", "1024"],
    ["dp:checksum", "sha256:abcdef123456..."],
    ["dp:compression", "gzip"],
    ["dp:schema", "https://schema.org/MedicalTest"],
    ["dp:expires", "1760000000"],
    ["dp:acl", "private"],
    ["dp:enc", "nip44", "keyref-xyz"],
    ["dp:keyreq", "npub1keyserver..."],
    ["dp:version", "1.0"]
  ],
  "content": "BASE64_OR_ENCRYPTED_PAYLOAD"
}
```

---

## Relay Recommendations

Relays SHOULD:

* Index all events of `kind: 5000`.
* Index on the `p` tag (owner/recipient) as the **primary search key**.
* Index on the `e` tag (event-id) for looking up missing packages in the chain.
* Index on the `dp:id` tag for grouping and retrieval of datasets.
* MAY index on `dp:app` and `dp:source` to support application-specific queries (even if obfuscated).
* SHOULD allow querying by `dp:ts` (timestamp) for chronological reconstruction.
* MAY support integrity checks by validating `dp:checksum` if provided.
* SHOULD treat `dp:expires` as a hint for pruning data after expiration.

---

## Privacy Considerations

1. **Reuse of `p` only**: ensures indexing without leaking sensitive application identifiers.
2. **Obfuscation**: `dp:app` and `dp:source` SHOULD be opaque identifiers.
3. **Designated producer**: `dp:producer` allows separating the visible producer from the actual user.
4. **Encrypted datasets**: even public datasets may be encrypted; key requests go to `dp:keyreq`.
5. **Private delivery**: The same tag structure can be used inside direct encrypted messages (NIP-04).

---

## Rationale

* Uses `dp:*` namespace to avoid collisions, while leveraging `p` for compatibility with existing indexing.
* Supports both **streaming** (multi-part, `dp:part`, `dp:prev`) and **static datasets**.
* Enables **interoperability**, **privacy-preserving metadata**, and **integrity checks**.

---

## Backward Compatibility

Legacy clients will ignore unknown tags and events of `kind: 5000`. Relays only need to index `p` and `dp:id` for basic functionality.

---

## References

* [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
* [NIP-04](https://github.com/nostr-protocol/nips/blob/master/04.md)
* [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md)
* [NIP-90](https://github.com/nostr-protocol/nips/blob/master/90.md)

