# Peer social network

A peer-to-peer social network where readers explicitly choose publishers and control local discovery through positive trust relationships.

## Language

**Identity**:
A cryptographic public key representing a publisher or reader. A username is mutable profile metadata, not the identity.
_Avoid_: Account

**Username**:
A human-readable, non-unique name attached to an identity.
_Avoid_: Handle, unique username

**Publisher**:
An identity that signs and announces public posts.
_Avoid_: Content creator, account

**Reader**:
An identity viewing followed or discovered posts on their device.
_Avoid_: Consumer

**Follow**:
A reader's direct subscription to a publisher whose available posts must appear in the Following feed.
_Avoid_: Interest signal

**Trust**:
A reader's positive, explicit opinion that may contribute to local discovery recommendations.
_Avoid_: Like, popularity, global reputation

**Recommendation source**:
An identity whose positive trust choices a reader explicitly allows to supply candidates to their Discovery feed.
_Avoid_: Trusted follow, influencer

**Direct block**:
A private local decision to hide an identity and its posts. It does not alter what other readers can access.
_Avoid_: Ban, takedown

**Following feed**:
A chronological view of available public posts from directly followed identities.
_Avoid_: For You feed, ranked feed

**Discovery feed**:
An optional local view of posts recommended through bounded positive trust propagation and recency.
_Avoid_: Main feed, global feed

**Host peer**:
A participant that retains and serves replicated data to keep it available.
_Avoid_: CEO, platform owner, central server

**Available post**:
A signed public post currently retrievable from at least one reachable peer.
_Avoid_: Permanent post

**Censorship resistance**:
The absence of a central authority able to remove every copy or silently prevent a reachable followed publisher from appearing.
_Avoid_: Censorship-free, censorship-proof
