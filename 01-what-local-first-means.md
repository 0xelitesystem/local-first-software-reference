# What Local-First Actually Means

## The one-sentence version

Local-first software keeps your data on your device, and the software keeps working when the server does not exist.

That second clause is the test. Plenty of apps cache data locally. Plenty of apps have an offline mode. Local-first means the local copy is the real copy, and any server involved is an optional convenience, not the source of truth.

We build local-first apps for a living: desktop tools, browser tools, and the plumbing that goes with them. This reference is what we wish someone had handed us at the start.

## A short history of how we got here

The term is recent; the idea is not. It helps to see the pendulum:

- **The desktop era.** Software ran on your machine and wrote files to your disk. You owned your data by default, because there was nowhere else for it to be. What you gave up: collaboration was floppy disks and email attachments, and syncing two machines was manual labor.
- **The cloud era.** The data moved to servers. In exchange we got real-time collaboration, access from any device, and nothing to install. What we gave up crept in quietly: the authoritative copy of your work now sits in a company's database, gated by their login, their pricing, and their continued existence.
- **The local-first synthesis.** The claim of the local-first movement is that this was a false trade. With the right architecture (and, for the collaboration piece, the right data structures, see [05-sync-and-collaboration.md](05-sync-and-collaboration.md)), you can keep the cloud era's conveniences while restoring the desktop era's default: your data, on your device, yours.

Local-first is not nostalgia for the desktop era. It is the position that the cloud era overpaid, and that we now know how to get most of the benefits back without the ownership transfer.

## Where the term comes from

The term was coined in the 2019 essay "Local-first software: You own your data, in spite of the cloud" by Martin Kleppmann, Adam Wiggins, Peter van Hardenberg, and Mark McGranaghan, published by the research lab Ink and Switch. If you read one primary source on this topic, read that one. It is long but plainly written, and everything in this field traces back to it.

The essay's core complaint: cloud apps gave us collaboration and convenience, but quietly took ownership. Your documents live in someone else's database, behind someone else's login, on someone else's uptime. If the company pivots, gets acquired, or raises prices, your data is a hostage in the negotiation.

The essay's core proposal: build software where the primary copy of the data lives on the user's own device, and treat servers as optional infrastructure for sync and sharing.

## The seven ideals, in plain words

The Ink and Switch essay defines seven ideals for local-first software. Here they are without the academic framing:

1. **No spinners.** Work happens against local data, so the interface responds instantly. There is no round trip to wait on. See [02-why-it-matters.md](02-why-it-matters.md) for why this is a bigger deal than it sounds.

2. **Your work is not trapped on one device.** Local-first is not single-device. The data should be able to follow you to your laptop, phone, and desktop. How much work this takes is one of the honest costs, covered in [06-honest-tradeoffs.md](06-honest-tradeoffs.md).

3. **The network is optional.** The app works fully offline. Connectivity improves the experience (sync, sharing) but is never required to read or edit your own data.

4. **Collaboration still works.** You should be able to work with other people, ideally in real time. This is the hardest ideal to hit, and it is where CRDTs come in. See [05-sync-and-collaboration.md](05-sync-and-collaboration.md).

5. **The Long Now.** Your files should be readable in decades, not until the vendor's next pivot. Open formats and plain files are the practical answer here.

6. **Security and privacy by default.** Data that stays on the device cannot leak from a server breach, because there is no server holding it. End-to-end encryption covers the sync path when there is one.

7. **You retain ultimate ownership and control.** No account termination, no terms-of-service change, and no company shutdown can take your data away, because the authoritative copy was always yours.

No shipping app hits all seven perfectly. The ideals are a direction, not a certification. A tool that hits five of them honestly is more local-first than a tool that claims all seven in marketing copy.

## What local-first is NOT

The term gets stretched. Here is where we draw the lines.

### Not just an offline cache

A web app with a service worker and an "offline mode" is usually still server-first. The test: if the company deletes your account tonight, what happens to your data tomorrow?

- Offline-capable, server-first: the cache is a replica. The server is authoritative. Delete the account, and the cache is orphaned data you cannot fully use, export, or trust.
- Local-first: the device copy is authoritative. Delete the server, and nothing about your daily use changes except sync.

An offline cache answers "what if the network is down for an hour." Local-first answers "what if the company is gone forever."

### Not just self-hosted

Self-hosting moves the server into your closet. That is a real improvement for ownership, and we respect it, but it is a different answer to a different question. You still have a server: something to patch, back up, secure, and keep running. Your data is still in a database that requires a running service to be useful.

Local-first removes the server from the critical path entirely. A self-hosted app that dies when its container stops is not local-first. The two approaches are compatible (a self-hosted sync server behind a local-first app is a great combination), but they are not the same thing.

### Not an anti-cloud purity test

Local-first apps are allowed to use the network. Sync services, license checks, update checks, and optional AI features that call an API are all fine, as long as the app degrades to fully working, not to a login wall, when those are unavailable. The architecture question is what breaks when the server is gone, not whether a server is ever contacted.

### Not the same as end-to-end encrypted cloud

An end-to-end encrypted cloud service is a privacy improvement over a plain one: the vendor stores your data but cannot read it. It is still server-first, though. The vendor can lock your account, lose the data, shut down, or change terms, and your access dies with the service. Encryption answers "can they read it." Local-first answers "do they control it." The strongest designs do both: local authoritative copy, encrypted optional sync.

## The distinctions at a glance

| Question | Cloud app | Cloud app with offline mode | Self-hosted | Local-first |
|---|---|---|---|---|
| Where is the authoritative copy? | Vendor server | Vendor server | Your server | Your device |
| Works with no network, indefinitely? | No | Partially, temporarily | No (needs the service running) | Yes |
| Survives vendor shutdown? | No | No | Mostly (until it breaks) | Yes |
| Survives your own server dying? | n/a | n/a | No | Yes |
| Who does backups by default? | Vendor | Vendor | You (as admin) | You (as user) |
| Data readable without the app running? | No | No | Usually no | Yes, if formats are open |

The last row is the sleeper. Local-first only delivers its longevity promise when paired with open formats, which is why format choice is treated as an architecture decision in [03-architecture-patterns.md](03-architecture-patterns.md), not an afterthought.

## A working definition you can test against

We use this checklist when deciding whether to call one of our own tools local-first:

- [ ] The authoritative copy of user data is on the user's device.
- [ ] Every core feature works with the network cable unplugged.
- [ ] The user can get their data out in an open, documented format without asking anyone.
- [ ] If our company disappeared tomorrow, the installed app would keep working indefinitely.
- [ ] Any server we operate is for sync, licensing, or updates only, never for storing the primary copy of user data.

If a tool fails any of these, we do not use the label. The label is worth something only if it is falsifiable.

Note what is absent from the checklist: "never uses the network," "no company involved," "free of charge." Local-first is compatible with commercial software, with paid sync services, and with online features. It is a statement about where authority over the data lives, nothing more and nothing less.

## Where to go next

- Why any of this matters in practice: [02-why-it-matters.md](02-why-it-matters.md)
- How to actually build it: [03-architecture-patterns.md](03-architecture-patterns.md)
- AI features without a middleman: [04-byok-pattern.md](04-byok-pattern.md)
- Sync, done in the right order: [05-sync-and-collaboration.md](05-sync-and-collaboration.md)
- What it costs you: [06-honest-tradeoffs.md](06-honest-tradeoffs.md)
