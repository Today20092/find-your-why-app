# Frontend reference library and design constraints

This file is the implementation contract for the Windows MVP UI and the intake point for future visual references. It is not a mood board. A reference earns a place here only when it demonstrates a concrete behavior or constraint we intend to test.

## Decision

Build a familiar, quiet social interface whose peer-to-peer machinery is invisible until it affects the reader. Use semantic HTML first, Tailwind CSS for layout and presentation, and a small set of semantic CSS theme tokens. Do not install shadcn/ui in the framework-free renderer: shadcn/ui distributes React component source, while the chosen Electron scaffold currently uses plain HTML, CSS, and JavaScript. If the UI prototype later justifies React, copy only the shadcn components actually needed and treat them as owned source—not as the product's visual identity.

Following and Discover are separate destinations. Following contains available posts from explicitly followed identities in newest-first order. Discover is optional, newest-first, and every item explains its recommendation path. Neither view uses engagement ranking, infinite attention traps, or hidden personalization.

Sources: [Pear Windows architecture decision](../../research/pear-windows-architecture.md), [Tailwind installation](https://tailwindcss.com/docs/installation/tailwind-cli), [Tailwind theme variables](https://tailwindcss.com/docs/theme), [shadcn/ui introduction](https://ui.shadcn.com/docs), and [shadcn/ui theming](https://ui.shadcn.com/docs/theming).

## Enforceable design constraints

### Information architecture

- Default to **Following**. Keep **Discover** a separate, plainly labeled destination; never mix its candidates into Following.
- Keep the text-post composer obvious but secondary to reading. Publishing, following, and enabling “Use this person's recommendations” require explicit actions.
- Show the reason beside every Discover item, for example “Recommended through Bob.” Put paths and technical detail behind a details disclosure; do not expose raw trust scores by default.
- Use familiar words in the main UI: Following, Discover, Follow, Mute, Block, and “Use this person's recommendations.” Reserve keys, cores, replication, and peer counts for Settings → Network & Storage or actionable availability messages.
- Use plain availability states: “Waiting for this person to come online,” “Available from 3 peers,” and “Saved on this device.” Never promise permanence, guaranteed recovery, or censorship-proof storage.
- Use one primary action per surface. Establish hierarchy with type, spacing, grouping, and restrained color before adding borders, shadows, decoration, or extra controls.
- Do not add Trending, watch-time signals, popularity scores, streaks, autoplay, or engagement-driven notifications.

### Responsive Windows layout

- Design mobile-narrow first even though Windows ships first. At narrow widths use one content column and an explicit navigation control; at wider widths add persistent navigation and optional contextual detail without changing reading order.
- The central reading column must remain comfortably readable rather than stretching to fill a desktop window. Text must reflow at 320 CSS pixels without two-dimensional scrolling, except content that inherently requires it.
- At 200% browser zoom and Windows text scaling, all content and actions remain available, labels do not truncate essential meaning, overlays do not cover focused controls, and no fixed-height feed clips text.
- Pointer, touch, keyboard, Windows Narrator, high-contrast/forced-colors, and reduced-motion settings are supported. Motion conveys no information by itself and is disabled or simplified under `prefers-reduced-motion`.

### Tailwind and component policy

- Compile Tailwind locally at build time; never load a CDN stylesheet or runtime compiler. Restrict source scanning to renderer files and commit the small source stylesheet, not ad hoc generated utility bundles.
- Define semantic tokens once (`background`, `foreground`, `surface`, `muted`, `border`, `primary`, `danger`, `focus`) and map them to Tailwind theme variables. Components use semantic tokens, not scattered palette literals.
- Prefer native elements (`button`, `a`, `input`, `textarea`, `dialog`, headings, lists) over ARIA replicas. CSS handles responsive layout; JavaScript handles state, not presentation breakpoints.
- Do not create a generic component layer before repeated use exists. Reuse one local primitive only after the same accessible behavior appears at least twice.
- If React is adopted, shadcn candidates are limited initially to Button, Dialog/Alert Dialog, Dropdown Menu, Tabs, Tooltip, and Toast/Sonner. Review generated dependencies and source; remove unused variants; verify against the relevant [WAI-ARIA Authoring Practices pattern](https://www.w3.org/WAI/ARIA/apg/patterns/). No registry component is accepted on appearance alone.

## Accessibility acceptance criteria

The MVP targets [WCAG 2.2 Level AA](https://www.w3.org/TR/WCAG22/) and Windows assistive technology behavior. Passing automated checks is necessary but not sufficient.

- Every function works with keyboard alone. Tab follows visual/logical order; focus is never trapped; dialogs return focus to their trigger; composite widgets use the keyboard behavior in the corresponding APG pattern. See [APG keyboard interface guidance](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface/).
- Focus is clearly visible, is not hidden by sticky navigation or overlays, and maintains at least the WCAG AA contrast requirements for non-text UI. Do not remove outlines without an equal or stronger replacement.
- Interactive pointer targets meet WCAG 2.2 AA's 24 by 24 CSS pixel minimum or its spacing exception. Prefer 44 by 44 for primary touch-capable controls.
- Normal text has at least 4.5:1 contrast; large text at least 3:1; meaningful controls, boundaries, states, and focus indicators at least 3:1 against adjacent colors. Information never depends on color alone.
- Landmarks, headings, lists, buttons, links, fields, dialogs, tabs, and status messages have correct native semantics and concise accessible names. Visible labels and accessible names agree. See [APG naming guidance](https://www.w3.org/WAI/ARIA/apg/practices/names-and-descriptions/).
- Composer character count, publishing state, replication/availability updates, errors, and successful saves are announced without moving focus (`role=status`/polite live region where appropriate). Urgent destructive errors alone use alert behavior.
- Forms use persistent visible labels, instructions before input, programmatic error association, and actionable error text. Validation never clears valid input.
- Recovery permits copy, paste, password managers, and file selection. It has no forced transcription, cognitive-function test, CAPTCHA, or timeout. The selected-file verification round trip remains keyboard and screen-reader operable.
- Zoom/reflow, forced-colors, reduced-motion, light/dark themes, long names, long unbroken identity addresses, empty/loading/offline/error states, and 200% text are checked before acceptance.
- A release candidate is manually exercised with keyboard only and Windows Narrator in addition to automated HTML/accessibility tests. Microsoft recommends treating accessibility as a core quality requirement and testing keyboard, screen readers, scaling, and high-contrast behavior ([Windows accessibility overview](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility)); [Fluent 2 accessibility](https://fluent2.microsoft.design/accessibility) reinforces clear hierarchy and inclusive interaction.

## Reference entries

### Refactoring UI

- Source: [official Refactoring UI site](https://www.refactoringui.com/)
- Borrow: practical visual hierarchy, a deliberate spacing/type scale, restrained color, and designing complete feature states before polishing decoration.
- Avoid: reproducing book text, illustrations, component examples, or paid assets; treating polish as a substitute for usability testing; importing a generic SaaS visual identity.

### Microsoft Windows and Fluent 2

- Sources: [Windows accessibility overview](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility) and [Fluent 2 accessibility](https://fluent2.microsoft.design/accessibility)
- Borrow: Windows keyboard expectations, Narrator compatibility, high contrast and scaling support, predictable hierarchy, and platform-respectful focus behavior.
- Avoid: copying XAML-only implementation advice into the Electron renderer or making the app look like a system settings panel.

### W3C WCAG and ARIA APG

- Sources: [WCAG 2.2](https://www.w3.org/TR/WCAG22/), [APG keyboard guidance](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface/), and [APG patterns](https://www.w3.org/WAI/ARIA/apg/patterns/)
- Borrow: testable acceptance criteria and conventional interaction behavior.
- Avoid: adding ARIA where native HTML already supplies the role and keyboard behavior; assuming an APG example alone proves production accessibility.

### Mastodon

- Source: [official Mastodon repository](https://github.com/mastodon/mastodon)
- Borrow: a mature open-source text-social product as a comparison point for composer, chronological reading, content warnings, profile actions, empty states, and narrow/wide navigation.
- Avoid: copying screenshots/assets, federation-specific server concepts, engagement counters as primary hierarchy, or dense multi-column layouts before user testing.

### Bluesky Social

- Source: [official Bluesky social-app repository](https://github.com/bluesky-social/social-app)
- Borrow: cross-platform social navigation, readable post anatomy, composer ergonomics, and explicit feed selection as implementation references.
- Avoid: importing algorithmic/custom-feed assumptions, protocol terminology, branding, screenshots/assets, or its React Native architecture into the Windows Pear runtime without evidence.

These products are interaction references, not templates. Our defining behavior remains the separate guaranteed Following feed, optional explainable Discover feed, local controls, and progressive disclosure of peer mechanics.

## Reference contribution template

Add one entry below this heading for each user-supplied screenshot, link, sketch, or recording. Store files in this directory only when the contributor has the right to share them; otherwise record the original URL. Do not copy third-party assets into the repository merely for convenience.

```markdown
### Short reference name

- Source/owner: [name and original URL]
- Added by/date: [person, YYYY-MM-DD]
- Screen or flow: [for example, narrow Following feed empty state]
- Problem it helps solve: [one sentence]
- Borrow: [specific behavior, hierarchy, spacing, or interaction]
- Avoid: [specific behavior that conflicts with this product]
- Accessibility notes: [keyboard, screen reader, contrast, scaling, motion]
- Rights/storage: [linked only, original work, or permission details]
- Status: proposed | accepted | rejected
```

## Open questions (fog)

1. Does the first interactive prototype remain framework-free, or does measured renderer state complexity justify React and selective shadcn/ui adoption?
2. Which exact narrow/wide navigation pattern performs best in keyboard, Narrator, and usability tests?
3. What visual identity—typeface, color family, icon set, density, and tone—best supports a calm social experience without mimicking an incumbent network?
4. Should offline/peer availability appear per post, per profile, or only when retrieval fails?
5. Which automated accessibility stack fits the eventual renderer test setup, and what minimum supported Windows/Narrator versions define release testing?
