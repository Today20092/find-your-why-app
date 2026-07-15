# Pear backend research for a trust-ranked text social app

Date: 2026-07-14

## Executive conclusion

[Pear](https://docs.pears.com/) is a credible foundation for a native, local-first, peer-to-peer social app, but it is not a hosted backend-as-a-service. It supplies a cross-platform runtime, encrypted peer discovery and transport, replicated append-only storage, multiwriter state, and peer-distributed application updates. The application must still define its identity recovery, moderation, data-availability, discovery, indexing, and ranking protocols.

The smallest credible MVP is desktop-first (Electron), text-only, and local-first. Each user owns a signed append-only event log for posts and trust decisions; peers replicate logs they explicitly follow or discover; each client builds its own local indexes and computes a bounded personalized trust score. Use `Corestore` + `Autobase` with a `Hyperbee` view, one `Hyperswarm` replication topic per social space, and Keet Identity proofs for authorship. Keep ranking client-side and deterministic. Add mobile shells only after the protocol stabilizes.

Pear calls this “zero-infrastructure,” but production still needs always-online seeders or blind peers. A new peer cannot fetch an app or dataset unless some online peer has the bytes ([Pear overview](https://docs.pears.com/), [availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering)).

## What Pear provides

### Runtime and cross-platform shape

Pear is an installable P2P runtime, development, and deployment platform for desktop, mobile, and terminal applications ([Pear documentation](https://docs.pears.com/)). Its recommended architecture separates the UI host from a “Pear-end” Bare worker that owns identity, storage, and peer protocols. The worker can be reused across Electron desktop, terminal, and mobile shells; the UI is platform-specific. Desktop embeds `pear-runtime`; mobile currently embeds Bare through React Native/Bare Kit and must manage worklet lifecycle itself because `pear-runtime` is not supported there yet ([runtime and languages](https://docs.pears.com/explanation/runtime-and-languages), [one core, many platforms](https://docs.pears.com/explanation/bare-on-native), [embed Bare in React Native](https://docs.pears.com/how-to/run-on-native/embed-bare-in-react-native/), [Pear desktop architecture](https://docs.pears.com/explanation/pear-desktop-architecture)). Mobile worklets must stop active I/O when backgrounded or the operating system may terminate them.

Implication: “cross-platform” does not mean one browser UI deployed everywhere. Start with the official [`hello-pear-electron`](https://github.com/holepunchto/hello-pear-electron) structure, isolate all protocol/data/ranking logic in the Bare worker, and expect separate React Native mobile UI and update-lifecycle work later. The official production chat scaffold already includes the useful shape—worker, Corestore persistence, Hyperswarm, encrypted Autobase multiwriter state, a HyperDB view, blind-pairing invites, IPC, and desktop OTA—so reuse it rather than assembling those pieces independently ([reshape chat into a production app](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/reshape-into-a-production-app/)).

### Networking and synchronization

`Hyperswarm` discovers peers by 32-byte topics and opens encrypted connections. Underneath, `HyperDHT` identifies peers by public key, performs NAT traversal, and uses Noise/Secretstream encryption ([Hyperswarm](https://docs.pears.com/reference/building-blocks/hyperswarm), [HyperDHT](https://docs.pears.com/reference/building-blocks/hyperdht), [Secretstream](https://docs.pears.com/reference/helpers/secretstream)). Direct connectivity can fail when both sides use randomizing NAT; HyperDHT does not relay by default, so the application needs relay/blind-peer capacity for reliable consumer connectivity ([connect peers by key](https://docs.pears.com/how-to/connect-to-peers/connect-two-peers-by-key-with-hyperdht)).

Pear replication is peer-to-peer and incremental. Offline writes can be stored locally and synchronized when peers reconnect, but no peer can retrieve unseen data while every holder is offline. This is a local-first availability model, not cloud durability ([storage and distribution](https://docs.pears.com/explanation/storage-and-distribution), [availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering)).

### Data primitives

- `Hypercore` is an authenticated append-only log and the replication primitive ([Hypercore](https://docs.pears.com/reference/building-blocks/hypercore)).
- `Corestore` manages many related Hypercores under one storage root and replication stream. Pear recommends one Corestore per application, with namespaces to avoid name collisions ([Corestore](https://docs.pears.com/reference/helpers/corestore)).
- `Hyperbee` is a sorted key/value B-tree over one Hypercore, supporting point lookups, ordered/range iteration, atomic batches, namespaces, and snapshots ([Hyperbee](https://docs.pears.com/reference/building-blocks/hyperbee)).
- `Autobase` combines multiple writer logs into a causally ordered, deterministic view. Its `apply` handler must behave like a pure reducer because previously observed operations can be reordered as causal information arrives ([Autobase](https://docs.pears.com/reference/building-blocks/autobase)).

These are lower-level building blocks, not SQL or a managed query API. Queries must be designed as materialized keys/ranges in a Hyperbee view. Schema migrations, authorization rules, deletion semantics, spam resistance, and moderation are application protocol concerns.

### Identity and authentication

Pear connections authenticate peer keys, and Keet Identity can create a portable identity with per-device keys. An author attaches an identity proof to a payload; any peer with the identity public key can verify authorship without a shared secret ([portable identity](https://docs.pears.com/how-to/manage-identity/create-a-portable-identity-with-keet-identity-key), [add identity to chat](https://docs.pears.com/how-to/manage-identity/add-keet-identity-to-a-chat-app)).

This supports cryptographic authorship, not a turnkey account service. The app still needs onboarding, display-name rules, key backup/recovery, device revocation, and protection of secret keys. Identity and P2P state belong in the Bare worker, not the renderer. Electron should keep `contextIsolation: true` and expose a narrow validated preload API ([workers](https://docs.pears.com/explanation/workers), [build a P2P chat](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/build-a-peer-to-peer-chat)).

### Deployment and operations

Pear distributes applications through stable `pear://` links and supports P2P over-the-air updates. The documented production flow progresses through stage, provision, and multisignature release gates; `pear seed pear://<app>` keeps application bytes online ([deploy an application](https://docs.pears.com/how-to/operate-an-app/manual-deployment/deployment), [release pipeline](https://docs.pears.com/explanation/release-pipeline), [availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering)).

Build on `pear-runtime`, not the deprecated global `pear run` API ([migration guide](https://docs.pears.com/how-to/operate-an-app/migration/)).

Application availability and user-data availability are separate. Seeders can keep the application installable; data seeders or blind peers must also retain and serve social logs. Plan for at least two independent availability peers before calling the service durable. Pear's public documentation reviewed here does not publish a hosted service price table or universal storage/bandwidth quota; self-operated peers bear those costs. Verify commercial support or managed peer pricing directly with Holepunch before budgeting.

## Concrete MVP protocol

### Replicated events

Use signed, immutable operations rather than mutable records:

```text
profile.set { displayName, bio }
post.create { postId, body, createdAt, replyTo? }
post.delete { postId }
trust.set   { subjectIdentity, value: -1 | 0 | 1, createdAt }
```

Every operation includes `authorIdentity`, `authorDevice`, a monotonic per-writer sequence, and a Keet Identity proof over the canonical encoded payload. Validate size, encoding, signature, author/writer binding, and operation shape before `Autobase.apply` mutates the view. A delete is a signed tombstone; immutable replicated bytes cannot be guaranteed to disappear from other peers.

Use one Corestore for the app, an Autobase for the replicated operation set, and Hyperbee materialized keys such as:

```text
post/<author>/<reverse-time>/<postId> -> post
post-id/<postId>                     -> post | tombstone
trust/<author>/<subject>             -> latest edge
profile/<author>                     -> latest profile
```

Do not create one global Autobase containing every user on the internet. Start with one invite-based social space/community. Autobase writer membership then provides a tractable authorization boundary, while each user's identity proof preserves authorship.

### Personalized trust ranking

For the first release, use a bounded personalized propagation pass, computed locally from the viewer's replicated trust graph:

1. Seed the viewer at score `1.0`.
2. Propagate positive trust for at most two hops with decay `0.5` per hop.
3. Treat a viewer's direct distrust as an exclusion and a strong negative score.
4. Treat propagated distrust as a bounded penalty, never as unlimited recursive exclusion.
5. Rank eligible posts by `trustScore(author) + recencyScore`; break ties by signed post ID.
6. Always provide “latest” and “directly trusted only” modes so ranking is inspectable and recoverable.

This is intentionally simpler than a global eigenvector. EigenTrust demonstrates normalized transitive reputation in P2P networks, but its global trust value and pre-trusted peers do not directly match a viewer-personalized feed ([original EigenTrust paper](https://www.ra.ethz.ch/CDStore/www2003/papers/refereed/p446/p446-kamvar/)). MeritRank is a stronger later candidate if Sybil manipulation becomes measurable because it adds transitivity/connectivity decay to bound Sybil benefit ([MeritRank paper](https://doi.org/10.1109/BRAINS55737.2022.9908685)). No trust propagation scheme creates Sybil resistance by itself; identity creation is cheap, so the product needs rate limits, invite friction, local block/mute controls, and conservative propagation.

Keep the algorithm versioned in local settings and recompute the view when it changes. Do not replicate a user's feed score as authoritative shared state: the graph subset and preferences are personal, and deterministic client-side computation is the point of this design.

### Peer discovery and availability

For an invite-only MVP, an invite carries the social-space discovery key plus the inviter's identity and writer/discovery information. Joining the space connects to its Hyperswarm topic, replicates authorized logs, and learns additional identities from verified events. Run independent blind peers/data seeders that retain the community's cores. Public global user/search discovery is deferred because Pear does not provide a global account directory or full-text search service.

## Security and privacy constraints

- P2P traffic is end-to-end encrypted, but connected peers see IP addresses and can correlate swarm presence and timing. Pear recommends VPN or Tor when IP privacy matters ([dependencies and network](https://docs.pears.com/explanation/dependencies-and-network)).
- Encryption in transit does not make public posts private. Anyone given replication capability can retain plaintext indefinitely.
- Never trust renderer input or replicated operations. Validate at the worker boundary and again in the deterministic apply function.
- Keep identity/device secrets out of Electron renderer state and logs.
- Cap post bytes, batch sizes, connections, replicated history, and ranking graph breadth to resist storage/CPU amplification.
- Pear blocks loading JavaScript over HTTP(S) by default; ship dependencies with the application rather than fetching executable code at runtime ([dependencies and network](https://docs.pears.com/explanation/dependencies-and-network)).
- Content removal in a replicated append-only system is best-effort suppression via tombstones and peer policy, not guaranteed erasure. This must be explicit in product policy.

## Recommended implementation order

1. Clone `hello-pear-electron`; keep the renderer/main/preload/worker boundary intact.
2. Implement one signed writer posting text into a persisted Corestore and render its local timeline.
3. Add Hyperswarm replication between two desktop peers.
4. Add Keet Identity proof verification and reject malformed/oversized operations.
5. Add trust edges and the bounded two-hop local ranker, with latest/direct-only fallbacks.
6. Add an invite-only multiwriter community through Autobase and a Hyperbee view.
7. Operate two seed/blind peers and test first-install, all-clients-offline, reconnect, conflict, key-loss, and NAT-failure cases.
8. Stage and seed a signed desktop build. Only then extract the stable worker protocol into mobile shells.

## Explicitly deferred

- Images/video, hashtags, full-text global search, quote posts, and notifications.
- A public global firehose or global user directory.
- Server-enforced deletion and centralized moderation guarantees.
- A sophisticated EigenTrust/MeritRank implementation before real attack data exists.
- A shared UI codebase across Electron and React Native.

These deferrals preserve the hard part worth validating: signed local-first social data, reliable replication, availability, and whether personalized trust propagation produces a useful feed.

## Primary sources consulted

- [Pear full documentation corpus](https://docs.pears.com/llms-full.txt)
- [Pear documentation](https://docs.pears.com/)
- [hello-pear-electron](https://github.com/holepunchto/hello-pear-electron)
- [Corestore source](https://github.com/holepunchto/corestore)
- [Autobase source](https://github.com/holepunchto/autobase)
- [Hyperbee source](https://github.com/holepunchto/hyperbee)
- [Hyperswarm source](https://github.com/holepunchto/hyperswarm)
- [EigenTrust original paper](https://www.ra.ethz.ch/CDStore/www2003/papers/refereed/p446/p446-kamvar/)
- [MeritRank final paper](https://doi.org/10.1109/BRAINS55737.2022.9908685)
