# Browser and mobile integration boundaries

Status: decision for the Windows MVP specification, 2026-07-15.

## Decision

Ship Windows first with the existing Electron shell, narrow preload bridge, and Bare worker. Preserve a platform-neutral **Pear-end** behind a versioned message seam, but do not promise that the Electron renderer is reusable on mobile or that a normal browser can become a Pear peer.

The portable unit is the signed social protocol and the Bare-hosted domain/data core—not the screen implementation. Pear's documented cross-platform shape is a platform-specific shell, a typed IPC/RPC seam, and a Bare core that owns identity, storage, Hyperswarm, HyperDHT, and replication ([One core, many platforms](https://docs.pears.com/explanation/bare-on-native/), [Runtime and languages](https://docs.pears.com/explanation/runtime-and-languages/)).

On Windows, keep the official desktop boundary: a sandboxed renderer calls a narrow preload API, Electron main proxies the IPC stream, and the Bare worker owns the P2P modules ([Pear desktop architecture](https://docs.pears.com/explanation/pear-desktop-architecture/)).

This boundary must preserve the product contract on every future platform:

- **Following** shows every locally available, valid post from directly followed identities, newest first. Ranking or gateway policy may not suppress one.
- **Discover** stays separate, optional, locally calculated, newest first after bounded positive-trust eligibility, and explains the recommendation path.
- The client verifies signed events before materializing or displaying them. Transport and storage providers are availability providers, not authorities over identity or post validity.
- “Not currently available” is not “deleted.” A client cannot prove it has the latest event while newer holders are offline or withholding data.
- No browser or mobile adaptation may introduce behavioral profiling or engagement ranking.

## What is genuinely portable

Keep these modules free of Electron, DOM, React, React Native, Node-only built-ins, absolute storage paths, and ambient `Pear` globals:

1. Protocol version, event type definitions, field limits, and deterministic event encoding.
2. Identity/device proof and signature verification, post identifiers, invite payload parsing, and identity-address formatting.
3. Pure validation and materialization rules for profiles, posts, edits, deletion notices, follows, recommendation sources, mute/block, and starter packs.
4. Following selection and deterministic chronology, including the future-clock clamp and tie-break rules.
5. Two-hop positive-trust discovery, normalization/capping, local exclusions, and human-readable recommendation-path data.
6. Corestore/Hypercore replication and local indexes **when they run inside Bare and receive storage/lifecycle inputs from the host**. Pear explicitly documents that a mobile Bare worklet can open Corestore and Hyperswarm as on desktop ([Embed Bare in a React Native app](https://docs.pears.com/how-to/run-on-native/embed-bare-in-react-native/)); the Pear stack describes these as ordinary JavaScript modules that run under Bare across hosts ([The Pear stack](https://docs.pears.com/explanation/the-pear-stack/)).
7. A versioned, bounded command/event schema between shell and core. Raw IPC framing is host-specific, but operation names, payload schemas, limits, error codes, and stream semantics are shared. Pear recommends a typed RPC seam and documents unary, event, and streaming patterns ([One core, many platforms](https://docs.pears.com/explanation/bare-on-native/), [Bare Kit](https://docs.pears.com/reference/bare/bare-kit/)).

Treat IPC as a byte stream, not an atomic message bus. Bare Kit documents non-blocking reads/writes that may complete partially, so each host must use a correct framing/RPC implementation rather than assuming one write equals one command ([Bare Kit](https://docs.pears.com/reference/bare/bare-kit/)).

Bare is similar to Node, not identical: it has a minimal global API and supplies standard functionality through installable `bare-*` modules. Portable core code must use modules documented for Bare or a verified compatibility shim, not assume arbitrary Node APIs exist ([Runtime and languages](https://docs.pears.com/explanation/runtime-and-languages/), [Bare modules](https://docs.pears.com/reference/modules/bare-modules/)). TypeScript is portable only after compilation to JavaScript; native addons must be built for every target.

## What remains host-specific

| Boundary | Windows MVP | Later mobile | Possible browser UI |
| --- | --- | --- | --- |
| Shell/UI | Electron renderer, HTML, semantic CSS/Tailwind, DOM accessibility | New React Native/Expo UI with native navigation, accessibility, safe areas, keyboard, share sheet, QR/camera, and app links | Web UI may reuse information architecture and perhaps web components, but needs a different transport/storage host |
| Core host | Electron main starts a Bare worker with `pear-runtime` | React Native/Expo starts a Bare Kit worklet | No supported Bare/Pear host in an ordinary browser |
| Privileged seam | Electron `contextBridge`/IPC/preload | `react-native-bare-kit` IPC with typed RPC | HTTPS/WebSocket/WebRTC bridge, depending on the chosen architecture |
| Storage root | `pear.storage`/Windows application data supplied by `pear-runtime` | App-owned native data directory supplied to the worklet | Origin storage or remote gateway; neither has desktop Corestore's availability assumptions |
| Lifecycle | Window/app shutdown and desktop network lifecycle | Explicit foreground, suspend, wake, and resume handling | Page/service-worker lifetime is controlled by the browser |
| Packaging/updates | Electron packaging plus Pear P2P update flow | Store/native build pipeline; Bare bundle and linked addons | Conventional web deployment, browser security model, and gateway deployment if used |

The desktop renderer is therefore reusable **design**, not guaranteed reusable code. Copy, information hierarchy, semantic design tokens, responsive breakpoints, empty/error states, and accessibility acceptance tests can guide each UI. DOM components, Tailwind class composition, shadcn/ui source, Electron preload calls, focus assumptions, and Windows-only interactions must be re-evaluated or rewritten for React Native. Pear's own guidance says the UI is platform-specific while the Pear-end is shared ([Runtime and languages](https://docs.pears.com/explanation/runtime-and-languages/)).

Windows-only adapters include Electron window/security configuration, preload exposure, deep-link registration, native file dialogs and recovery-file handling, Windows credential protection, application-data paths, installer/signing, and `pear-runtime` updates. None belongs in the core.

## Later iOS and Android architecture

Use a React Native or Expo shell with `react-native-bare-kit`. Start the shared core as a separate Bare worklet and communicate through a bounded typed RPC seam. Mobile does not currently support `pear-runtime`; the app must wire the worklet and its update/distribution path itself ([One core, many platforms](https://docs.pears.com/explanation/bare-on-native/)).

The mobile build must:

- produce a platform-targeted Bare bundle with `bare-pack`; `--linked` is required because iOS and Android link native addons ahead of time rather than loading them from disk ([Bundle a Bare app](https://docs.pears.com/how-to/run-on-native/bundle-a-bare-app/));
- inject an app-owned storage directory instead of importing desktop path logic;
- connect host background/foreground events to worklet suspend/resume;
- on suspend, pause Hyperswarm and flush Corestore before the event loop becomes idle; otherwise the OS may terminate the app and recent buffered writes may be lost ([Handle app suspension](https://docs.pears.com/how-to/run-on-native/handle-app-suspension/));
- assume mobile is not an always-on seeder. While suspended, its swarm connections and DHT socket are paused, so availability must come from another online holder or an explicit host peer;
- implement native recovery export/import, app links, QR scanning, share actions, notifications, secure local-secret storage, accessibility, and platform release signing as mobile adapters.

This lets mobile reuse the protocol and core while giving its UI and lifecycle honest platform implementations. It does not make the Electron renderer portable.

## Browser architectures

The current Pear-end cannot simply be bundled into a web page. Hyperswarm uses HyperDHT networking in Bare, while browser networking exposes different transports and lifecycles. WebRTC peers require out-of-band signaling and may require STUN/TURN relays ([WebRTC 1.0](https://www.w3.org/TR/webrtc/)); WebSocket is a connection from a browser client to a server endpoint ([WebSockets Standard](https://websockets.spec.whatwg.org/)). The conclusion that an ordinary page cannot directly run the current HyperDHT/Hyperswarm transport is an inference from those platform APIs and Pear's documented Bare-host architecture, not a claim that peer-to-peer browser applications are impossible.

Three viable directions remain:

### 1. Local companion bridge — preferred if browser access is needed soon

The installed desktop app keeps the Bare core, keys, Corestore, replication, validation, and ranking. A browser UI connects to a narrowly authenticated local bridge.

- **Trust/privacy:** closest to the Windows model; data and ranking remain local. The bridge creates a sensitive localhost/origin boundary and must resist cross-site requests, origin confusion, and unauthorized commands.
- **Availability:** no better than the desktop core; the browser view works only while the companion is running.
- **Guarantees:** direct-follow and local-ranking guarantees remain intact because the same verified local core answers the UI.
- **Cost:** two installed pieces and browser/companion pairing; it is not a browser-only product.

### 2. User-chosen remote gateway — practical browser-only access, weaker trust

A gateway runs the Bare core and exposes a signed, bounded API over HTTPS/WebSocket. The browser verifies event signatures and computes presentation/ranking locally where practical. Identity signing keys should remain client-held; placing them on the gateway turns it into an account custodian.

- **Trust/privacy:** the gateway learns IP address, request timing, followed discovery topics, and the public data it serves. It can omit, delay, or selectively serve valid events even if it cannot forge signatures. Multiple or user-hosted gateways reduce reliance but do not prove completeness.
- **Availability:** can be strong if continuously hosted, but “serverless” no longer describes the browser path. Hosting and abuse costs reappear.
- **Guarantees:** the browser can guarantee that it does not algorithmically suppress any **verified posts the gateway supplied**. It cannot honestly guarantee receipt of every available followed post from a gateway that withholds data.
- **Cost:** gateway protocol, deployment, authentication, rate limits, operations, and clear disclosure of the changed trust boundary.

### 3. Browser-native WebRTC network — long-term research, not an adapter

Build a separate transport using WebRTC data channels plus signaling and possibly TURN, then adapt or redesign replication above it.

- **Trust/privacy:** signatures and local ranking can remain client-side, but signaling/relay operators observe metadata and TURN may carry traffic.
- **Availability:** tabs and service workers are not dependable seeders. User agents may terminate service workers when idle ([Service Workers](https://w3c.github.io/ServiceWorker/#service-worker-lifetime)), and best-effort origin storage can be evicted under storage pressure unless persistence is granted ([Storage Standard](https://storage.spec.whatwg.org/)).
- **Guarantees:** direct-follow selection can remain deterministic over retrieved valid events, but retrieval and continuous hosting are weaker.
- **Cost:** a new transport, signaling/relay infrastructure, interoperability work, browser storage migration, and separate conformance tests. This is not automatic Hyperswarm compatibility.

Do not choose a browser architecture in the Windows MVP specification. Preserve the typed seam and signed event protocol so a later investigation can compare a local companion with user-chosen gateways using a working network and measured requirements.

## Test boundary

Shared conformance fixtures should assert the same bytes and outcomes in desktop Bare and later mobile Bare:

- valid/invalid event vectors and signature proofs;
- edit/delete ownership and ordering;
- invite parsing and limits;
- Following completeness and chronology over the same local event set;
- Discover eligibility, two-hop decay, normalized influence, exclusions, chronology, and explanations;
- RPC schema compatibility and malformed/oversized message rejection.

Keep host integration checks separate: Electron sandbox/preload security on Windows; bundle/addon linking, storage paths, suspend/resume, and forced termination on mobile; authentication, omission behavior, storage eviction, and signaling/relay behavior for any browser architecture. Passing shared unit tests does not prove a host preserves availability or lifecycle behavior.

## Newly surfaced fog

1. Should browser access first target a local companion or a user-owned remote gateway, and what minimum disclosure makes its weaker availability/completeness guarantee understandable?
2. Which schema tool and wire format will define the typed seam without forcing Electron-only or React-Native-only dependencies into the core?
3. Which current cryptography, encoding, indexing, and test dependencies are verified on every intended Bare mobile target and architecture?
4. What exact mobile storage directory, OS-backed secret store, backup/export path, and lifecycle test matrix are required for iOS and Android?
5. How are mobile bundle updates coordinated with protocol migrations while `pear-runtime` is unavailable there?
6. Does a later host peer need a browser-facing read API, or should gateway and availability hosting remain separate roles?
