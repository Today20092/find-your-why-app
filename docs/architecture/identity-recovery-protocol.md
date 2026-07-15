# Identity recovery protocol decision

## Status

**Decision:** keep production recovery out of the Windows MVP until the protocol below has an interoperable implementation, adversarial tests, and an independent security review. The disposable prototype may create local identities, but it must not promise that a recovered identity will converge safely for followers.

Pear provides the required pieces, but not this complete policy. `keet-identity-key` derives a stable identity from a mnemonic and lets that identity attest fresh device keys; its official guide does not define device revocation, delegated pairing, recovery epochs, or concurrent-restore reconciliation ([Pear portable identity guide](https://docs.pears.com/how-to/manage-identity/create-a-portable-identity-with-keet-identity-key/)). Hypercore authenticates append-only log states with signed Merkle trees and exposes exact `fork`, `length`, and `treeHash` checkpoints, but truncation intentionally creates a new monotonically increasing fork; it does not decide which of two identity recovery branches represents the person ([Hypercore source and API](https://github.com/holepunchto/hypercore)).

Do not paper over that gap with a shared recovery secret writing one “stable manifest Hypercore.” Restoring the same secret on two disconnected devices can produce two valid histories under the same key. Authentication proves who could sign each history, not which history every reader should select.

## Authority model

Use three distinct keys:

1. **Recovery authority:** the long-term root secret in the encrypted recovery file. Its public key is the stable `identityPublicKey` and therefore the social identity. It signs only initial and recovery certificates. It is not an everyday Hypercore writer and must not remain plaintext in the normal device store.
2. **Operational device key:** a fresh Ed25519 key pair held by the currently authorized writing device. It signs application events and a normal handoff to the next device. The root secret never needs to sign posts.
3. **Writer core key:** the Hypercore key for that operational generation. It identifies an authenticated log, not the person. The authorization chain binds it and the operational device key to `identityPublicKey`.

This retains Pear's useful distinction between a portable identity, per-device key, device proof, and data proof, while adding application-level authorization state that the documented library does not currently provide. Ed25519 and verification behavior must come from a maintained library with the [RFC 8032](https://www.rfc-editor.org/rfc/rfc8032.html) test vectors, never handwritten cryptography.

All signed objects use one versioned, domain-separated, deterministic binary encoding. Deterministic CBOR is a suitable specification target, subject to implementation-library review; RFC 8949 defines deterministic encoding requirements needed to make signatures and certificate digests portable ([RFC 8949, section 4.2](https://www.rfc-editor.org/rfc/rfc8949.html#section-4.2)). Unknown versions or non-canonical encodings are rejected before signature verification.

## Stable rendezvous, not a global registry

Derive a 32-byte topic locally:

```text
rendezvousTopic = BLAKE2b-256(
  "find-your-why/identity-rendezvous/v1" || identityPublicKey
)
```

Every follower can derive the same topic from an invite's full `identityPublicKey`. Active devices, followers that cache authorization chains, and later opt-in host peers may announce on it. On a Hyperswarm connection, a small versioned Protomux protocol exchanges the best known authorization chain and the current writer core key. Peers then replicate the selected writer core through the existing Corestore connection.

This topic is a rendezvous mechanism, not storage or consensus. Pear documents Hyperswarm topics as arbitrary 32-byte identifiers and says the DHT primarily finds and connects peers rather than providing long-term application storage ([Pear peer-to-peer model](https://docs.pears.com/explanation/peer-to-peer-demystified/)). Therefore a replacement remains undiscoverable while nobody holding its certificate is online. Seeding or blind peering can improve availability but cannot choose the legitimate recovery branch; Pear explicitly describes blind peering as an availability cache, not a replacement for capability checks ([Pear availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering/)).

Invites carry `protocolVersion`, `identityPublicKey`, and the best known authorization chain. The display address remains derived from `identityPublicKey`, so recovery never changes the person's address.

## Certificate chain

The exact compact-encoding schema is an implementation ticket. The semantic records are fixed here.

### Initial authorization

A root-signed `identity-genesis` certificate contains:

- protocol and certificate versions;
- `identityPublicKey`;
- recovery epoch `0` and no parent recovery digest;
- operational generation `0`;
- initial device public key and writer core key;
- a random 128-bit branch nonce;
- a signature over every preceding field.

The writer core's first block contains this certificate and binds the core to it. A post is valid only when its device signature verifies and its writer core is reachable through the selected authorization chain.

### Normal device pairing

The MVP permits **one active writer**, not concurrent multi-device writing. “Add another device” therefore transfers writing authority:

1. The old writer appends a `writer-transfer` event authorizing the new device public key and new writer core key at operational generation `g + 1`.
2. The accepted cutoff for the old core is exactly the prefix ending with that transfer block: `(coreKey, fork, length, treeHash(length))`.
3. The new core's genesis block references that exact checkpoint and the transfer event.
4. Events from the old core after the cutoff are valid Hypercore bytes but invalid identity events. The old device becomes read-only.

This is pairing, not recovery: it requires the active writer, does not use or transmit the root recovery secret, and cannot help when every active writer is lost. Pear's production chat uses Blind Pairing to admit a new Autobase writer, but that is room membership rather than a portable identity recovery rule ([Pear production pairing tutorial](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/reshape-into-a-production-app/)). Concurrent writers are deferred; if later required, Autobase is the relevant primitive because it linearizes causal writer DAGs and builds deterministic views, but its ordering can change until quorum-signed checkpoints and it would add a membership/indexer policy that this single-writer identity does not need ([Autobase source and ordering](https://github.com/holepunchto/autobase)).

### Recovery

Importing the encrypted recovery file creates a fresh device key and writer core, then emits a root-signed `identity-recovery` certificate containing:

- `identityPublicKey` and protocol version;
- `recoveryEpoch = parent.recoveryEpoch + 1` as an unsigned 64-bit integer;
- the digest of the parent recovery certificate;
- a fresh 128-bit branch nonce;
- the replacement device public key and writer core key;
- the best verified cutoff of the previously selected active writer as `(coreKey, fork, length, treeHash)`; zero length is allowed when no authenticated prefix is available;
- the root signature over all fields.

The replacement core's genesis block contains the full self-contained certificate chain. The client must persist the new certificate and writer genesis durably before enabling Publish.

Recovery does not erase history. Events at or below the declared authenticated cutoff remain part of the selected identity history. Events from the replaced writer beyond that cutoff, and events on losing recovery branches, remain retrievable signed data but are excluded from the current profile and feeds.

## Deterministic split-brain rule

Every client validates all complete candidate chains it receives, then selects one without timestamps, arrival order, popularity, or a central service:

1. Reject a chain with a bad root/device signature, wrong identity, unknown version, epoch gap or overflow, missing parent digest, repeated certificate, invalid writer handoff, or checkpoint whose `(fork, length, treeHash)` cannot be proved.
2. Prefer the valid chain with the greatest recovery epoch.
3. If valid chains end at the same recovery epoch, prefer the lexicographically smaller BLAKE2b-256 digest of the canonical final recovery certificate.
4. Within the selected recovery epoch, follow only the single valid operational-transfer chain with consecutive generations. A conflicting transfer from one old writer is resolved by the same lexicographically smaller canonical transfer-event digest, and the loser is excluded.

This makes two concurrent restores converge once clients observe the same candidate set. A client may temporarily select a different branch while partitioned and must rebuild its derived profile/feed view if a preferred branch later arrives. Hyperbee or any other local index is only a materialized view; the signed certificates, exact Hypercore checkpoints, and writer events are the authority.

The root holder can deliberately resolve a same-epoch tie by issuing the next recovery epoch referencing the desired parent. No branch is physically deleted.

## Replay and rollback rules

- Persist, atomically, the greatest selected recovery epoch, its final certificate digest, and all accepted writer cutoffs for every known identity.
- Never automatically replace that state with a lower epoch or a losing same-epoch digest, including after restart or when only an older peer is reachable.
- Verify an old core prefix using the exact key, fork, length, and tree hash. A matching key alone is insufficient because Hypercore supports signed truncation and fork changes ([Hypercore truncation and `signable`](https://github.com/holepunchto/hypercore#await-coretruncatenewlength-options)).
- Replaying an already indexed `<coreKey, fork, blockIndex>` is idempotent. The same coordinate with different content is a conflict and must not be indexed.
- On branch selection change, rebuild derived state deterministically from the selected chain; do not patch the old Hyperbee view in place with unaudited side effects. Autobase applies the same general rule to reorderable inputs: derived views must be rebuilt deterministically from their store ([Autobase views](https://github.com/holepunchto/autobase#views)).
- A fresh install with no cached checkpoint can accept an older valid branch until it encounters a newer one. Without an available peer, transparency log, or consensus service, this rollback window is irreducible and must be disclosed rather than hidden.

## Root compromise

Compromise of the recovery authority is catastrophic for that identity. An attacker can authorize devices, create higher recovery epochs, select cutoffs that hide later posts, and impersonate the person. No local tie-break can distinguish the attacker because both possess the same root authority, and this project intentionally has no global registrar that can revoke it.

The only safe outcome is to create a new identity and ask contacts to verify and follow it. A still-controlled old device may publish a signed warning, but that warning is evidence, not a cryptographic revocation of the compromised root. The product must say “Anyone with your recovery file and passphrase can act as you,” not promise account recovery after root compromise.

## Required executable invariants before production recovery

1. **Encoding vectors:** Windows/Bare and every future client produce identical bytes and digests for each certificate; non-canonical variants are rejected.
2. **Signature boundary:** mutation of every field, domain tag, identity key, or signature fails validation; RFC 8032 positive and negative vectors pass.
3. **Rendezvous determinism:** the same identity key always derives the same 32-byte topic and different identity keys do not share test vectors.
4. **Initial authorization:** posts from an unlisted device/core never enter indexes even when their Hypercore proofs are valid.
5. **Pairing cutoff:** the transfer block is accepted, every prior event remains, and every old-writer event after it is excluded.
6. **Concurrent restore convergence:** all delivery permutations of two same-epoch restores select the same certificate; a valid next epoch supersedes it.
7. **Rollback persistence:** restart, peer disappearance, replay, and delayed older chains never lower a stored checkpoint.
8. **Fork binding:** wrong `fork`, `length`, or `treeHash` fails even under the correct core key.
9. **Crash atomicity:** a crash before durable certificate/genesis storage cannot enable publishing; a crash during branch-view rebuild resumes without mixed-branch state.
10. **Parser/resource abuse:** fuzzed chains, cycles, epoch overflow, excessive chain depth, oversized certificates, and many rendezvous candidates remain bounded and cannot crash the worker.
11. **Availability loss:** with the recovered device and all certificate-seeding peers offline, followers report “recovery update unavailable” rather than silently claiming the old writer is current.

## Newly surfaced fog

- Choose and audit the exact compact-encoding schema, digest/signature functions, maximum chain size, and maintained Bare-compatible libraries.
- Decide whether application-level delegated device proofs should extend `keet-identity-key` or replace the existing assumption that every event can be verified by that library alone.
- Threat-model rendezvous-topic metadata leakage, candidate spam, connection limits, and host-peer retention before implementing discovery.
- Specify UI for a temporarily selected branch, a later branch switch, data excluded by a recovery cutoff, and moving writing authority between devices.
- Define an identity-migration and contact-verification ceremony for root compromise; the old identity cannot repair itself cryptographically.
- Revisit Autobase only if simultaneous active devices become a real requirement; do not add it for one-writer handoff.

