# Sync and Collaboration: An Optional Layer, Not the Foundation

The most common way local-first projects go wrong is building sync first. Sync is the hardest part of the whole field, and it is optional. This file covers the right ordering, the conflict problem in plain words, what CRDTs actually are, and the simpler alternatives that are often all you need.

## The ordering rule

Server-first apps get sync for free because there is only one copy of the data and it lives on the server. The price was everything in [02-why-it-matters.md](02-why-it-matters.md).

Local-first flips the ordering: the app must be complete without sync. Data on device, all features offline, export always available. Then, if the product needs it, sync is added as a layer that moves data between devices, without ever becoming the authority over it.

The test for whether you built the layer or the foundation: turn the sync service off forever. If the app still works fully on each device and only cross-device convenience is lost, sync was a layer. If anything else breaks, you built a cloud app with extra steps.

Most tools should ship v1 with no sync at all. Our portfolio is full of tools that never grew sync and never needed it, because the data is single-user and the export path (see [03-architecture-patterns.md](03-architecture-patterns.md)) covers the occasional device move.

Two cheap things to do in v1 even when shipping no sync, because they keep the door open:

- Give every record a stable unique id and an updated timestamp. Merging is impossible later without identity and ordering, and retrofitting them across installed copies is miserable.
- Never mutate records in ways that destroy history you might need (overwrite-in-place is fine for data; just avoid reusing ids).

That is the entire tax. Everything else about sync can wait until the product proves it needs it.

## The conflict problem, in plain words

The moment the same data can be edited in two places without a network between them, this happens:

1. Your laptop and your phone both have the note "Call the bank."
2. On a plane, you edit it on the laptop: "Call the bank about the wire."
3. Meanwhile the phone edits it too: "Call the bank Tuesday."
4. Both devices come online. There are now two true versions of one note, and no third party was watching to say which came first.

Every sync design is an answer to "what happens at step 4." The menu of answers, worst to best:

- **Last write wins.** Compare timestamps, keep the newer edit, silently discard the other. Simple, and it quietly destroys user data. In the example above, either "about the wire" or "Tuesday" vanishes without a trace, and the user finds out weeks later, if ever. Acceptable only for data where losing an edit is genuinely harmless (a settings toggle). For documents it is a bug you shipped on purpose.
- **Detect and surface.** Notice the conflict, keep both versions, present them to the user to resolve. Honest and sometimes right, but conflict dialogs about their own notes make users distrust the app, and frequent dialogs mean the design has failed upstream.
- **Merge automatically, correctly.** Structure the data so both edits can be kept without a referee. This is what CRDTs do.

Notice that the problem is not the network being slow or flaky; those are engineering annoyances. The problem is concurrency without coordination, and it is fundamental: no amount of clever transport removes step 4. You either prevent concurrent writes (the single-writer designs below), accept losses, surface conflicts, or adopt data structures that merge.

## CRDTs, explained simply

CRDT stands for Conflict-free Replicated Data Type. Strip the jargon and it is this: a data structure designed so that two divergent copies can always be merged, automatically, and every device arrives at the same result, no matter what order the changes are applied in and without any central referee deciding who wins.

The trick is that a CRDT does not store just the current state. It stores enough information about the history and identity of each change (who made it, relative to what) that merging becomes deterministic math instead of a judgment call. A CRDT text document does not hold "the string"; it holds every character with a stable identity, so "insert 'about the wire' after character X" and "insert 'Tuesday' after character Y" are both applied cleanly, on every device, with no coordination. The bank-note example from above merges to a note containing both edits, identically on both devices.

What you pay for this:

- **Metadata overhead.** CRDT documents carry history and identity, so they are larger than the plain data, sometimes much larger for long-lived documents with heavy edit histories.
- **A real learning curve.** Modeling your app's data as CRDT types is a design exercise, not a drop-in.
- **Consistent but occasionally surprising merges.** The merge is always the same everywhere, but it is not always what a human editor would have chosen. Consistent-but-surprising beats a conflict dialog for most products; know the deal you are making.

The practical takeaway: **do not invent your own CRDT.** The academic literature is deep, the edge cases are brutal, and two mature open-source implementations already exist:

- **Yjs** - the pragmatic, performance-focused option with a large ecosystem of editor bindings and network adapters. The common default for collaborative text and rich documents.
- **Automerge** - grew directly out of the Ink and Switch research line behind the local-first essay (see [01-what-local-first-means.md](01-what-local-first-means.md)); JSON-shaped documents with a full change history.

Both are serious, maintained projects. If your product genuinely needs real-time multi-writer collaboration, pick one of these and build on it. That single decision replaces a research project with an integration project.

A note on transport: CRDTs solve merging, not moving. You still need something to carry changes between devices (a small relay server, a peer-to-peer channel, or a file-sync folder as below). That transport can be dumb, which is the point: with CRDTs even the sync server is just a pipe with no authority over the data. It can also be end-to-end encrypted for the same reason; the pipe never needs to understand what it carries.

## Simpler alternatives that are often enough

Before reaching for CRDTs, walk this list from the top. Each step down adds engineering cost; stop at the first one that fits.

### 1. No sync

Single-device tools with clean export are a legitimate finished product. A tax calculator, a metadata scrubber, a one-shot generator: the data has no life across devices. Shipping without sync is not a gap if the product does not need it.

### 2. Export/import as manual sync

Ship a complete-state export (open format, versioned, as in [03-architecture-patterns.md](03-architecture-patterns.md)) and a merge-capable import. The user moves the file. Crude, transparent, zero infrastructure, and it doubles as backup and as the vendor-death escape hatch. For tools used on a second device once a month, this is genuinely the right amount of engineering.

### 3. The user's own file sync

If data lives in plain files or a single SQLite file, the user can point any file-sync service they already run at it: a commercial cloud drive folder, a self-hosted sync tool, a network share. You write zero sync code and the user keeps custody end to end.

Two rules make this safe instead of flaky:

- **Design for whole-file replacement.** One file per document syncs far better than one big mutable blob for everything, because a conflict then costs one document, not the world. File-sync services handle concurrent edits by duplicating the file ("conflicted copy"), which is the detect-and-surface strategy at per-file granularity. Crude, visible, survivable.
- **Never assume exclusive access.** Write atomically (write to a temp file, rename over the target), tolerate the file changing underneath you, and reload gracefully. SQLite over file sync needs care: close cleanly, and avoid write-ahead-log mode for synced files since the sidecar files confuse syncers.

This tier covers the large middle of real products: single user, multiple devices, edits rarely simultaneous.

### 4. Single-writer sync

Multi-device but only one device edits at a time (or one device is designated the writer, others read). Conflicts are impossible by construction, so a dumb copy mechanism suffices. Fits capture-on-phone, process-on-desktop workflows and dashboard-style companions.

### 5. CRDTs

For the real thing: multiple writers, concurrent edits, merging must be automatic, possibly live cursors in the same document. Yjs or Automerge, plus a transport. Enter this tier knowingly, because it dominates the engineering budget of any app it touches.

### The tiers at a glance

| Tier | Conflicts handled by | Infrastructure you run | Engineering cost |
|---|---|---|---|
| No sync | Cannot occur | None | None |
| Export/import | The user, explicitly | None | Days |
| User's own file sync | Conflicted-copy duplication | None | Days to weeks |
| Single-writer | Prevented by construction | Minimal or none | Weeks |
| CRDTs | Automatic merge | A dumb relay, at most | Months, ongoing |

## The honest summary

Sync is where local-first earns its "harder than cloud" reputation, which is exactly why it must stay optional. Ship the tool. Add the lightest sync tier the product actually needs. Treat CRDTs as the specialist tool for the collaboration tier, not the default. The costs you are choosing between are itemized in [06-honest-tradeoffs.md](06-honest-tradeoffs.md).
