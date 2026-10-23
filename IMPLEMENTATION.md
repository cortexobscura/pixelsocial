# PixelSocial implementation evidence and open issues

Baseline: `e15d1ece2619595dea3a3c68f671f688dbadbbd3`.
Companions: [SPEC.md](SPEC.md) and [ARCHITECTURE.md](ARCHITECTURE.md).

This is a reverse-blueprint investigation, not a completed security or UI audit.
Production source, configuration, dependencies, and workflows were not changed.
Findings below are source observations unless a separate execution check is named.
They are not instructions to preserve suspected bugs.

## Evidence collected

The inspection covered entrypoints, navigation, composers and actions, all Manager
message branches, schema, identity, media utilities, localization, dependency
versions, packaging, CI, and existing contributor guidance. No application test
files, test script, fixture suite, or deterministic seeded demo were found.

| Check                       | Result and limits                                                                                                                                                                                                                                                                                  |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Initial Git status          | Clean at the baseline commit.                                                                                                                                                                                                                                                                      |
| `pnpm check`                | Passed TypeScript and Prettier before and after documentation changes. Does not test behavior.                                                                                                                                                                                                     |
| `pnpm build`                | Passed before and after documentation changes; produced `dist-xdc/app.xdc`. Vite warned that the classic `webxdc.js` script cannot be bundled; the host supplies it.                                                                                                                               |
| `unzip -l dist-xdc/app.xdc` | Confirmed HTML, manifest, icon, bundled JS/CSS, and font files. The development WebXDC mock was absent.                                                                                                                                                                                            |
| Generated manifest          | `version = "0.0.0"`; this checkout has no tags available to `git describe --tags`, so the configured fallback ran.                                                                                                                                                                                 |
| Dexie ordering probe        | Executed the installed Dexie 4.0.11 `Collection.sortBy` implementation with reverse direction and 501 synthetic records. Applying `.slice(-500)` returned dates 500 through 1 and omitted newest date 501. No database was opened; this isolates sorting, not IndexedDB traversal or UI rendering. |
| Lock/config diff            | No changes to `package.json`, `pnpm-lock.yaml`, or `pnpm-workspace.yaml`.                                                                                                                                                                                                                          |

The checks ran with Node 22.23.2 and pnpm 11.3.0. The first pnpm invocation also
installed missing dependencies from the lockfile and ran the workspace-allowed
core-js/esbuild install scripts. These are local verification versions, not a
new supported-platform promise.

Not exercised: a real WebXDC host, the external bot, multi-peer synchronization,
IndexedDB crash recovery, browser interactions, image conversion/sharing, debug
packaging, accessibility, or visual correctness. No UI or stylesheet was changed,
and no screen is being handed off as UX-verified.

## Material findings

### I-01: Command authority is unresolved

**OBSERVED / UNCERTAIN; security contract.** `BasePostItem` renders deletion
controls for every author. `Manager.processUpdate` applies delete commands without
checking ownership or administrative authority. `botMode` replaces local identity
and admin state without sender validation. Incoming post/reply identity and badge
fields are also trusted.

The installed `@webxdc/types` 2.1.2 receive envelope has no authenticated sender
field. The mock's private `_sender` field is not a production contract. This code
alone cannot establish author-only or admin-only permissions.

**User decision:** document this behavior and leave intended permissions unresolved
until host/bot evidence is available. Do not describe permissive behavior as either
an approved trusted-group policy or a confirmed violation of a nonexistent policy.
Needed evidence: the intended deployment model, bot command origin, and supported
host identity guarantees.

### I-02: A later checkpoint can skip a failed update

**SUSPICIOUS; data consistency.** The worker removes an update before processing
it, catches errors, logs them, and continues. `processUpdate` saves `maxSerial`
when a later update reaches the host's maximum, without verifying that all prior
serials succeeded. A missing-parent reply/like or storage error can therefore be
followed by a checkpoint beyond the failure. Restart then requests only later
updates. There is no retained failure record or retry path.

Evidence: `Manager.init` worker and `processUpdate` serial handling. The local
storage checkpoint and IndexedDB writes are also not one transaction. Recovery
policy must be decided and tested; this is not an exactly-once mechanism.

### I-03: Expiration removes parents but leaves child state

**OBSERVED / SUSPICIOUS; retention and recovery.** Startup deletes only posts with
`active` below local midnight minus 360 calendar days. Replies and likes survive.
Subsequent updates that require those parents dereference a missing post and can
fail. The global comments view can contain orphan comments.

Explicit `deleteP` does cascade to replies/likes, showing an inconsistent lifecycle
between expiry and user deletion. Expiry runs after listener registration but
before the queued initialization item; replay may insert old posts after cleanup.
It is local pruning, not host-history removal, replicated deletion, or verified
erasure. Evidence: `Manager.init`, `processUpdate`, `storage.ts`.

### I-04: Long threads keep the oldest 500 replies

**OBSERVED / SUSPICIOUS; reading history.** `getReplies` performs
`where({ postId }).reverse().sortBy("date")`, then `.slice(-500)`. Installed Dexie
sorts descending for reverse direction; keeping the tail omits the newest replies
once there are more than 500. By contrast, `getAllReplies` keeps the newest 500.
The dependency-level execution probe confirmed the sort-and-slice result.

Separately, App resolves a thread parent only from the 500 loaded posts. Opening
a comment whose parent is outside that window returns home even if the database
still holds the parent. Evidence: `Manager.getReplies`, `getAllReplies`,
`App.tsx`, `ReplyItem.tsx`. Intended thread-window behavior remains **UNCERTAIN**.

### I-05: Activity after deletion is not based on newest reply date

**SUSPICIOUS; feed ordering and expiry.** `deleteR` and `deleteAll` use
`db.replies.where({ postId }).reverse().first()` to choose a remaining reply.
This query traverses the `postId` index rather than sorting by `date`; records with
the same postId are ordered by primary key. UUID order does not establish date
order. The parent activity can move earlier than its newest remaining reply,
affecting feed position and later expiration. No runtime test was performed.

### I-06: Media input checks do not enforce host or output limits

**OBSERVED / SUSPICIOUS; interoperability and resource use.** `Draft` checks video
input bytes against 1 MiB and resizes selected images, but never checks the final
serialized update against `webxdc.sendUpdateMaxSize`. Base64 and text/envelope
fields add bytes. Incoming media has no corresponding size or shape validation.
The Manager also does not observe the host's sending interval.

The bundled `Pixelit.resizeImage` calculates width and height ratios separately,
with height overriding width. For a 1000-by-600 image and 500-by-500 limits, its
arithmetic chooses a height ratio and leaves width above 500. Thus "configured
maximum dimensions" must not be documented as an enforced two-axis bound.
Evidence: `Draft`, `Manager.maxSize`, `pixelit.js`, installed WebXDC types.
Actual canvas/codec behavior and supported host limits remain untested.

### I-07: Image errors can remain pending or escape the composer

**SUSPICIOUS; failure feedback.** `loadImage` resolves only on `onload`; it has no
`onerror` rejection. Failed image decoding can leave conversion waiting forever.
The file-picker callback does not await/catch the asynchronous processing callback.
Pixelation also lacks user-visible error handling. Paste catches errors but logs
them. Evidence: `lib/util.ts`, `FilePicker.tsx`, `Draft.tsx`.

### I-08: Whole-record replay can overwrite local derived state

**SUSPICIOUS; replay and identity.** A received `post` is put wholesale, including
counters and optional `liked`. Reprocessing an older post can replace counters or
self-like state already derived from later events. Complete ordered replay might
rebuild some values, but interrupted/partial replay is not proven safe.

Changing identity through `botMode` does not recompute existing `liked` flags.
A saved nonzero serial with missing IndexedDB data has no reset/rebuild detection.
Evidence: `Manager.processUpdate`, constructor/init, `storage.ts`.
Do not generalize compound-key uniqueness into whole-application idempotency.

### I-09: Rendering has fragile hook and memoization patterns

**SUSPICIOUS; UI freshness.** Feed and RepliesList call `useMemo` inside variable
length item maps. App and BasePostItem have conditional hook execution. Feed's
memo dependencies omit text, media, author name, style, and activity; RepliesList's
dependencies omit changing reply fields. A replacement record with the same ID
and unchanged dependencies can reuse stale content. The composer adds an input
event listener on every effect run without cleanup.

Evidence: `App.tsx`, `Feed.tsx`, `RepliesList.tsx`, `BasePostItem.tsx`, `Draft.tsx`.
These are source-level risks, not browser-reproduced failures in this review.
The current compiler/formatter checks do not include a hook-rule linter.

### I-10: Several keyboard and focus paths need verification

**OBSERVED / SUSPICIOUS; accessibility.** Tab controls and "Show More" are clickable
non-button elements; style choices are clickable divs marked `aria-hidden`.
The image-opening wrapper is not a keyboard control. The compact reply field
refocuses itself on blur. Share's click handler is on the icon rather than its
surrounding button. `index.html` disables user scaling in its viewport settings.

Evidence: `TabNavigation`, `PostItem`, `ReplyItem`, `StylesReel`, `PostImage`,
`ReplyDraft`, `PostActionsBar`, `index.html`. These warrant keyboard, screen-reader,
and host testing. This review makes no contrast or visual-quality claim.

### I-11: Contributor release guidance disagrees with version generation

**OBSERVED; documentation drift.** `.github/copilot-instructions.md` tells
contributors to update the public manifest's version before release. The actual
manifest contains `{{APP_VERSION}}`, and Vite replaces it from Git's tag description
in generated output. Replacing the source placeholder would bypass that mechanism.

CI checks out without an explicit full-history/tag-fetch configuration, installs
an unpinned pnpm, and has no host test. Exact release-version behavior should be
verified in a tag build; the local untagged build only proved the fallback.
The existing contributor file was not edited during reverse blueprinting.

## Ambiguity map

| Dimension                                 | State                    | What remains                                                                                  |
| ----------------------------------------- | ------------------------ | --------------------------------------------------------------------------------------------- |
| Runtime, build entrypoints, schema, pages | RESOLVED as observations | Current mechanisms are mapped, not newly selected.                                            |
| Product audience                          | ASSUMED / INFERRED       | Small group-chat use is supported by README and structure; no measured scale target.          |
| Permissions and bot identity              | OPEN                     | User explicitly retained uncertainty; inspect host/bot contract before choosing enforcement.  |
| Expiry and deletion scope                 | OPEN                     | Local pruning versus shared lifecycle, child cleanup, and history retention intent.           |
| Failure, replay, and recovery             | OPEN                     | Missing-parent outcome, failed-update handling, partial storage loss, and restore guarantees. |
| Wire and host compatibility               | OPEN / EXTERNAL          | Supported host versions, bot fixtures, resource limits, and change compatibility.             |
| Thread window                             | OPEN                     | Oldest versus newest replies and parent loading outside the feed window.                      |
| Test/fixture strategy                     | OPEN                     | Existing checks are known; behavior coverage and reproducible host fixtures are absent.       |
| Application server/TLS/HTTP API           | N/A to current app       | Those are not deployable surfaces in this repository.                                         |

## Follow-up evidence, not an implementation plan

Resolve the trust model using the supported host and external bot first. Then
specify lifecycle and failure outcomes with synthetic update sequences: missing
parents, failed writes followed by later serials, duplicate/partial replay,
identity changes, expired parents, and deletion with nonchronological reply IDs.
Do not select an authentication scheme, retry policy, or migration silently.

Use 501 timestamped replies to expose the thread-window issue. Add media cases for
landscape resizing, invalid images, maximum-size videos, base64 expansion, and
host limits. Exercise normal two-peer flows and reopening in an actual supported
host. The development mock alone cannot prove encryption, sender authority, or
recovery from host history.

After those decisions, update the candidate contract in SPEC.md, retain deviations
here until fixed and verified, and only then generate a stable AGENTS.md handoff.
No new glossary or ADR was needed: the user deferred a policy rather than choosing
an architecture or redefining a domain term.
