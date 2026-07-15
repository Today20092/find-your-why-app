# Product vision

## Why this exists

Mainstream social platforms optimize feeds for attention, advertising, shopping, and platform revenue. They infer interests from watch time, likes, clicks, and follows, then use those profiles to maximize continued engagement.

This app exists to provide a social space without an engagement-maximizing intermediary. People choose whom they follow, always receive those authors' available public posts, and may discover related voices through an understandable trust network they control.

## Product promise

- **Following is a guarantee, not a signal.** Available public posts from directly followed identities appear in the chronological following feed. Trust scores cannot suppress them.
- **Discovery is chosen and explainable.** Positive trust may recommend posts from identities trusted by people the reader trusts. Propagation is bounded, decays with distance, and is computed locally.
- **No behavioral profiling.** Watch time, dwell time, clicks, and inferred interests do not rank the feed or build an advertising profile.
- **No engagement objective.** The app does not optimize session length, retention, compulsive use, ad impressions, or purchases.
- **No central feed authority.** Each reader controls their follows, local filters, trust roots, and ranking parameters.
- **User-owned identity and data.** Cryptographic keys identify publishers. Signed public data is replicated between peers rather than owned by a central platform.
- **Open participation.** Any person or community may publish, follow, host, and build a compatible client without permission from a central operator.

## Feed contract

The home experience has two explicit views:

1. **Following:** chronological available posts from directly followed identities. Nothing is algorithmically removed.
2. **Discovery:** optional posts ranked by recency and positive trust propagated at most two hops with decay.

Direct mute and block choices apply locally and immediately. Negative trust does not propagate in the MVP because propagated accusations can be weaponized for coordinated exclusion. Direct choices override recommendations.

## Data availability and censorship resistance

Publishing creates signed data that other peers can copy and verify. No central operator can delete every copy or prevent a user from following a reachable publisher.

This is censorship-resistant, not censorship-proof:

- A device may refuse to display or store any content.
- A publisher may stop announcing data or lose their keys.
- Data becomes unavailable when no reachable peer retains and serves a copy.
- Illegal or harmful material remains subject to device owners' choices and applicable law.

Hosting is voluntary. People and communities can keep material available by running peers; data with no willing host naturally disappears from the reachable network.

## Privacy

The app avoids centralized behavioral dossiers and keeps private preferences local. Public posts, public profiles, and public trust statements should be treated as public replicated data. Peer-to-peer transport does not by itself provide anonymity; network metadata and public graph relationships may still reveal information.

## MVP boundary

The first release is desktop-first and text-only. It includes local identity creation, a username/display name, signed posts, following, direct trust, direct mute/block, chronological following, and bounded trust-based discovery.

It excludes ads, shopping, engagement analytics, media, global moderation, propagated distrust, global reputation, and claims of globally unique usernames.

## Decision test

When considering a feature, ask:

1. Does it help people intentionally connect, publish, follow, discover, or host?
2. Does it preserve the direct-follow guarantee and local user control?
3. Can it work without behavioral profiling or an engagement/revenue objective?
4. Does it avoid creating a central authority capable of silently controlling reach?

If not, it conflicts with the product's purpose.
