# MVP privacy and abuse threat model

Status: implementation constraint for the Windows, text-only MVP. Review whenever a trust boundary, event type, hosting mode, or platform changes. The structure follows OWASP's system-model → threats → mitigations → review loop ([OWASP Threat Modeling](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)).

## Security and privacy goals

Protect the identity signing authority, recovery secret and passphrase, private reader choices, local database, and the integrity and availability of signed public events. Prevent untrusted content from gaining operating-system privileges. Minimize linkable network and social-graph metadata.

Public posts and profiles are intentionally public. The product does not promise anonymity, confidential publishing, guaranteed availability, guaranteed deletion, global spam prevention, or safety from a compromised Windows account.

## System and trust boundaries

```text
untrusted publisher data / DHT / remote peers
                    |
          encrypted peer connection
                    |
      Bare worker: validate, replicate, index
                    |
       narrow validated message protocol
                    |
  Electron main + preload: privileged broker
                    |
     sandboxed renderer: render text only
                    |
        local Windows user and storage
```

- **Renderer:** untrusted-content display boundary. It receives inert, length-bounded data and has no Node, Pear, filesystem, shell, or generic IPC access.
- **Electron main/preload:** privileged boundary. It exposes named operations only and authenticates the sending frame.
- **Bare worker:** network/data boundary. Every peer, event, clock, starter pack, and replicated byte is untrusted until bounded and verified.
- **Local Windows profile:** protects ordinary at-rest state from other accounts, not from the signed-in user, administrators, malware, debuggers, or memory inspection.
- **DHT and transport:** peers exchange bytes over encrypted streams, while Hyperswarm uses a 32-byte topic—commonly a Hypercore discovery key—to find peers ([Pear P2P explanation](https://docs.pears.com/explanation/peer-to-peer-demystified/)). Encryption protects payloads in transit; it does not hide connection timing, IP addresses, traffic volume, or topic participation from every observer.
- **Host/blind peer:** improves reachability but is outside the reader's device and must be treated as an untrusted availability provider. Blind peering is explicitly configured using blind-peer public keys ([Pear blind peering](https://docs.pears.com/how-to/blind-peering/keep-data-available-with-blind-peering/)).

## Data visibility decision

| Data | MVP visibility | Reason |
|---|---|---|
| Profiles, posts, edit/delete notices | Public, replicated | They are the public publishing product. |
| Follow/unfollow | Private local state, selectively synchronized only between the identity owner's authorized devices | Following is a reading subscription, not an endorsement. Publishing it would expose associations and enable graph scraping. It is removed from the public event schema. |
| Mute, block, dismiss, hide-from-Discover | Private local state; never sent to publishers, peers, starter-pack authors, or recommendation sources | These are personal safety choices and must not become retaliatory or propagated reputation signals. |
| Positive trust endorsement | Public, signed, explicitly published | Two-hop discovery requires the next reader to retrieve and verify a recommendation source's outgoing endorsements without a central service. The UI must say that enabling “Use my recommendations” publishes that relationship. Removing it appends a public revocation but cannot erase old copies. |
| “Use this person's recommendations” | Private local choice | It selects the reader's roots; publishing it would reveal additional graph edges without helping propagation. |
| Computed scores, paths, feed state and behavior | Private local derived data | No behavioral dossier or centralized ranking service is needed. |

Public trust is a deliberate disclosure, not a covert side effect of Follow. A future privacy-preserving/selective trust protocol would need a separate design; encrypting endorsements only to current readers breaks permissionless two-hop discovery and forward sharing.

Private choices must live in a separate local store from public Hypercores. Authorized-device encrypted synchronization and recovery of this store are not designed yet; until they are, the UI must say that restoring identity authority may not restore private preferences.

## Observers and capabilities

- **Remote peer:** sees its direct connection metadata and any public cores exchanged; may send malformed, oversized, duplicated, stale, or strategically withheld data.
- **DHT/discovery participant or network observer:** may correlate source IP, time, volume, and discovery-topic activity. A discovery key avoids announcing the Hypercore key itself, but is still a stable rendezvous topic, not an anonymity system.
- **Followed identity:** learns nothing directly from a local follow, but may infer readership from direct availability connections or later interactions.
- **Recommendation source:** can publish endorsements, churn them, recommend Sybils, or withhold its log. It does not see who locally enabled it merely from that choice.
- **Host peer:** sees requested public discovery topics and stores/serves selected public bytes; it can log, withhold, delay, or disappear.
- **Local user, administrator, or malware:** can read displayed/public data and may steal local secrets or manipulate the client. OS-backed encryption reduces offline cross-account disclosure but cannot defeat code executing as the user.

## MVP threats and required mitigations

### Identity and local secrets

- Store identity/writer secrets outside the renderer and protect the at-rest wrapping key with **current-user Windows DPAPI**, without `CRYPTPROTECT_LOCAL_MACHINE`. Microsoft says DPAPI normally restricts decryption to the same user's logon credentials on the same computer and adds an integrity check ([CryptProtectData](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)). Restrict filesystem ACLs as defense in depth.
- Never persist or log plaintext private keys or recovery passphrases. Keep secret lifetimes short and clear buffers where runtime APIs permit. OWASP classifies private keys and passwords as secrets that must not be logged ([OWASP Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)).
- The exported recovery file remains password-encrypted with authenticated encryption as specified by the recovery decision. Theft permits offline passphrase guessing; strong passphrases and benchmarked Argon2id parameters mitigate but do not eliminate it.
- Treat a stolen active identity key as full impersonation authority. Production release is blocked on the already identified recovery-epoch, device-revocation, writer-rotation, and split-brain design; the MVP must not claim that compromise can be remotely undone.

### Malicious, replayed, and rolled-back events

- Accept only known version/type pairs; canonical encoding; valid Hypercore proof and application identity/device proof; authorized device; exact field types; Unicode normalization; byte/count bounds; and author ownership for edits/deletes. Reject unknown fields where ambiguity could change signed meaning.
- Render post/profile text as text (`textContent`), never HTML. Do not resolve markup, scripts, remote embeds, custom URL schemes, or media in the MVP.
- Use Hypercore block index as per-writer order; keep the highest verified contiguous length/checkpoint locally. Duplicate event IDs are idempotent. Reject a claimed earlier state as current, while permitting peers to fill genuinely missing blocks. Hypercore is an authenticated append-only log backed by signed Merkle trees ([Hypercore](https://github.com/holepunchto/hypercore)); it authenticates inclusion, not application meaning or freshness.
- Publisher timestamps are untrusted and only approximate cross-publisher chronology; clamp future timestamps as already decided. Bound events per processing batch, bytes per peer, concurrent replication, index growth, and retry/backoff so valid signatures cannot become CPU/disk denial of service. OWASP recommends limiting request size and input-driven resource allocation ([OWASP DoS](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html)).

### Network observation, scraping, and availability attacks

- State plainly in onboarding/settings: peers and network intermediaries may observe IP/network metadata; a VPN/Tor claim is out of scope and unsupported. Do not call encrypted transport anonymous or metadata-private.
- Join discovery topics only for active publishing, explicit follows, or explicit hosting; leave them when no longer needed, avoid global prefetch, and do not publish the local follow list. This reduces, but does not prevent, correlation.
- Assume all public profiles, posts, trust endorsements, starter packs, and historical revocations can be crawled and archived. Short discriminators are labels, not secrets.
- Detect eclipse/withholding only as availability uncertainty: use multiple independently discovered peers where available, retain verified local data, use bounded reconnect/backoff, and show “not currently available” rather than “deleted” or “does not exist.” No client can prove it has seen the latest block while every newer holder withholds it.
- Host peers are opt-in and public-data-only in the MVP. They receive no identity secrets or private graph. Their disappearance reduces availability; their possession makes later erasure impossible to guarantee.
- A deletion is a signed hide notice for compliant clients. Original blocks, screenshots, exports, caches, and hostile archives may persist.

### Spam, Sybils, and social abuse

- Unlimited cheap identities are expected. Unknown identities get no automatic Discover reach. Following is explicit; Discover is optional and starts empty.
- Trust propagates only positively, at most two hops with `0.5` decay. Each source's influence is divided across its outgoing endorsements, independent paths combine, and influence is capped at `1.0`. A local block/hide overrides eligibility. These controls contain reach; they do not establish that a person is honest or unique.
- Apply per-peer transfer/processing quotas and local backoff. Do not add proof-of-work, global identity limits, global bans, negative propagation, or central reputation in the MVP.
- Starter packs are signed public documents, never privileged code or authority. Show author/key, contents, age, and requested effects before one explicit import confirmation; import neither follows nor recommendation roots silently. Packs can be malicious, stale, cloned, or compromised, and their authors gain no permanent status.

### Electron and content execution

- Package and load only local application code. Do not display remote pages in the app. Keep `nodeIntegration: false`, `contextIsolation: true`, renderer sandboxing enabled, `webSecurity` enabled, and a restrictive CSP such as `default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'none'; frame-src 'none'`.
- Disable unexpected navigation and new windows; do not use `<webview>`; parse and allowlist any external URL before opening it in the system browser; never pass peer-controlled strings to shell commands.
- The preload exposes one typed method per product operation, never `ipcRenderer`, generic `send`, filesystem paths, or arbitrary channels. Validate sender frame and every argument again in main/worker. Electron's security checklist requires isolation, sandboxing, CSP, navigation limits, current releases, and IPC sender validation ([Electron Security](https://www.electronjs.org/docs/latest/tutorial/security)); its context-isolation guide specifically warns against exposing raw IPC ([Electron Context Isolation](https://www.electronjs.org/docs/latest/tutorial/context-isolation)).
- Keep Electron, Chromium, Pear runtime, and dependencies patched; minimize dependencies and review lockfile changes.

### Logs, diagnostics, and telemetry

- No analytics or remote telemetry in the MVP. Local structured security logs may contain event category, coarse time, validation failure code, and bounded counters only.
- Never log post/profile text, private graph choices, public/private keys in full, discovery keys, IP addresses, invite/recovery material, passphrases, recovery paths, clipboard contents, raw IPC payloads, or feed behavior. Sanitize peer-controlled text to prevent log injection; cap log size and retention.
- Diagnostic export is explicit, previewable, redacted, and saved locally. OWASP notes that external-zone log data may be forged/replayed and recommends excluding secrets and sensitive personal data, sanitizing untrusted fields, and preventing logs from exhausting disk ([OWASP Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)).

## Residual risks accepted for MVP

- IP, timing, traffic-volume, and discovery-topic correlation; targeted peer enumeration and public-graph scraping.
- Sybil identities, harassment, unlawful content, recommendation-source compromise, and social engineering. Local controls reduce exposure, not publication.
- Offline key extraction or identity takeover by malware/admin access; offline guessing of a stolen recovery file.
- Withholding, eclipse, stale views, unavailable publishers, host-peer disappearance, disk exhaustion attempts, and clock manipulation within the stated clamps.
- Permanent third-party copies despite edit/deletion notices, and incompatible clients ignoring local conventions.
- Loss of private follows/preferences after disaster recovery until encrypted authorized-device sync/backup exists.

## Forbidden product claims

Do not say **anonymous**, **untraceable**, **private by P2P**, **end-to-end encrypted posts** (public posts are replicated public data), **censorship-free**, **censorship-proof**, **moderation-free**, **spam-proof**, **bot-proof**, **always available**, **permanent**, **fully deleted**, **globally deleted**, **identity theft recoverable**, **backup guaranteed**, or **we can restore your identity**.

Permitted phrasing: “No central feed operator,” “public signed posts,” “encrypted peer connections,” “local private reading choices,” “censorship-resistant while reachable copies exist,” and “a deletion notice asks compatible clients to hide a post but cannot erase every copy.”

## Release gates and newly surfaced questions

Before production rather than a disposable prototype:

1. Define identity epochs, writer replacement/revocation, rollback checkpoints, and split-brain convergence.
2. Define encrypted authorized-device synchronization and recovery expectations for private follows and filters.
3. Specify exact per-event, per-peer, storage, retention, and retry limits and leave one abuse/resource test for each parser or bounded queue.
4. Decide whether trust disclosure deserves a separate first-use confirmation and how public revocation history is explained.
5. Threat-model deep links/invite parsing and the future host-peer protocol when their formats exist.
6. Establish dependency-update, vulnerability-response, diagnostic-redaction, and security-reporting procedures before public distribution.

Review this model with adversarial tests for malformed events, replay/rollback, oversized graphs, hostile starter packs, renderer injection, IPC sender spoofing, secret/log leakage, and peer withholding. Threat modeling is iterative, not a one-time claim of safety.
