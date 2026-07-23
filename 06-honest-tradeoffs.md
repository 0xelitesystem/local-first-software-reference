# Honest Tradeoffs: What Local-First Costs You

Advocacy that hides the costs is marketing. We build local-first tools and we choose this architecture on purpose, which is exactly why we can afford to list what it gives up. If you are deciding whether to build local-first, this file is the one to argue with.

## Multi-device continuity is work

Cloud apps get "pick up on any device" for free, because there is only one copy and it lives on the server. Local-first inverts that: every device has a real copy, and continuity between them is now your engineering problem.

The full menu of answers, from no-sync through file-sync to CRDTs, is in [05-sync-and-collaboration.md](05-sync-and-collaboration.md). The tradeoff to internalize here is blunter: whatever tier you pick, you are building or integrating something a cloud app gets as a side effect of its architecture. For a solo developer or a small team, that cost is the single biggest line item in going local-first, and it is why the correct v1 answer is usually "no sync, clean export" rather than a half-built sync engine.

What users experience if you get this wrong is worse than a missing feature: two devices with silently diverged data. A cloud app is never wrong about what the current version is. A badly synced local-first app can be. Ship less sync, honestly labeled, over more sync that lies.

## Backup responsibility shifts to the user

When the vendor holds the data, the vendor does the backups. When the user holds the data, a lost laptop can mean lost data, and some fraction of users will learn that the hard way and blame the app.

You cannot remove this tradeoff, but you can shrink it:

- Store data in user-visible files and say so in the UI ("your data is this file, back it up like any file"). Backup tools the user already runs then cover it for free. This is Pattern 1 in [03-architecture-patterns.md](03-architecture-patterns.md) pulling double duty.
- Make export one click and prompt occasionally. An exported open-format file in the user's documents folder is a backup even if they never think of it as one.
- Auto-snapshot locally where the data warrants it (rotating copies in a sibling folder), so the app survives its own bugs and the user's accidental deletes.
- Treat the browser tier honestly: browser storage can be evicted or cleared with site data, so browser tools must push export-to-file harder than desktop tools do.

The honest framing for users: local-first trades "trust us with everything" for "hold your own keys." Most people who choose local-first tools understand that trade. State it anyway.

## Distribution: the web's instant deploy versus installed software

A cloud web app has the best distribution ever invented: send a link, user clicks, they are running the latest version. Everyone is always current; a fix deploys to every user in minutes.

Installed local-first software gives that up:

- **Friction at the front door.** Download, install, OS warnings about unsigned or newly signed binaries. Every step loses users a link would have kept.
- **Release engineering is real.** Per-OS builds, code signing certificates, installers, an update mechanism. This is a permanent tax on a small team, and skipping the update mechanism is worse (stranded users on buggy versions forever).
- **Old versions never die.** You will get bug reports from versions you shipped years ago, because nothing forces an upgrade. Data-format versioning (see the format rules in [03-architecture-patterns.md](03-architecture-patterns.md)) stops being hygiene and becomes survival.
- **Fixes travel slowly.** A security patch in a cloud app reaches everyone by lunch. Yours reaches whoever updates, whenever they update. Design accordingly: smaller privileged surface, conservative defaults, and an update check that is easy to say yes to but never blocks launch.

The single-file HTML pattern is the notable exception: it keeps web-grade distribution (send the file, or serve it from any static host) while staying fully local. That is a big part of why our browser portfolio uses it: every tool runs entirely in the browser, and updating means replacing one static file. The pattern only covers small tools, but within its range it dodges this tradeoff almost completely.

## Some features genuinely need servers

Local-first discourse sometimes slides into pretending servers are never needed. Not true, and pretending it costs credibility. Features that require someone to run infrastructure:

- **Cross-user anything.** Search across all users' public documents, a shared marketplace, a social graph, discovery feeds. If the feature is about data from people who are not the user, a server must aggregate it.
- **Realtime presence.** Who is online, live cursors, "three people are viewing this." Presence is ephemeral coordination state; a relay of some kind has to carry it, even in CRDT systems where the document itself needs no authority.
- **Always-on behavior.** Scheduled jobs that run while every user device is asleep, webhooks received on the user's behalf, sending email from an address users trust. A laptop lid closes; a server does not.
- **Server-side scale.** Indexing a corpus too large for a device, batch compute beyond local hardware, serving a model too big to run locally. The device ceiling is real, even as it rises every year.
- **The commercial edges.** Purchase fulfillment and license minting (the private-key side of the licensing pattern in [03-architecture-patterns.md](03-architecture-patterns.md)), update distribution, telemetry if you choose to have it. Small, but nonzero.

The local-first move is not to eliminate these servers; it is to keep them out of the critical path:

| Server role | In the critical path (avoid) | As an optional edge (fine) |
|---|---|---|
| Licensing | App refuses to launch without a check | Mint and deliver online, verify offline forever |
| Sync | Server owns the data | Server relays changes between authoritative devices |
| Collaboration | Document lives on the server | Presence and transport relay over CRDT documents |
| AI features | Developer proxy sees all prompts | BYOK direct calls, see [04-byok-pattern.md](04-byok-pattern.md) |
| Updates | Forced, blocking | Offered, skippable |

If the server's death breaks sharing but not the user's own data and daily use, the architecture held.

## The subtler costs, briefly

Four smaller items that surprise first-time local-first builders:

- **Support without visibility.** No server logs means no dashboard telling you what went wrong. Debugging happens through user-supplied exports and reproduction steps. Build good in-app diagnostics (local logs the user can choose to send) or fly blind.
- **No usage analytics by default.** You will not know which features get used unless you add opt-in telemetry, and adding it honestly (off by default, clearly disclosed) yields sparse data. Product decisions lean more on user conversations than on charts.
- **Migration debt compounds.** Every schema change must carry every past version forward, forever, on machines you cannot touch. Discipline in [03-architecture-patterns.md](03-architecture-patterns.md) is what keeps this manageable.
- **The user's machine is hostile terrain.** Antivirus false positives, weird locales, full disks, clock skew. A cloud app runs on one machine you control; a local-first app runs on ten thousand machines you have never seen.

None of these are dealbreakers. All of them belong in the estimate.

## The discipline answer: when to pick local-first

After the costs above, the decision rule we actually use:

**Pick local-first when the data is personal and the value is the tool, not the network.**

Unpacked:

- **The data is personal.** One person's notes, money, health, drafts, keys, files. The user is the natural owner, privacy claims matter, and a server adds exposure without adding value. Every argument in [02-why-it-matters.md](02-why-it-matters.md) fires at full strength.
- **The value is the tool.** The user pays for what the software does to their data: edits it, organizes it, analyzes it, transforms it. That value survives the vendor, which makes pay-once honest and the seven ideals in [01-what-local-first-means.md](01-what-local-first-means.md) reachable.
- **The value is NOT the network.** If the product is valuable because other users are there (marketplaces, social products, cross-user search), the network is the product, a server is the product's home, and forcing local-first onto it produces a worse product wearing an ideology. Build it server-first without apology, and be honest in the other direction too.

Borderline cases exist: collaborative documents are personal data plus a small network, which is why CRDTs are worth their cost there and mostly not elsewhere. But the rule sorts the vast majority of product ideas in one sentence, and it keeps you from the two failure modes this field produces: cloud apps cosplaying as local-first in their marketing, and local-first purists shipping products that needed a server and refused to admit it.

Personal data, tool value: build local-first, take the tradeoffs in this file with open eyes, and make promises your architecture can keep after you are gone. That is the whole discipline.
