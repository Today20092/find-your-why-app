# Signed events, replication, and availability decision

## Decision

The Windows MVP uses **one public, append-only Hypercore per identity**. An identity owns its log; following another person means opening that log by key, joining its discovery topic, and replicating it through the app's shared Corestore.

Do not put the public social network in one Autobase. Autobase is a multiwriter system whose membership, indexers, causal linearization, and deterministic derived view fit a shared room, but they would introduce a global membership authority and an unnecessary consensus domain here. Its ordering can also change until a signed checkpoint, so it is not needed for independent publisher histories ([Autobase API and ordering](https://github.com/holepunchto/autobase#ordering)). Reconsider Autobase only when one identity must coordinate concurrent writes from multiple devices.

Hypercore supplies authenticated log integrity: blocks belong to an append-only log backed by signed Merkle trees and can be replicated sparsely ([Hypercore source/API](https://github.com/holepunchto/hypercore)). That guarantee proves that a block belongs to a particular Hypercore; it does **not** prove that the log represents a portable person. Pear explicitly distinguishes a Hypercore key from a person. Therefore every application event also carries a `keet-identity-key` data proof bound to the stable identity public key ([portable identity model](https://docs.pears.com/how-to/manage-identity/create-a-portable-identity-with-keet-identity-key/), [identity chat example](https://docs.pears.com/how-to/manage-identity/add-keet-identity-to-a-chat-app/)).

## Minimum event envelope

Encode and sign a deterministic binary payload before appending it. The proof is outside the signed payload:

```text
event = {
  v: 1,
  type,
  identityKey,       // stable Keet identity public key
  deviceKey,
  createdAt,         // publisher-claimed Unix milliseconds
  body,
  proof              // device proof + signature over every preceding field
}
```

The Hypercore key and block index are transport coordinates, not duplicated inside the signed payload. A post ID is `<identityKey>:<coreKey>:<blockIndex>`; this is deterministic and requires no random identifier. The reader accepts an event only when its application proof verifies against `identityKey`, its device attestation is valid, and its schema and limits pass. This explicit application proof is required even though Hypercore separately verifies the log.

## Event types

```text
profile.set  { displayName, addressName, bio }
post.create  { text }
post.edit    { targetPostId, text }
post.delete  { targetPostId }
follow.set   { subjectIdentityKey, value: true | false }
trust.set    { subjectIdentityKey, value: true | false }
```

- The latest valid `profile.set` in an identity's log is its current profile. Display name is 1–50 Unicode characters. Address name is 3–24 ASCII lowercase letters, digits, or underscores. Bio is at most 160 Unicode characters.
- A text post is at most 10,000 UTF-8 bytes after normalization. Empty/whitespace-only posts are invalid.
- An edit or deletion is valid only when its author identity equals the target post's author. The original signed event remains replicated. An edit is a replacement linked to the original; a deletion is a signed notice that compliant clients hide by default, never a promise that bytes disappeared.
- The latest valid `follow.set` and `trust.set` by log order wins for that subject. Trust is positive-only and affects Discover; following controls the separate Following feed.
- Mute and block are **not events**. They are private device-local records, are never announced or replicated, and override local display.

Replies, likes, reposts, media, propagated distrust, shared blocklists, and global reports add no value to the agreed MVP journey and have no schema yet.

## Ordering and conflicts

Within one publisher, Hypercore block index is authoritative. Later valid events supersede earlier events where the schema says “latest.” A publisher clock is untrusted: `createdAt` is displayed as a claim and used for approximate cross-publisher chronology, but it never changes identity-log precedence. Following sorts available posts by `createdAt` descending, then identity key and block index for a deterministic tie-break; clients clamp obviously future timestamps to “now” for sorting so a bad clock cannot pin a post indefinitely.

There is no cross-author write conflict because an identity cannot edit another identity's state. If the same recovered identity writes through two independent cores, both logs are valid branches and the UI may show duplicates or ambiguous latest state. The MVP prevents that by supporting one active writer core per identity; safe multi-device writer discovery and convergence remain an explicit later decision.

## Replication and local indexes

The Bare worker owns one Corestore and one Hyperswarm. On each swarm connection it passes the encrypted stream to `store.replicate(connection)`, the documented Corestore/Hyperswarm pattern ([availability guide](https://docs.pears.com/explanation/availability-and-blind-peering/), [native lifecycle example](https://docs.pears.com/how-to/run-on-native/handle-app-suspension/)). For every owned or followed publisher core it joins `core.discoveryKey`; Hyperswarm topics commonly use discovery keys and provide discovery, encrypted connections, and reconnection while Hypercore performs persistence and verification ([peer-to-peer networking model](https://docs.pears.com/explanation/peer-to-peer-demystified/)).

Replication boundaries:

- Always retain and serve the owner's complete public log.
- Cache complete followed text logs for offline reading and serve them only while the app is running. This is opportunistic replication, not a durability promise.
- Discovery may fetch candidate identity metadata and bounded recent events on demand; sparse replication exists specifically to download selected blocks rather than every block ([Hypercore source/API](https://github.com/holepunchto/hypercore)). Do not automatically retain or rehost the entire discovered network.
- Keep one **local-only** materialized index for profiles, current follow/trust edges, post IDs, edits, tombstones, and feed scans. It is rebuildable from verified logs and is never treated as authoritative or replicated. Hyperbee is sufficient; HyperDB schemas and Autobase views are unnecessary for the first independent-log implementation.

The app must cap concurrent joined publisher topics and downloads; initially load direct follows plus the bounded identities needed for two-hop discovery. A Corestore can replicate many opened Hypercores over the same peer connection, so a connection does not need to be created per log ([multi-room Corestore pattern](https://docs.pears.com/how-to/connect-to-peers/host-multiple-rooms-in-one-chat-app/)).

## Invite payload

Use a versioned, URL-safe payload suitable for links, paste, and QR encoding:

```text
pear-social://identity/<base32(identityKey)>?v=1&core=<base32(coreKey)>
```

The identity key verifies application proofs; the core key opens the publisher log. The receiver derives the Hypercore discovery key locally, so it is not included. Profile/address text is not trusted invite data and is learned from the verified log. Address search remains local; the invite is the guaranteed way to find a never-seen identity.

Blind Pairing is not used for public following. The official production chat uses it to admit a candidate as an Autobase writer and disclose room/encryption keys ([production chat flow](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/reshape-into-a-production-app/)); following a public read-only log needs no admission ceremony.

## Availability states shown to people

Peer-to-peer availability is not automatic: a new reader can fetch data only while at least one online peer has it, and data may be temporarily unreachable when all holders are offline ([availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering/)). Normal screens therefore show no networking jargon. Show only actionable states:

- **Saved on this device** — required blocks are locally readable.
- **Checking for updates…** — discovery/replication is active but freshness is unknown.
- **Waiting for this person to come online** — requested data is absent locally and no serving peer is connected.
- **Available from _n_ peers** — optional detail when a live source count is known; it is not a durability guarantee.
- **Last checked _time_** — the last successful sync, not “last posted” or “account inactive.”

Keys, joined topics, local bytes, connected peers, and replication errors belong under Settings → Network & Storage.

Blind peers and user-run host peers are deferred. Official blind peers cache and serve cores without interpreting their contents, but blind peering does not replace encryption/capability checks ([blind-peering threat model](https://docs.pears.com/explanation/availability-and-blind-peering/)). This MVP's public logs are not encrypted, so a blind host would still store publicly decodable events; the UI must never imply otherwise.

## Validation boundary

The Bare worker validates every renderer command and every replicated event before indexing or displaying it:

1. supported protocol version and known event type;
2. exact required fields, canonical encoding, and total event size at most 16 KiB;
3. valid identity/device keys and application proof over the full payload;
4. valid field lengths, UTF-8, and no control characters except ordinary whitespace;
5. referenced target exists and is owned by the same identity for edits/deletes;
6. duplicate `<coreKey>:<blockIndex>` ignored idempotently.

Invalid events remain harmless replicated bytes: do not index, render, propagate trust from, or crash on them. Rate limiting is local resource protection, not identity issuance: bound open cores, simultaneous downloads, index work per update, and rendered results.

## Newly surfaced fog

1. How a recovered identity announces and reconciles a replacement writer core without a central directory.
2. Whether follows and trust edges should remain public events long-term or move to a selectively shared/private graph; discovery needs access, but public edges reveal social relationships.
3. Exact clock-skew clamping and feed freshness semantics after testing real offline/reconnection behavior.
4. Host-peer consent, retention limits, revocation, jurisdiction, and whether public-log hosting needs user-selectable peers.
5. Resource ceilings (joined topics, cached bytes, concurrent downloads) require measurements on the first Windows build rather than invented production defaults.
