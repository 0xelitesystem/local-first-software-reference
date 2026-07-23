# Why Local-First Matters

Six arguments, in descending order of how often we actually make them to users. None of them are hypothetical; every one maps to a failure mode that has happened to real products and real people.

## 1. Ownership: no vendor shutdown takes your data

Every year, a batch of cloud products shuts down, gets acquired and sunset, or pivots away from its users. The pattern is always the same: an email with a deadline, an export tool of varying quality, and then a dead login page. Whatever did not export cleanly is gone.

The count of documented shutdowns is in the hundreds; entire sites exist just to catalog discontinued products. The point is not that cloud companies are evil. The point is that companies are mortal and databases die with them.

Local-first sidesteps the whole failure mode:

- There is no deadline email, because there is nothing to migrate.
- The app you installed keeps running, because nothing about it depends on the vendor existing.
- The files it wrote keep opening, because they were always on your disk in a readable format.

This is also the honest framing for the developer side. When we ship a local-first tool, we are making a promise we can actually keep. We cannot promise our company will exist in ten years. We can promise the tool will still run in ten years, because we removed ourselves from the dependency graph. For a small shop, that is the only ten-year promise available.

## 2. Privacy: data that never leaves the device cannot leak from a server

Server breaches are not an edge case; they are a base rate. Credential dumps, misconfigured storage buckets, compromised employees, subpoenas, and plain bugs all end the same way: user data that lived on a server is now somewhere else.

The local-first answer is structural, not procedural:

- A procedural answer is "we have good security practices." It depends on people continuing to behave well, forever.
- A structural answer is "the data is not here." You cannot breach what was never collected.

No bucket to misconfigure. No production database to dump. No support engineer with read access. No log pipeline quietly retaining prompts. The entire class of server-side privacy failure is not mitigated; it is absent.

This matters most for tools that handle sensitive input: journals, financial records, health notes, interview prep, legal drafts, anything you would not paste into a random web form. For that category, "we promise we will not look" is a weaker offer than "we built it so we cannot look." See [04-byok-pattern.md](04-byok-pattern.md) for how this extends to AI features, which are the modern place this principle gets tested.

## 3. Longevity: files outlive companies

A plain file on your disk has a remarkable property: it does not need anyone's permission to keep existing. A markdown file from 2004 opens today. A database row in a defunct startup's Postgres instance from 2019 does not.

Formats matter as much as location. A local file in a proprietary binary format is only half the win; you own the bytes but not the meaning. The full win is local files in open formats: markdown, JSON, CSV, SQLite, standard image formats. Anything a different program could read in twenty years.

Our rule: if the user cannot open their data in software we did not write, we have not finished the job. Every tool ships with export to an open format, and where possible the working format IS the open format. The patterns are in [03-architecture-patterns.md](03-architecture-patterns.md).

Two practical corollaries worth stating:

- **Longevity is a feature you can test today.** Open the exported file in a generic editor. If a human can make sense of it, the format will survive. If it needs your app to be intelligible, it will not.
- **Longevity compounds with the single-file tool pattern.** A self-contained HTML tool is a document; browsers have carried decades of backward compatibility, so the tool ages like a file, not like a service.

## 4. Speed: no round trip

Every server-first interaction pays a network tax: request out, processing, response back. On a good connection that is tens of milliseconds; on hotel wifi it is seconds; on a plane it is infinity. The spinner is the interface admitting that your data is far away.

Local-first operations run at disk and memory speed:

- Opening a document is a file read, not a fetch.
- Searching your notes is a local index lookup, not an API call.
- Saving is a write, not an upload with a retry queue.

The app feels the way software felt before we normalized waiting for our own data.

This is not just comfort. Latency changes behavior: people batch their work, avoid small edits, and lose trust in tools that make them wait. Instant tools get used differently than slow tools. The Ink and Switch essay lists "no spinners" first among the seven ideals (see [01-what-local-first-means.md](01-what-local-first-means.md)) for a reason: it is the difference users feel in the first ten seconds, before any of the philosophy registers.

## 5. Cost honesty: no subscription needed to open your own data

Subscription pricing for cloud software is often justified, because servers cost money every month. But it creates a dark pattern when applied to your own data: stop paying, lose access to things you made. That is rent on your own work.

Local-first inverts the economics:

- The developer's costs are pay-once (building the software), so the price can be pay-once.
- There is no monthly server bill to pass through, because there are no servers in the product's critical path.
- Opening your own files costs the vendor nothing, so it should cost you nothing.

The test we apply: if a user pays once, uninstalls the app in five years, and still has every piece of their data in a readable format, the pricing was honest. Licensing that respects this model (offline key verification, no phone-home requirement) is covered in [03-architecture-patterns.md](03-architecture-patterns.md).

To be fair to subscriptions: when a product includes a real recurring service (hosted sync, server-side compute), a recurring price for that component is honest. The dishonesty is charging rent on the local part, the tool and the data, where nothing recurs. Split the pricing the way the architecture splits: pay once for the tool, subscribe only to genuinely recurring services, and keep the tool working when the subscription lapses.

## 6. The trust argument: sensitive input demands verifiable claims

This one is the umbrella over the other five. Any tool that handles sensitive input is making a trust claim, and there are only two kinds:

- **Policy claims**: "we do not sell your data," "we delete logs after 30 days," "employees cannot read your documents." You cannot verify any of these. You are trusting a document that the vendor can amend, and an internal culture you cannot see.
- **Architecture claims**: "the data never leaves your machine," "there is no server," "the app works with the network disabled." You can verify all of these. Unplug the network. Watch the traffic. Read the source when it is open.

Local-first lets a small developer make architecture claims, which is the only category of claim a small developer can make credibly. Nobody audits our privacy policy. Anybody can run a network monitor against our apps. For a solo shop or a small team, verifiable-by-design is not just ethics, it is the only trust model that scales without a brand.

The single-file browser tools in our portfolio take this to the extreme: the entire tool is one HTML file, auditable by view-source, running entirely in the browser with no external dependencies. The trust claim and the architecture are the same artifact.

### How to evaluate any tool's claims, in five minutes

The same test works on our tools and on anyone else's. We encourage users to run it:

1. Disconnect the network. Does every core feature still work?
2. Reconnect and watch the network tab or a traffic monitor during normal use. Where does traffic go, and does any of it contain your content?
3. Find your data on disk. Can you locate the actual file or database, and open it with something else?
4. Read the export. Is it complete, documented, and in a format other software understands?
5. Ask the shutdown question. If this vendor vanished tonight, what exactly would you lose?

A local-first tool passes all five. A cloud tool with good marketing passes none of them, however warm the privacy page reads.

## The counterweight

None of this is free. Multi-device sync becomes your problem, backups shift to the user, and some features genuinely require servers. We keep the sales pitch honest by keeping the costs in print: read [06-honest-tradeoffs.md](06-honest-tradeoffs.md) before deciding this architecture fits your product.
