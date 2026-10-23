# PixelSocial current architecture

Baseline: `e15d1ece2619595dea3a3c68f671f688dbadbbd3`.
This describes the implementation, not a proposed redesign. Evidence labels and
product scope are defined in [SPEC.md](SPEC.md); risks and check results are in
[IMPLEMENTATION.md](IMPLEMENTATION.md).

## Runtime boundary

**OBSERVED**: one client-side application runs inside a WebXDC-capable host. The
host supplies `webxdc.js`; development Vite middleware supplies a mock instead.
The production archive contains HTML, bundled JavaScript/CSS, fonts, an icon,
and the manifest. It contains no PixelSocial server or database service.

```text
Chat host / WebXDC runtime
  supplies self identity, update delivery, chat sharing
                    |
               index.html
                    |
             src/main.tsx
                    |
                 App.tsx
                    |
        ManagerContext + PageContext
           |                    |
        Manager             Pages/components
           |                    |
        Dexie              local composition,
           |               media conversion,
       IndexedDB           sendToChat sharing

Shared changes:
UI -> Manager send methods -> webxdc.sendUpdate
host listener -> in-memory queue -> processUpdate -> IndexedDB
IndexedDB -> throttled posts callback -> App -> pages
```

**EXTERNAL**: host transport, encryption, history retention, storage isolation,
and delivery to other devices. Local browser mocks are not evidence that these
properties hold in a messaging host.

## Source map

| Boundary             | Responsibility and evidence                                                                                                                             |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Entry and startup    | `index.html`, `src/main.tsx`, `src/App.tsx`: mount Preact, construct one Manager, start initialization, retain posts and navigation in component state. |
| Shared context       | `src/contexts.ts`: expose Manager and current page/setPage to components.                                                                               |
| Application behavior | `src/lib/manager.ts`: identity, outbound updates, sequential receive processing, database mutations, post/reply queries.                                |
| Persistence          | `src/lib/storage.ts`: Dexie database and schema version 1.                                                                                              |
| Domain/message types | `src/types.d.ts`: global Post, Reply, Like, Payload, and PageData types; no runtime validator.                                                          |
| Pages                | `src/pages/`: Home, NewPost, PostComments, Settings.                                                                                                    |
| Presentation         | `src/components/`: feeds, items, actions, composition, navigation, image viewer, and native video controls.                                             |
| Media                | `Draft.tsx`, `lib/util.ts`, `lib/pixelit.js`: FileReader, image loading, canvas resize/pixelation; `PostItem.tsx` uses html-to-image for card sharing.  |
| Identity artwork     | `MonsterAvatar.tsx`, `lib/monsterid/`: deterministic avatar rendering from the user ID; not an authentication mechanism.                                |
| Localization         | `lib/i18n.ts`, `lib/langs/es.json`: browser-language selection; Spanish lookup with English-key fallback.                                               |
| Styling/assets       | `App.css`, component CSS, inline styles, `constants.ts`, `fonts/`, and bundled Jersey 10 fonts. No separate DESIGN.md exists at this baseline.          |
| Packaging            | `vite.config.ts`, `public/manifest.toml`, `.github/workflows/ci.yml`: build, version substitution, archive, and tag releases.                           |

`lib/pixelit.js` and `lib/monsterid/` contain third-party-derived JavaScript kept
in the source tree. `dist/`, `dist-xdc/`, and `node_modules/` are generated/ignored;
`pnpm-lock.yaml` is tracked. The repository license is GNU AGPL version 3.

## Dependency and build boundary

**OBSERVED**, from the lockfile inspected during reconstruction:

| Tool/library              | Locked version |
| ------------------------- | -------------- |
| Preact                    | 10.26.6        |
| TypeScript                | 5.8.3          |
| Vite                      | 6.3.5          |
| Dexie                     | 4.0.11         |
| html-to-image             | 1.11.13        |
| Lodash                    | 4.17.21        |
| WebXDC types              | 2.1.2          |
| WebXDC Vite plugins       | 1.5.1          |
| WebXDC development runner | 0.19.0         |
| Prettier                  | 3.5.3          |

The manifest uses ranges and several `latest` specifiers. Neither Node nor pnpm
has a repository version pin. `tsconfig.json` targets ES2017 with DOM types,
strict checks, unused-symbol checks, no emit, and a `~/*` alias to `src/*`.
It includes `src`, not `vite.config.ts`. Preact's Vite preset supplies compatibility
for the source's `react` imports.

Vite invokes Preact, manifest-version replacement, XDC packaging, debug tooling,
the development WebXDC mock, and icon generation. Version replacement uses
`git describe --tags`, strips a leading `v`, and falls back to `0.0.0` if no
matching description can be obtained. It changes the generated manifest, not
`public/manifest.toml`. The public manifest retains `{{APP_VERSION}}`.

The installed debug plugin embeds Eruda for a non-production `NODE_ENV`;
`NODE_ENV=debug` is the README's documented invocation. Normal production output
relies on the host-provided `webxdc.js`; the development mock is not bundled into
the archive. No deployment manifests or application monitoring service exist.

## Navigation and UI state

**OBSERVED**: navigation uses a `PageData` union in memory, not URL routes:

- `home` with `showComments` selects the posts or all-comments tab.
- `newpost` opens the composer.
- `settings` changes the display name.
- `comments` carries `postId` and optionally `focusReplyId`.

A thread's parent is looked up only in App's loaded 500-post array. If it is not
found, navigation returns home, even if the parent still exists in IndexedDB.
Thread replies are read separately. A selected reply is scrolled into view after
loading when its element exists. Opening a thread from a post stores the feed
scroll offset in `sessionStorage`; returning to the feed consumes that offset.

Composition, selected media, style, image-viewer zoom/pan, and modal state are
component state. The composer does not persist unsent drafts. The image viewer
uses a native dialog, zoom steps of 0.25 within 0.5–4, and mouse/touch panning.

## State ownership

**OBSERVED** schema: Dexie database `appdb`, version 1. No later migration exists.

| Store     | Key and indexes                                        | Contents and ownership                                                                      |
| --------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `posts`   | Primary key `id`; indexes `active`, `authorId`         | Shared post fields plus cached `likes`, `replies`, and device-specific `liked`.             |
| `replies` | Primary key `id`; indexes `postId`, `authorId`, `date` | Comments attached directly to a post; no reply-to-reply relation.                           |
| `likes`   | Compound primary key `[postId+userId]`                 | One row per post/user pair. Dexie supports lookup by the compound index's leading `postId`. |

Post fields are `id`, `authorName`, `authorId`, `isAdmin`, `date`, `active`, `text`,
`file`, `filename`, `style`, `likes`, optional `liked`, and `replies`. Reply fields
omit activity/counters/liked and add `postId`. File contents are data URLs inside
the record, not separately stored attachments. Author names/admin flags are
snapshots carried by each content record, not joins against a user table.

Database key uniqueness is enforced; foreign-key integrity is not. Counters and
activity are recalculated by selected mutation paths, not by database constraints.
`posts.put` accepts a whole incoming post, including its counters.

| Other state                         | Location                                           | Restart behavior                                                                             |
| ----------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `selfId`, `selfName`, `isAdmin`     | Manager fields and optional localStorage overrides | Reconstructed from overrides, otherwise host identity; admin defaults false.                 |
| `maxSerial`                         | localStorage string                                | Parsed and passed to the host listener on startup. Not transactionally coupled to IndexedDB. |
| `feedScrollPos`                     | sessionStorage                                     | Consumed when Feed mounts; not shared with peers.                                            |
| Incoming queue and refresh callback | Manager memory                                     | Lost when the page/process closes; no stop/dispose API.                                      |
| Posts window/navigation/drafts      | Preact state                                       | Recreated on launch.                                                                         |

**INFERRED**: IndexedDB is intended as a replayable local projection of host
updates. **EXTERNAL/UNCERTAIN**: whether all required history survives and whether
replay restores a consistent state after partial storage loss. There is no app
repair, export/import, or backup routine.

## Shared update protocol

**OBSERVED**: sends use `webxdc.sendUpdate({ payload, ... }, "")`. New posts and
replies add an English `info` string naming the author and action. IDs use
`crypto.randomUUID()` with a Math.random-based fallback. Dates are client wall-clock
milliseconds, not host-assigned timestamps.

| Payload branch | Fields                          | Receive effect                                                                                                                                                    |
| -------------- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `post`         | Full Post                       | Put/replace the post; refresh posts.                                                                                                                              |
| `reply`        | Full Reply                      | Put reply, count replies, increase parent activity using reply date; refresh.                                                                                     |
| `like`         | `{ postId, userId }`            | Put compound-key row, recount, set `liked` for self; refresh.                                                                                                     |
| `unlike`       | `{ postId, userId }`            | Delete matching row, recount, clear `liked` for self; refresh.                                                                                                    |
| `deleteAll`    | Author ID string                | Delete that author's posts and their children/likes; delete that author's remaining replies; update affected parents. Likes by that author on other posts remain. |
| `deleteP`      | Post ID string                  | Transactionally delete post, its replies, and its likes; refresh.                                                                                                 |
| `deleteR`      | `{ postId, replyId }`           | Delete matching reply, recount and recalculate parent activity; refresh.                                                                                          |
| `setName`      | `{ userId, name }`              | If userId matches this Manager's identity, store the name and use it for future sends; empty name falls back to host name.                                        |
| `botMode`      | `{ selfId, selfName, isAdmin }` | Replace local identity/admin override; replace name only if truthy. No target or sender check.                                                                    |

The same TypeScript union includes `init: () => void`, used only for the locally
enqueued initialization callback. A function is not a JSON wire value; this is not
a supported external command. There is no runtime boundary separating that branch
from received payloads. The `if/else` chain follows the table order after `init`;
if malformed input carries multiple recognized branches, only the first runs.
Unknown objects reach serial handling without changing database state.

`setName` and `botMode` do not trigger the posts-change callback. Display-name
saving shows success immediately after sending, not after host delivery or a
completed local write.

The installed `ReceivedStatusUpdate` type supplies `serial` and `max_serial`
alongside the sending envelope. The development mock has an internal `_sender`,
but the application does not use it and the public type does not guarantee it.
It must not be treated as production sender authentication.

## Startup, concurrency, and recovery

**OBSERVED**, in `Manager.init`:

1. Start an asynchronous loop that drains the in-memory queue serially, catches
   each processing exception, logs it, and schedules the next drain after 100 ms.
2. Register the host listener from parsed `localStorage.maxSerial` or zero. The
   listener only appends to the queue; it does not await database application.
3. After listener registration resolves, delete posts older than local midnight
   minus 360 days by `active`. No child cleanup occurs here.
4. Enqueue an internal initialization item that installs a 500 ms-throttled
   callback. It reads the 500 highest-activity posts and publishes them to App.

Database-changing receive branches generally use transactions for related rows.
The single-post put and name/bot overrides are separate paths. Startup expiry can
interleave with queued history processing; expired content can be reintroduced
from history depending on timing and the checkpoint.

A processed update saves `maxSerial` only when its serial equals the host's
current maximum. A failed update has already left the queue. Later successful
updates can advance the checkpoint past that failure. Repeated keyed puts avoid
some duplicate rows but do not establish replay safety for cached counters,
activity, deletion, or changes of local identity.

The app does not inspect host `sendUpdateInterval` or `sendUpdateMaxSize`.
Send methods return without an application acknowledgment. Logging consists of
console exceptions, media-size diagnostics, and selected media errors; no metrics,
health endpoint, audit log, or persistent failure ledger is present.

## External integrations and trust boundaries

**OBSERVED**: application code uses local browser APIs and WebXDC rather than
direct fetch/XHR/WebSocket integration. Posting shares content through the host;
`sendToChat` exports text/media through a host-controlled chat selection flow.
Image transformation is local canvas work; styled sharing renders a DOM node.

**EXTERNAL**: bot/RSS integration, messaging encryption, participant admission,
message size enforcement, host database backups, and WebView codec support.
No bot implementation or compatibility fixture is in this repository.

**SUSPICIOUS**: identity fields, admin badges, destructive commands, attachment
contents, IDs, and timestamps are trusted as received. TypeScript types do not
validate incoming JSON. Permission enforcement, resource limits, and recovery
remain unresolved contracts; see [IMPLEMENTATION.md](IMPLEMENTATION.md).
