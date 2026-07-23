# Architecture Patterns for Local-First Software

This is the practical menu. Every pattern here is something we have shipped, and each section says when to reach for it and when not to. The patterns compose: a real app usually combines two or three of them.

## Pattern 1: Plain files as source of truth

The oldest pattern is still the best default. User data lives in files the user can see, in a directory the user chose, in a format other software can read.

**What it looks like:**

- Notes as markdown files
- Structured data as JSON or CSV
- Collections as a directory of files, not a blob

**Why it wins:** the operating system becomes your free feature team. Users get backup (any backup tool works on files), sync (any file-sync service works, see [05-sync-and-collaboration.md](05-sync-and-collaboration.md)), search (OS indexing), and portability (copy the folder) without you writing a line of code.

**The rule that makes it work:** the file format must be documented and open. A plain file in an undocumented binary format is a database with worse ergonomics. If your working format must be binary for performance, ship a one-click export to an open format and treat that exporter as core functionality, not a nice-to-have.

```json
{
  "format": "myapp-notes",
  "formatVersion": 2,
  "exportedAt": "2026-07-01T12:00:00Z",
  "notes": [
    { "id": "n1", "title": "Example", "body": "Plain text.", "updatedAt": "2026-06-30T09:15:00Z" }
  ]
}
```

**Format versioning is part of the pattern, not hygiene.** Because installed apps never fully die (see [06-honest-tradeoffs.md](06-honest-tradeoffs.md)), version 9 of your app will meet files written by version 1. The discipline:

- Every file and every export carries a formatVersion.
- Readers accept every version ever shipped; writers write only the current version.
- Migrations run forward only, on read, and never destroy the original file until the migrated copy is safely written.
- Breaking the format without a version bump is treated as a data-loss bug, because that is what it is.

**When it does not fit:** high-write-rate structured data (use SQLite), or data with heavy cross-record queries.

## Pattern 2: Embedded storage

When plain files are too loose, embed the database in the app. Three tiers, in order of capability:

### SQLite (desktop, mobile, and increasingly the browser via WASM)

The workhorse. A full relational database in a single file, zero configuration, no server process, battle-tested for decades. Use it when you have structured records, real queries, or more data than you want to parse into memory on launch.

The single-file property is the local-first bonus: the entire database is one file the user can copy, back up, and move. Document that fact in your app. "Your data is the .db file at this path" is a feature.

```sql
-- The whole "server" is this file on the user's disk
-- myapp.db: schema for a typical local-first tool
CREATE TABLE items (
  id TEXT PRIMARY KEY,
  payload TEXT NOT NULL,       -- JSON, kept open-format even inside the db
  updated_at INTEGER NOT NULL  -- unix ms, needed later if sync is added
);
```

Note the updated_at column even in a no-sync v1. Timestamps and stable ids cost nothing now and are the difference between "sync is a feature we can add" and "sync requires a schema migration across every installed copy."

### IndexedDB (browser)

The browser's real storage engine: asynchronous, transactional, stores structured objects and blobs, holds gigabytes. Use it for any browser tool with meaningful data volume. The API is awkward; wrap it once (or use a tiny wrapper library) and move on.

The honest caveat: browser storage is owned by the browser profile, not the user. Clearing site data deletes it, and browsers may evict storage under pressure unless the app requests persistence. Every browser tool we ship with IndexedDB also ships export-to-file, and prompts the user to use it. Browser storage is working storage; files are ownership.

### localStorage (browser)

Synchronous, string-only, small quota (around 5 MB). Correct uses: settings, UI state, small tool state, the API key slot described in [04-byok-pattern.md](04-byok-pattern.md). Incorrect use: the user's actual documents. If you are JSON-encoding an array of user records into localStorage, you have outgrown it; move to IndexedDB.

### Choosing at a glance

| Data | Right home |
|---|---|
| Settings, UI state, small tool state | localStorage (browser) or a JSON settings file (desktop) |
| Real browser-tool data, blobs, big collections | IndexedDB, with export-to-file always available |
| Desktop structured records, real queries | SQLite, single file, user-visible path |
| User-facing documents the user should own outright | Plain files in open formats (Pattern 1) |
| Secrets such as API keys | OS secret storage on desktop; see [04-byok-pattern.md](04-byok-pattern.md) for the browser reality |

## Pattern 3: The single-file HTML tool

Our favorite pattern for small tools, and the backbone of our browser portfolio: the entire application is one HTML file. All markup, styles, and code inline. No build step output split across files, no external dependencies, no network requests at all.

**Properties you get for free:**

- **Runs anywhere.** Double-click the file, it opens in any browser, on any OS, from a USB stick, forever. There is no deployment and no environment.
- **Auditable by view-source.** The entire program is readable by anyone who opens the file in a text editor. For tools handling sensitive input, this is the strongest trust artifact possible: the claim "this never phones home" can be checked in one read, and verified live with the network tab open. This is the architecture-claim advantage from [02-why-it-matters.md](02-why-it-matters.md) in its purest form.
- **Trivially archivable.** The tool is a document. Email it, back it up, version it in git as one file.
- **Web-grade distribution without a server dependency.** Serve the file from any static host and users get it with a click; hand them the file and they own it outright. Same artifact both ways.
- **The Long Now, achieved by accident.** Browsers have kept backward compatibility for decades. A self-contained HTML file from 2010 still runs. Your tool inherits that.

**Discipline required:**

- Zero external requests. Not for fonts, not for icons, not for libraries, not for analytics. The moment the file needs the network, every property above degrades.
- Inline everything: icons as inline SVG, any images as data URIs, all code written in or bundled into the file.
- Keep it honest under the size that makes view-source auditing plausible. A 200 KB tool is readable; a 5 MB bundle is technically one file but no longer auditable in practice.
- State persistence is localStorage plus explicit export-to-file, per the table above.

**When it does not fit:** anything needing OS integration, large persistent data, or background operation. That is when you reach for the next pattern.

## Pattern 4: Desktop shells

When the tool needs to be an app (file system access, system tray, global shortcuts, notifications, real windows), wrap the web stack in a desktop shell. Electron is the mature default; Tauri is the lighter alternative if binary size matters more than ecosystem.

**What the shell buys a local-first app specifically:**

- Real file system access: plain-files-as-truth (Pattern 1) with native open/save dialogs.
- SQLite without WASM contortions.
- OS-provided secret storage for encrypting things like API keys at rest (Electron exposes this as safeStorage; it backs onto the platform keychain or DPAPI). This is load-bearing for [04-byok-pattern.md](04-byok-pattern.md).
- No CORS. The main process can call any HTTPS API directly, which matters because some AI providers do not permit browser-origin calls.
- Offline by default: nothing about the shell requires a network.

**The local-first discipline inside a shell:**

- Keep the renderer sandboxed and the privileged surface small; the web content should not have more OS power than it needs.
- Keep user data in user-visible locations (documents folder, not buried app data, wherever reasonable), and tell the user the path.
- Never let auto-update or license checks block app launch. A desktop app that refuses to start without a network check has quietly stopped being local-first.
- Treat the shell as packaging, not architecture. The data patterns (1 and 2) do not change because a shell is wrapped around them.

**The honest cost:** installers, code signing, per-OS builds, and update distribution are real ongoing work. See [06-honest-tradeoffs.md](06-honest-tradeoffs.md).

## Pattern 5: Pay-once licensing with offline key verification

The business model has to match the architecture. If the app works forever without your servers, the license check must also work forever without your servers. The pattern: signed license keys verified offline with a public key baked into the app.

**How it works, end to end:**

1. You hold an Ed25519 private key. It never ships with the app and never leaves your infrastructure.
2. When someone buys, you mint a license: a small payload (name, email, product, tier, issue date) signed with the private key.
3. The key the customer receives is `payload.signature`, both base64-encoded, for example: `EXAMPLE-PAYLOAD.EXAMPLE-SIGNATURE` (real keys are longer; the structure is what matters).
4. The app ships with only the public key and verifies the signature locally. No network call, ever, on the verification path.

```js
// Inside the app: verification is pure local math.
// PUBLIC_KEY is baked into the build. Placeholder shown; use your real public key.
const PUBLIC_KEY_B64 = "your-public-key-here";

function verifyLicense(licenseKey) {
  const [payloadB64, sigB64] = licenseKey.trim().split(".");
  if (!payloadB64 || !sigB64) return { ok: false };
  const payload = base64Decode(payloadB64);
  const sig = base64Decode(sigB64);
  const ok = ed25519Verify(sig, payload, base64Decode(PUBLIC_KEY_B64));
  return ok ? { ok: true, license: JSON.parse(textDecode(payload)) } : { ok: false };
}
```

Ed25519 verification is available in Node's crypto module, in small audited libraries, and in the Web Crypto API in modern runtimes. Signatures are 64 bytes; keys are 32 bytes; verification is fast enough to run on every launch.

**Why this specific design:**

- **Matches the promise.** The customer paid once for software that runs forever; the license check cannot be the thing that breaks that promise when your server or your company goes away.
- **Forgery-proof without secrecy in the app.** Shipping the public key reveals nothing. Only the private key can mint valid licenses.
- **A server is optional, not required.** You can still run a small licensing server for purchase fulfillment and key delivery: checkout completes, the server mints and emails the key. The point is that it sits before the sale, never in the launch path. If that server dies, every sold copy keeps working; you have a sales outage, not a customer outage.
- **Tiering fits in the payload.** Free versus pro, seat counts, and version ceilings ("free updates for versions released before this date") are just fields in the signed payload. The app reads them locally.

**Honest notes:** offline verification does not prevent key sharing, and determined users can patch any client-side check. Price the product fairly, treat the check as a gate for honest customers, and do not burn engineering months on DRM that contradicts your own architecture. A local-first app with aggressive online license enforcement is a contradiction users will notice.

## Composing the patterns

A typical shape for a serious local-first product:

- Desktop shell (Pattern 4)
- SQLite or plain files as truth (Patterns 1 and 2), with open-format export always available
- Pay-once license verified offline (Pattern 5)
- Optional AI features via BYOK ([04-byok-pattern.md](04-byok-pattern.md))
- Optional sync as a layer on top, never the foundation ([05-sync-and-collaboration.md](05-sync-and-collaboration.md))

And a typical shape for a small free tool: one self-contained HTML file (Pattern 3), localStorage for state, export to file for ownership.

Both shapes pass the checklist at the end of [01-what-local-first-means.md](01-what-local-first-means.md), and both keep every promise in [02-why-it-matters.md](02-why-it-matters.md) structurally rather than by policy. That is the whole point of choosing patterns this way: the marketing page and the architecture diagram end up saying the same thing.
