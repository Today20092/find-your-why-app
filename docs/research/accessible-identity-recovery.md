# Accessible identity recovery decision

## Decision

The Windows MVP uses a **password-encrypted recovery file** as its sole disaster-recovery method. Recovery words are not generated, and online device pairing is a separate convenience for adding a device while an authorized device still works.

An identity may be created immediately, but it remains **not backed up**. In that state the person may browse public content and change disposable local profile fields, but cannot publish, follow, trust, mute, or block. Those actions create identity-bound state that a device loss would strand. Activation requires this ceremony:

1. Explain, before asking for a password: “There is no company account and no reset service. If every authorized device and recovery file is lost, this identity cannot be recovered.”
2. Ask the person to save a versioned encrypted recovery file and create a strong recovery passphrase. Permit password-manager generation, autofill, paste, and a show/hide control.
3. Clear the in-memory generated download from the UI, then ask the person to select the saved file from disk and enter its passphrase.
4. Decrypt and authenticate the selected bytes, verify the embedded public identity matches the newly created identity, and show the destination path the person selected.
5. Ask for an explicit checkbox: “I understand this verifies this copy on this device; I still need another safe copy.” Only then activate identity-bound actions.

This is a verification of **file integrity, possession at that moment, and passphrase correctness**. It is not described as proof of an off-device backup, protection from malware, or elimination of project liability.

## Backup format and local storage

The export is a small, self-describing binary container with a fixed magic value and version, KDF identifier and parameters, random salt, AEAD identifier and nonce, public identity key, and authenticated ciphertext. The ciphertext contains the minimum secret recovery authority and writer metadata required to restore the identity; it does not contain cached posts or other people's data.

Use Argon2id for password-based key derivation. RFC 9106 requires Argon2id support, recommends a unique 16-byte salt for password hashing, and supplies parameter profiles and test vectors. Choose the concrete memory/time parameters only after measuring the supported low-end Windows hardware; store them in every file so future clients can decrypt older exports ([RFC 9106](https://www.rfc-editor.org/rfc/rfc9106.html)). Use an authenticated-encryption construction from a maintained crypto dependency rather than handwritten cryptography; the final container algorithm and library remain an implementation ticket.

The normal on-device secret store is distinct from the portable export. On Windows, protect local static key material using the current Windows user context rather than leaving plaintext secrets in the Pear storage directory. Microsoft documents that `CryptProtectData` normally restricts decryption to the same logon credentials and computer and adds an integrity check ([Microsoft `CryptProtectData`](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)). That device-bound protection cannot replace the recovery file because it normally cannot be decrypted on another computer.

Do not write the recovery passphrase to disk, logs, analytics, crash reports, clipboard history, or the recovery container. Clear secret buffers where the selected crypto/runtime APIs permit it.

## Why not recovery words

A 12/24-word ceremony is not inherently safer or more recoverable. A mnemonic would require selecting and documenting a compatible entropy-to-words and key-derivation specification, language/word-list behavior, checksum handling, and how it maps to this app's identity authority. It also encourages manual transcription as the only route. None of that is needed for a Windows-first file backup.

Recovery words may be reconsidered only if user research shows a real need for paper-only recovery or a later mobile ecosystem requires an interoperable mnemonic. It must then be an additional recovery representation with test vectors, not an improvised list of words.

## Accessibility requirements

Do **not** block copy, paste, password-manager fill, or file selection. Do not ask for selected word positions, memory questions, CAPTCHAs, or forced manual retyping. W3C explains that blocking paste or requesting selected characters forces transcription and fails Accessible Authentication (Minimum) unless another compliant path exists ([WCAG 2.2 SC 3.3.8 explanation](https://www.w3.org/WAI/WCAG22/Understanding/accessible-authentication-minimum.html)). WCAG also aims to avoid redundant entry in multi-step processes, while recognizing limited security-essential exceptions ([WCAG 2.2 SC 3.3.7 explanation](https://www.w3.org/WAI/WCAG22/Understanding/redundant-entry.html)). The selected-file round trip is essential here because it validates the artifact that recovery actually consumes, but the app still allows assistive tools for the passphrase.

All steps need programmatic labels, keyboard operation, visible focus, inline error identification, status announcements, and no timeout. Errors distinguish “unsupported/corrupt file,” “incorrect passphrase,” and “identity does not match” without exposing secret material. The person can go back, save another file, or change the passphrase without losing the newly created local identity.

NIST treats recovery codes as secrets used to regain account access and requires recovery verification to be protected against repeated guessing ([NIST SP 800-63B §4.2](https://pages.nist.gov/800-63-4/sp800-63b.html#account-recovery)). This application has no remote verifier to throttle, so offline attack resistance must come from a strong user passphrase, a memory-hard KDF, authenticated encryption, and conservative import error behavior.

## Device pairing is not recovery

Pairing is offered later as “Add another device” only when an existing authorized device can approve it. The Pear production-chat flow uses Blind Pairing to admit a candidate writer and disclose the keys it needs; that is an appropriate model for live authorization, not proof that an unattended recovery path exists ([Pear production-app pairing](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/reshape-into-a-production-app/)). The UI must never call pairing a backup.

## Restore and writer rotation

The stable `identityKey` remains the social identity. A recovery bundle carries the authority needed to authorize a replacement device; it must not make a new display identity. On import:

1. authenticate the container before parsing secret fields;
2. derive and display the same `identityKey` and address discriminator for confirmation;
3. create a fresh device key and writer core;
4. produce a root-authorized `writer-replaced` statement linking the prior writer/core, the replacement writer/core, and a monotonic recovery epoch;
5. do not publish until the replacement announcement is durably stored locally.

Readers accept the highest valid recovery epoch and treat lower-epoch writer events after its stated cutoff as stale. This makes the ceremony compatible with recovered-identity writer rotation and avoids copying one live writer secret to multiple devices.

However, the signed-events decision correctly identifies a remaining availability problem: followers need a deterministic place to discover the replacement announcement when the old core is permanently unavailable. The production recovery implementation is therefore blocked on a separate decision for a stable identity-manifest/rendezvous log and split-brain reconciliation. The first disposable technical prototype may omit recovery; it must not ship a “Restore” promise that merely creates an unreachable branch.

## Lost-backup behavior and required wording

If all authorized devices and usable recovery files are lost, the only action is **Create a new identity**. Support documentation may help diagnose a file but cannot recreate keys, reset a passphrase, merge identities, reclaim followers, or guarantee restoration of unavailable posts.

The UI must say:

- “Your recovery file and its passphrase are both required.”
- “Keep another copy somewhere you control.”
- “Anyone with both can act as you.”
- “Losing every authorized device or either recovery requirement can permanently lose access.”
- “Deleted or unavailable peer data may not return when your identity is restored.”

It must not say “account secured,” “backup guaranteed,” “censorship-proof,” “recoverable forever,” or “we can restore your account.”

## Newly surfaced fog

1. Define the stable identity-manifest/rendezvous log, replacement-writer discovery, recovery epochs, cutoff semantics, and deterministic split-brain reconciliation before production recovery ships.
2. Select and audit the maintained Argon2id + AEAD implementation available in the chosen Bare/Electron runtime; benchmark KDF parameters on supported Windows hardware and publish format test vectors.
3. Decide whether a future live-paired device can co-sign recovery or revoke a stolen recovery authority without introducing a central service.
4. Test the ceremony with screen-reader, keyboard-only, cognitive-accessibility, password-manager, and low-vision users; the checkbox wording and activation boundary are product hypotheses, not proof that people made a second copy.

