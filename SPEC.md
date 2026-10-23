# PixelSocial reconstructed specification

Status: **NEEDS DECISIONS**

Baseline: commit `e15d1ece2619595dea3a3c68f671f688dbadbbd3`.
This is a reverse blueprint of the existing app, not approval to change its behavior.

## How to use this document

Read this document with [ARCHITECTURE.md](ARCHITECTURE.md) for current wiring and
[IMPLEMENTATION.md](IMPLEMENTATION.md) for evidence, limitations, and suspicious
behavior. Code observations do not automatically become requirements.

Evidence labels used throughout the blueprint:

- **OBSERVED**: established from source, configuration, dependencies, or an identified check.
- **INFERRED**: likely product intent, not independently confirmed.
- **UNCERTAIN**: intended behavior needs a decision.
- **SUSPICIOUS**: existing behavior may be accidental or unsafe.
- **EXTERNAL**: depends on the chat host or another project.

During reverse review, the user chose to document participant permissions as
unresolved until the host/bot contract is checked. Neither unrestricted deletion
nor author/admin-only deletion is an approved security requirement.

## Purpose and users

**INFERRED**, supported by `README.md` and the implemented workflows: provide a
small, independent micro-blogging space inside a group chat, with pixel-art
presentation and optional image pixelation. Intended users are friends, families,
and other small chat groups.

**OBSERVED**: the app has no registration, login screen, backend service, or direct
RSS/ActivityPub client. It uses the host's WebXDC identity and shared updates.

**EXTERNAL**: Delta Chat encryption, message delivery, group membership, and bot
imports are host/integration responsibilities. This repository does not implement
or verify those guarantees. "No server" means no PixelSocial application server;
it does not mean the messaging host operates without network infrastructure.

**UNCERTAIN**: supported group size, total history size, minimum host versions,
performance targets, storage quotas, and recovery targets are unspecified. A
500-item view limit is not a tested capacity guarantee.

## Product scope

The existing product supports a posts feed, an all-comments view, post threads,
text and media composition, likes, deletion commands, sharing to a chat, and a
display-name setting. It includes English source strings and Spanish translations.

**OBSERVED scope exclusions**: there is no public web deployment contract,
application HTTP API, account directory, search, follow graph, nested reply tree,
post-editing UI, application backup/restore command, or built-in RSS importer.
These are absent capabilities, not permanent prohibitions on future work.

## Reconstructed workflows

These are **OBSERVED** current behaviors. Their presence is evidence for the
candidate product contract, not proof of correctness under every failure mode.

| Workflow           | Current behavior                                                                                                                                     | Evidence                                             |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Open the app       | Load local state and host updates, then show the posts feed. No explicit loading or startup-error screen.                                            | `src/App.tsx`, `Manager.init`                        |
| Read posts         | Show at most 500 posts, ordered by descending `active`. Replies can move a post up; likes do not.                                                    | `Manager.init`, `processUpdate`                      |
| Read comments      | Show up to 500 newest comments globally; open a comment's parent thread and attempt to focus it.                                                     | `Home`, `ReplyItem`, `PostComments`, `getAllReplies` |
| Compose            | Trim text; reject an empty submission unless it has an attachment. Send a post and return home without waiting for peer delivery.                    | `Draft`, `Manager.sendPost`                          |
| Choose a text card | Offer styles 1–100. More than 250 JavaScript string code units or 10 newline-separated lines disables styled text, not posting.                      | `constants.ts`, `Draft`, `StylesReel`                |
| Attach media       | Select one image or video; paste the first clipboard image. Media selection resets the card style; card selection removes media.                     | `Draft`, `FilePicker`                                |
| Pixelate an image  | Convert the selected image using Pixel It and send the result as PNG.                                                                                | `Draft.pixelate`, `pixelit.js`                       |
| Reply              | Reply to a post with a compact text field or expanded text/media composer. Enter sends compact replies; Ctrl+Enter sends from the expanded composer. | `ReplyDraft`, `Draft`                                |
| Like or unlike     | Broadcast an action for the current identity and update local counts when processed.                                                                 | `PostActionsBar`, `Manager`                          |
| Delete             | Offer single-item deletion and deletion of all content by an author on every item. Intended permissions remain unresolved.                           | `BasePostItem`, `Manager`                            |
| Share              | Send plain post text and optional media through the host, or render a styled text card as JPEG for sharing.                                          | `PostItem.onShare`                                   |
| Change name        | Send a trimmed name update. Apply it to the matching local identity for future content; existing author-name snapshots remain unchanged.             | `Settings`, `Manager.setSelfName`, `processUpdate`   |

Plain text is shortened for display after 500 code units or 15 lines, with a
"Show More" action. This is not an input limit. Posts and replies render text as
JSX text rather than interpreting it as HTML (`PostItem`, `ReplyItem`).

### Media limits

**OBSERVED**: `Manager.maxSize` is 1,048,576 bytes. Videos exceeding this input size
are rejected. PNG/JPEG images are always converted to JPEG; other image formats
at or below that input size retain their original data URL. Larger images use the
resize path. Resizing and pixelation configure a maximum width and height of 500.

**SUSPICIOUS**: this is not a maximum serialized-update size. Base64 increases
size, converted output is not checked again, and `webxdc.sendUpdateMaxSize` is not
consulted. The contract must not promise that every accepted attachment can be
sent by every host. See implementation finding I-06.

## Technology and interfaces

**OBSERVED**: TypeScript and Preact, with React-compatible imports, run in a browser
or host WebView. Vite packages the application as `dist-xdc/app.xdc`. Dexie wraps
IndexedDB. The lockfile fixes the inspected dependency versions; package manifest
ranges alone do not. See [ARCHITECTURE.md](ARCHITECTURE.md).

The externally relevant interface is the JSON payload union in `src/types.d.ts`:
posts, replies, likes/unlikes, deletion, name changes, and `botMode`. The exact
message shapes and processing order are documented in the architecture file.
There is no payload-version field or negotiated compatibility window.

**EXTERNAL**: bot interoperability is referenced in the README but is not
implemented or tested here. Do not change payload fields, identity handling, or
bot behavior based solely on the local UI.

## Identity and security boundary

**OBSERVED**: the default identity comes from `webxdc.selfAddr` and
`webxdc.selfName`, with local overrides. An incoming `botMode` payload can replace
local identity and the admin flag. The flag is copied onto new content and shown
as a badge. It is not an authorization check.

**OBSERVED**: received payloads have TypeScript declarations but no runtime schema,
sender-ownership, or role validation. The installed WebXDC type definition exposes
serials and payloads, not an authenticated sender field. Host encryption does not
establish who is authorized to issue an individual application command.

**UNCERTAIN**: the intended participant trust model, trusted origin of `botMode`,
and deletion authority. Further evidence is required before specifying enforcement.

## State, lifecycle, and failure semantics

**OBSERVED**: local IndexedDB stores posts, replies, and likes. `localStorage`
stores identity overrides and the last saved host serial; `sessionStorage` stores
a feed scroll position. Drafts and current navigation are not durable.

The host update stream is the source of shared changes; IndexedDB is the local
state used to render them. Full reconstruction from host history is **EXTERNAL**
and unverified, not an application backup guarantee.

**OBSERVED**: updates enter an in-memory queue, are processed sequentially, and
trigger throttled post reads. A failed update is logged and removed from the
queue. There is no application retry queue or failed-update UI. The saved serial
advances only when a processed update has `serial === max_serial`, outside the
database transaction. This does not establish exactly-once processing or a
contiguous successfully applied history.

**OBSERVED**: startup removes posts whose `active` precedes local midnight minus
360 calendar days. This operation does not remove their replies or likes and does
not remove host history. Explicit post deletion does remove associated replies
and likes in a transaction. Deleting an author's posts also deletes other users'
replies attached to those posts.

**UNCERTAIN**: intended expiry scope, whether expiration should be replicated,
missing-parent policy, replay/retry requirements, storage-loss recovery, and
whether deletion means only removal from the current projection or something
stronger. No archival, verified erasure, or restore guarantee is established.

## Core invariants and limits on claims

These are **OBSERVED** structural properties to preserve while documenting this
baseline, not newly approved architecture decisions:

1. The production artifact is a client-side WebXDC package; app code does not
   require a separate PixelSocial backend or embed chat credentials.
2. Shared changes use `webxdc.sendUpdate`; UI submission does not directly insert
   posts/replies into the local database. The receive path performs those writes.
3. Likes have a compound primary key of `(postId, userId)`, preventing duplicate
   rows for that pair. Cached post counters can still become inconsistent.
4. Explicit post deletion uses one database transaction for its post, replies,
   and likes. The startup-expiration path does not share this guarantee.
5. User text is rendered as text, not raw HTML. This does not constitute complete
   validation of media, identifiers, timestamps, or command authority.

Do not claim trusted authorship, enforced admin rights, reliable replay, secure
erasure, bounded total storage, or verified backup recovery until mechanisms and
tests establish them.

## Build and verification contract

Use the existing commands; this blueprint does not introduce a new test framework.

| Command                     | Existing purpose                                                         |
| --------------------------- | ------------------------------------------------------------------------ |
| `pnpm i`                    | Install dependencies; the repository does not pin pnpm or Node versions. |
| `pnpm start`                | Start the Vite development server on port 3000 with the WebXDC mock.     |
| `pnpm emulator`             | Start Vite and the WebXDC development runner.                            |
| `pnpm check`                | Run TypeScript and Prettier checks; not behavioral tests.                |
| `pnpm build`                | Type-check, build, and package the `.xdc` archive.                       |
| `NODE_ENV=debug pnpm build` | Build with embedded debugging tools.                                     |
| `pnpm fix`                  | Reformat files; do not run over unrelated changes.                       |

CI runs installation, `pnpm check`, and `pnpm build` on pull requests and pushes
for `main`/`master`, and on matching version tags. Tag builds publish the `.xdc`
as a GitHub release. There is no automated application test suite, seeded demo,
security scan, or host interoperability check wired into CI.

## Acceptance evidence still needed

The following are validation scenarios, **not existing passing tests**. Use
synthetic participants and disposable app storage, not real chat history.

1. Two host instances exchange a text post, an attachment, a reply, and like/unlike
   actions; both converge after closing and reopening.
2. Card selection, size/line thresholds, image paste, conversion, pixelation,
   video rejection, and plain/styled sharing work in supported hosts.
3. Global comments and thread views remain usable at 500 and 501 items, including
   a parent outside the loaded post window. Decide the intended thread window first.
4. Name changes affect future content without silently rewriting old snapshots;
   bot fixtures establish identity and administrative semantics.
5. Approved deletion rules are exercised for own/other content and bulk deletion;
   missing-parent, expiry, duplicate, malformed, and reordered updates have explicit outcomes.
6. A process interruption or storage failure cannot silently skip required updates
   under the chosen recovery contract; restore behavior is verified separately.
7. Keyboard navigation, focus, modal dismissal, localization, and small-screen
   media handling are tested in a browser and the selected host versions.

## Readiness and completion

The reverse blueprint is complete when current surfaces, data ownership, limits,
external dependencies, evidence, and uncertainties are documented without code
changes or invented guarantees. Actual check results belong in
[IMPLEMENTATION.md](IMPLEMENTATION.md).

This specification remains **NEEDS DECISIONS**, not implementation-ready or
one-shot-ready. Before using it for a behavior-changing handoff, resolve the
permissions and bot contract, retention/recovery semantics, wire compatibility,
and supported host/resource limits. Then define failure-path fixtures and a
verifiable Definition of Done. `AGENTS.md` generation is deferred until that
source-of-truth contract is stable.
