# Protected profiles - later MVP

## Purpose

Protected profiles support friends, families, and private communities whose posts should be readable only by identities the publisher approves.

## User promise

- A person requests access to a protected profile.
- The publisher explicitly approves or declines the request.
- Only approved identities receive the information needed to decrypt future protected posts.
- Removing an identity prevents it from decrypting future posts after access rotates.
- The product never promises that removing access deletes posts the former follower already received, decrypted, copied, or captured.

## Required design work

- Encrypted post and audience-key format.
- Follow-request and approval events that do not expose more of the social graph than necessary.
- Adding, removing, and rotating audience access across multiple authorized devices.
- Recovery behavior when the publisher restores their identity.
- Metadata and traffic-analysis expectations for publishers and followers.
- Local caching, export, screenshots, and deletion wording.
- Interaction between protected posts, host peers, starter packs, and Discover. Protected posts must not enter public discovery.

## Scope decision

Protected profiles are not part of the first public-text Windows MVP. They require a separate threat model and protocol decision after public identity, posting, following, replication, and recovery work reliably.
