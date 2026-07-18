# 0x1

## Concept

BondChain is a multi-platform core for modeling social interactions as **Bond + Bond Chain**.
- **Bond** — an actor (person, business, place, digital product).
- **Bond Chain** — a link between Bonds that evolves through interactions.

Inspired by **Pokémon Go**, **Bump**, **Birds**, and other gamified social apps.
The key difference: **it is the links that evolve, not the characters**.

---

## Protocol Invariants

1. **Identity = keypair.** A Bond address is derived deterministically from the Ed25519 master pubkey. The master key exists only as a BIP39 mnemonic (cold); private keys never leave the device. Authorization is challenge-response.
2. **No record exists without two signatures.** Every Chain record carries signatures from both participants — the cryptographic embodiment of mutual consent.
3. **Chain = a strictly private, bilateral, append-only hash-DAG.** Each record references its parent records (`parents[]`); canonical order is topological sort with hash tie-break. Real-world chronology (timestamps) is **not stored** in the Chain.
4. **Append-only envelopes, erasable payloads.** A record = a permanent minimal envelope (structure, signatures, commitment) + a payload encrypted under a per-record key. Deleting a step = bilateral redaction: both parties countersign a redaction record and destroy the payload key (crypto-shredding).
5. **Strangers are never linked automatically.** All pre-genesis communication is deniable (no signatures, no permanent artifacts). A Chain is born only through explicit mutual consent (the handshake), which is the genesis.
6. **Multi-device.** Each device holds its own non-extractable keys (Ed25519 sign + X25519 encrypt), bound to the master via a cert chain. Device-set rotation uses monotonic epochs; revocation requires a mnemonic ceremony.
7. **Zero server-side state.** State lives on the participants' devices (each side holds a full copy of the shared Chain) and inside the signed payloads of handshake links/QR codes themselves.

---

## Core Principles

- **Bond = node** in the graph.
- **Bond Chain = edge/chain** between nodes.
- **Evolution** happens in the links (transactions, meetings, messages).
- A **business Bond** can act as a hub node holding many chains at once.
- The **network** forms as a living graph: Bonds stay static, Chains change.

---

## Architecture

- **Core**: a Rust crate (chain engine, COSE/CBOR envelope, HPKE) — a single crypto scheme across all platforms.
- **FFI targets**: **WASM (Telegram Mini App — current MVP)**, Swift (iOS), Kotlin (Android), Node.js.
- **MVP topology**: a static TMA bundle with no backend; Telegram is an untrusted transport (deep links, QR, chat blobs); the bot is an empty namespace for `t.me/bot/app`.
- **Persistence**: fully on-device (IndexedDB + DeviceStorage + SecureStorage). The `.bond` format = the canonical CBOR/COSE serialization of a record; also the export format. SQLite is an option for native clients in later phases.
- **Protocol specification**: [`docs/protocol-mvp.md`](docs/protocol-mvp.md).

---

## Chain Evolution Algorithm

- **Transaction** → +1 bond strength.
- **Communication** → +0.5.
- **Meeting** → +2.
- **Joint business visit** → +3.
- **Business event** → +5.

`score_delta` is a payload field signed by both parties; bond strength is a pure deterministic function over the local copy of the DAG (the sum is commutative, so concurrent branches do not affect the result).

---

## Vision

BondChain is not just a social network.
It is a **gamified social graph** where every Bond is an actor and every Chain is a living thread of interactions that evolves.
Goal: a **new architecture of relationships** that runs on any device, independent of platform.

---
