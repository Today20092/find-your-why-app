# Windows core journey

## Chosen direction

Use the prototype's **Rail + context** navigation combined with the **Calm stream** content presentation.

- Desktop uses a persistent left rail with Following, Discover, Find people, New post, and identity access.
- The main area is one calm, readable feed column.
- Following and Discover remain separate destinations.
- Do not use a persistent right sidebar. Recovery, identity, network, storage, and peer details belong in Settings or appear inline only when they require action.
- Discovered posts explain their source inline, such as “Recommended through Maya,” with optional details.

## Responsive behavior

### Wide desktop

- Persistent labelled left rail.
- Centered feed with a comfortable reading width.
- Empty space remains empty; do not fill it with low-value cards.

### Compact desktop and tablet widths

- Collapse the rail to icons with accessible names and tooltips.
- Keep the feed as a single column.

### Mobile widths

- Replace the rail with bottom navigation for Following, Discover, Find people, and Profile.
- Use a prominent compose action without covering content.
- Keep one content column with touch targets at least 24 by 24 CSS pixels and preferably 44 by 44.
- Dialogs become full-width sheets only when that improves comprehension; recovery remains a dedicated flow.

## Information placement

- Show “Waiting for this person to come online” or other availability text inline only when content cannot load.
- Put healthy peer counts, replication, storage, identity keys, recovery management, and hosting controls under Settings.
- Keep the ordinary experience focused on people and posts, not peer-to-peer infrastructure.

## Frontend implication

This structure does not require React by itself. Start with semantic HTML, locally compiled Tailwind CSS, and a small amount of renderer JavaScript. Adopt React and selected shadcn/ui source components only if implementation exposes state complexity that materially reduces clarity without them.
