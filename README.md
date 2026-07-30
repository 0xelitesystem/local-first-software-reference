# Local-First Software Reference

A plain-language reference on what local-first software actually means, the architecture patterns behind it, and the honest tradeoffs, written by practitioners who build local-first tools.

## Contents

- [01-what-local-first-means.md](01-what-local-first-means.md) - The core idea, where the term comes from, the seven ideals in plain words, and what local-first is not
- [02-why-it-matters.md](02-why-it-matters.md) - Ownership, privacy, longevity, speed, cost honesty, and the trust argument
- [03-architecture-patterns.md](03-architecture-patterns.md) - Plain files, embedded storage, single-file HTML tools, desktop shells, and pay-once licensing with offline key verification
- [04-byok-pattern.md](04-byok-pattern.md) - Bring Your Own Key: how it works, why it beats a proxy server, and its honest limits
- [05-sync-and-collaboration.md](05-sync-and-collaboration.md) - Sync as an optional layer, the conflict problem, CRDTs in plain words, and simpler alternatives that are often enough
- [06-honest-tradeoffs.md](06-honest-tradeoffs.md) - What you give up, which features genuinely need servers, and when to pick local-first

## Who this is for

Working developers deciding whether to build local-first, and anyone who wants to understand the architecture without wading through academic papers. Everything here comes from shipping real local-first apps: desktop tools, browser tools, and the licensing and AI-integration patterns that go with them. This reference is itself part of an open-source portfolio of client-side tools where every tool runs entirely in the browser.

## More

Part of a catalog of single-file browser tools and plain-language references, all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/). Built by [elitesystem.ai](https://elitesystem.ai).

## License

MIT. See [LICENSE](LICENSE).
