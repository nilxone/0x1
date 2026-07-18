# Protocol MVP — Telegram

Protocol specification for the Telegram MVP. Locks in decisions before the first line of code.
Status: draft. Source precedence: this document > historical discussions.

---

## 1. Key Roles

The word "key" covers three independent roles; they are never mixed within a single key instance:

| Role | Primitive | Purpose |
|---|---|---|
| Identity | Ed25519 master (mnemonic) | Bond address = f(master pubkey); root of the cert chain |
| Consent (signing) | Ed25519 device key | Signs Chain records; non-extractable, born and dies on its device |
| Data access | X25519 device key | Unwraps per-record keys `k_r`; cross-signed by the signing key |

## 2. Crypto Stack (uniform across all platforms)

- **Signatures**: Ed25519 (WebCrypto; fallback to `@noble/curves` on older WebViews — same client-side code path).
- **Key exchange/encryption**: X25519 + HPKE (RFC 9180).
- **Payload AEAD**: XChaCha20-Poly1305 with a random per-record key `k_r`.
- **Envelopes**: COSE_Sign (RFC 9052) over deterministic CBOR — native multi-signer support; canonical CBOR yields stable hashes.
- **Mnemonic**: BIP39, 12/24 words; shown exactly once, rendered only by the TMA DOM; opt-in seed in SecureStorage.
- Keys are generated with `extractable: false`; CryptoKey handles live in IndexedDB.

## 3. Record Format

A record = **permanent envelope + erasable payload** ("append-only envelopes, erasable payloads").

```
// ENVELOPE (permanent, ~250 bytes; record_id = H(envelope))
{
  v: 1,
  parents: [hash, ...],        // DAG head(s); [] = genesis; 2+ = merge
  commitment: H(ciphertext),   // hash of the ciphertext, NOT the plaintext (dictionary-attack resistance)
  authors: { A: addr#devref, B: addr#devref },
  epoch: { A: n, B: m },
  sig_A, sig_B                 // both signatures over the full envelope
}

// PAYLOAD (AEAD(k_r, ...); k_r wrapped to the X25519 keys of both parties' live devices)
{
  type,                        // meeting | transaction | message | redact | rotation | ...
  score_delta,                 // co-signed; evolution scale from the README
  idempotency: H(intent),      // dedup of concurrent proposals from different devices
  details: { ... }             // no trusted timestamps; dates are view-layer data only
}
```

- **Ordering**: topological sort of the DAG + tie-break by record hash. Real chronology is not stored in the Chain; provable time (if ever needed) is an external anchor over the envelope.
- **Merge**: an ordinary record with 2+ parents — no special machinery.
- **Bond strength**: a pure function over the DAG; the `score_delta` sum is commutative.

## 4. Handshake (genesis)

Three legs; the record exists only after the full cycle:

1. **A → invite**: `{edA, xA, nonce, ttl, sigA}` → CBOR → base64url → `t.me/bot/app?startapp=...` (≤512 chars) or QR (`showScanQrPopup`).
2. **B → countersign**: explicit consent in the UI → `{H(invite), edB, xB, sigB}` via the same channel.
3. **A → final signature** over the complete genesis (folded into the first working record — no UX cost).

Idempotency: nonce+TTL plus genesis dedup by address pair — a re-opened link is harmless.
Channels: remote — deep link through chat; co-present — QR (zero servers, co-presence = consent).

## 5. Action Taxonomy (policy layer)

One cryptography, one write path; only the human friction before signing differs:

| Action | Signatures | Friction |
|---|---|---|
| Map, reading own copy, drafts | — | — |
| Pre-genesis messaging | none (deniable MAC, ephemeral X25519) | — |
| Meetings, background events | both | zero/silent: the act itself (QR/tap) or a pre-set policy |
| Transactions, enrollment, rotation, redaction | both | explicit confirm + biometrics |

Rule: **no Chain record exists without two signatures; the key as consent is required for every record, the key as ceremony only for high-stakes actions.**

## 6. Multi-Device

- Each device: its own pair (Ed25519 + X25519), non-extractable, never migrates.
- **Enrollment**: from any live device, biometrics → endorsed cert `{pub_new, endorsed_by, epoch}`. No mnemonic required.
- **History access for a new device**: re-wrap of `k_r` (lazy on sync) — only wrapped copies travel, never private keys.
- **Epoch**: a monotonic device-set generation counter. A verifier that has seen epoch N permanently rejects epoch < N (ratchet). Never wall-clock.
- A **rotation record** is anchored into every Chain with the counterparty's countersign.
- **Revocation ceremony** (mnemonic only): epoch+1, the master re-signs certs of the *surviving* devices directly (flattening the endorsement tree — this also kills devices enrolled by an attacker); everything not re-signed is dead.
- Duplicate proposals from different devices: the `idempotency` field in the payload.
- MVP fleet limit: N ≤ 3–5.

## 7. Redaction (deleting a step)

1. Ordinary DAG flow: propose `{type: redact, target: record_id, score_delta}` → countersign.
2. Both parties destroy `k_r` + ciphertext → crypto-shredding: any surviving ciphertext copy is permanently noise.
3. The envelope and commitment remain — the DAG stays intact, no holes.
4. Levels: in-flight proposal — withdraw, traceless; committed — bilateral only; unilateral — local-forget + suppress flag against re-sync.

The redaction's `score_delta` is chosen by the parties (0 = "erase content, keep points"). The product default is TBD (see §11).

## 8. Storage (fully on-device)

| Store | Contents |
|---|---|
| IndexedDB (+`storage.persist()`) | CryptoKey handles, full Chain |
| DeviceStorage | mirror of head+chunks (against WebView cache eviction) |
| SecureStorage (Keychain/Keystore) | seed, opt-in; survives reinstall on iOS |
| CloudStorage | **nothing** |

In-flight state = the signed payloads of links/QR codes themselves. Server-side state does not exist.

## 9. Sync and Recovery

- Device handoff/enrollment transfers only an identity package (certs + manifest: counterparties, heads, pending proposals). History is pulled from counterparties: hash-DAG verification plus both signatures removes any need to trust them.
- Device lost, mnemonic alive → key recovery → challenge → the counterparty returns the history. **Mnemonic = identity, counterparty = data**; full recovery requires both.
- Own devices co-present: QR/WebRTC (phase 2); remote — via counterparty copies.

## 10. MVP Topology

1. **Telegram** — untrusted transport + runtime (links, QR, chat blobs, Device/SecureStorage APIs). Trusted for delivery, not for content.
2. **TMA static bundle** (Pages) — the entire product: WASM core, transport codecs, UI. No webhook, no initData validation (Bond ≠ Telegram identity).
3. **Bot @handle** — an empty namespace. Day-to-day UX may live in the bot chat; the TMA acts as a signing enclave (hardware-wallet pattern: open → sign → close).

**Business Bond**: a business's "device" is its own backend; a key on its own server does not violate the invariant. Custody (a key held by a third-party operator) is allowed only with a visible `custodial: true` flag in the cert; silent custody is forbidden.

## 11. Honest Limits and Open Questions

Limits (accepted deliberately):
- **Revocation lag**: a counterparty learns about a new epoch on first contact with any live device; until then a stolen device remains valid to them. The price of serverless.
- **Remote-handshake metadata** is visible to Telegram (who links with whom; content is not). The QR flow is free of this.
- **Trust root** = the static-site deployment: a malicious build cannot extract non-extractable keys but can sign with them.
- Wipe on redaction/revocation is cooperative; a malicious archivist who saved `k_r` in advance is outside the threat model.
- Signature non-repudiation also works outward (a defector can prove record contents to third parties); mitigated by payload encryption + redaction.
- N devices = N plaintext copies of history → at-rest encryption is mandatory.

Open (blocking their respective parts, not the MVP as a whole):
1. Default `score_delta` on redaction (keep points or remove) — determines whether one can "farm interactions and scrub the traces."
2. Remote device handoff (new device with you, old one far away) — needed, or co-present only?
3. Map discovery: in the MVP or phase 2? If MVP — it pulls in the deniable layer (X3DH pattern), ephemeral discovery addresses, and hashcash stamps against spam.
4. Business-device API (a branch of the cert format).