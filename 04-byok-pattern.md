# The BYOK Pattern: AI Features Without a Middleman

BYOK means Bring Your Own Key. It is how a local-first app adds AI features without betraying the architecture. The user supplies their own API key from the model provider, the app calls the provider directly from the user's machine, and the developer is not in the loop: no proxy, no relay, no custody of keys or data.

We use this pattern in every AI-enabled tool we ship. This file is the full case for it, the implementation details that matter, and the limits we state up front.

## The shape of the pattern

1. The user creates an account with a model provider and generates an API key in the provider's dashboard.
2. The user pastes the key into the app's settings once.
3. The app stores the key locally, encrypted where the platform allows (details below).
4. When the user invokes an AI feature, the app sends the request from the user's machine straight to the provider's HTTPS endpoint, authenticated with the user's key.
5. The response comes back to the user's machine. Nothing touches infrastructure the developer operates, because there is none.

The data path is two parties: the user and the provider they chose. The developer wrote the software and then got out of the way.

```js
// Direct call from the user's machine to the provider. No middleman.
async function callModel(promptText, settings) {
  const res = await fetch(settings.providerEndpoint, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "authorization": "Bearer " + settings.userApiKey  // the USER'S key, decrypted just-in-time
    },
    body: JSON.stringify({
      model: settings.modelName,
      messages: [{ role: "user", content: promptText }]
    })
  });
  if (!res.ok) throw new Error("Provider error " + res.status);
  return res.json();
}
```

## Why BYOK beats a proxy server

The conventional design is the opposite: the developer runs a server, holds one provider account, and relays user requests through it. That design is worse on both sides of the transaction, which is rare and worth spelling out.

### Better for the user's privacy

- **With a proxy:** every prompt and every document the user runs through the AI feature transits the developer's server. The developer can log it, an attacker who breaches the server can read it, and a subpoena can compel it. The user must trust a policy.
- **With BYOK:** the developer structurally cannot see prompts, outputs, or the key. There is no server to breach and no log to leak. The user extends trust only to the provider they already chose to have an account with. This is the architecture-claim-versus-policy-claim distinction from [02-why-it-matters.md](02-why-it-matters.md), applied to AI.

For tools that feed sensitive material to a model (journals, financial data, meeting audio, legal text), this is not a nice-to-have. It is the difference between a tool a careful person can use and one they cannot.

### Better for the developer

- **No key liability.** A proxy concentrates one high-value provider key (and often stored user content) on a server you must defend forever. BYOK means there is nothing of value to steal from you. Your worst-case breach is your marketing site.
- **No server cost.** Proxied AI features bill the developer per token, which forces subscriptions, quotas, and abuse fighting. BYOK usage bills the user's own provider account at cost, which is exactly the pay-once model from [03-architecture-patterns.md](03-architecture-patterns.md). Your costs do not scale with usage, so your pricing does not have to.
- **No data custody.** The moment user content transits your server, you have privacy-law obligations, retention questions, and breach liability. BYOK removes the entire category: you cannot mishandle data you never receive.
- **No abuse surface.** Nobody can farm your proxy for free tokens, because there is no proxy.

### The comparison in one table

| Concern | Developer-run proxy | BYOK |
|---|---|---|
| Who sees user prompts | Developer and provider | Provider only |
| Key that can be stolen from the developer | One high-value key | None |
| Developer's monthly cost | Scales with usage | Zero |
| Pricing model this forces | Subscription or credits | Pay once works |
| Breach blast radius | All users' content and the key | Nothing held, nothing lost |
| Who controls model spend | Developer (then rations it) | User, on their own account |
| Works if the developer disappears | No | Yes |

The last row is the local-first tell. A BYOK feature keeps working after the vendor is gone, exactly like the rest of a local-first app. A proxied feature is a leash.

## Key storage: encrypted where the platform allows

The key is a secret worth real money on the user's own bill, so store it with the best mechanism each platform offers, and be honest about the ceiling on each.

### Desktop (Electron and similar shells)

Use the OS secret facility. Electron exposes safeStorage, which encrypts with the platform keychain on macOS, the login keyring on Linux, and DPAPI on Windows. Encrypt on save, store only ciphertext on disk, decrypt just-in-time in the main process when a request is made.

```js
// Main process: never store the plaintext key on disk.
const { safeStorage } = require("electron");

function saveApiKey(plaintextKey) {
  if (!safeStorage.isEncryptionAvailable()) throw new Error("OS encryption unavailable");
  const encrypted = safeStorage.encryptString(plaintextKey);
  writeSettingsFile({ apiKeyEncrypted: encrypted.toString("base64") });
}

function loadApiKey() {
  const settings = readSettingsFile();
  const buf = Buffer.from(settings.apiKeyEncrypted, "base64");
  return safeStorage.decryptString(buf);  // decrypt only when needed
}
```

Two more rules that matter as much as the encryption: keep the key in the main process (the renderer requests completions, never the key itself), and never write the key into logs, error reports, or exported files.

### Browser tools

The browser has no OS keychain access from page context. The realistic options are localStorage or IndexedDB, which any code running on the page can read. Our practice:

- Store the key in localStorage under a clearly named slot, so the user can see and clear it.
- Say plainly in the UI that the key is stored in the browser and readable by the browser profile. No theater: client-side "encryption" with a hardcoded passphrase is obfuscation, not security, and claiming otherwise burns trust.
- Offer a session-only mode (key held in memory, gone on tab close) for users who prefer to paste per session.
- Because our browser tools are self-contained with no external dependencies and make no network calls except the user-initiated provider call, the user can verify by view-source and the network tab that the key goes to exactly one place. The self-containment discipline from [03-architecture-patterns.md](03-architecture-patterns.md) is what makes that verification meaningful; one third-party request on the page and the claim gets murky.

### Universal rules, any platform

- Masked input field, paste-friendly.
- Validate the pasted key immediately with one cheap test call, and show clear success or failure. Silent acceptance of a broken key is the worst first-run experience in this pattern.
- A visible "remove key" button that actually deletes the stored value.
- Key never appears in URLs, logs, analytics (you should not have analytics on it anyway), or crash reports.
- All features degrade gracefully with no key present: the app works, the AI features explain what is missing and link to setup.

## Honest limits

BYOK is the right pattern for local-first AI, not a free lunch. State these limits in your docs; users respect products that do.

- **The user needs their own provider account.** Sign-up, billing setup, and key generation are real friction, and for non-technical users they are the single biggest drop-off point. Good onboarding (a short walkthrough with screenshots per provider, in-app validation of the pasted key) reduces it but does not eliminate it. If your audience is fully non-technical, be honest with yourself about whether BYOK fits the product at all.
- **Per-provider quirks are your integration burden.** Endpoints, auth header formats, model names, rate limits, streaming formats, and error shapes differ across providers, and some providers do not permit direct browser-origin calls at all (a CORS restriction), which is one reason desktop shells matter for AI features (see [03-architecture-patterns.md](03-architecture-patterns.md)). Supporting several providers means maintaining several small adapters and keeping up when they change. Budget for it as permanent maintenance, not a one-time integration.
- **Key handling discipline is on both parties.** You must never leak the key; the user must treat it like a payment credential, set spend limits in the provider dashboard, and rotate it if exposed. Say so in the UI at the moment they paste it, not only in documentation nobody reads.
- **Cost visibility lives with the provider.** The user's usage bills to their own account, which is the honest place for it, but it means your app should estimate and display usage where possible rather than letting a surprise bill arrive.
- **No developer-side pooling benefits.** A proxy could give you centralized caching, volume discounts, or a free tier subsidized across users. BYOK gives all of that up. In our experience the privacy and liability wins outweigh it every time, but it is a real tradeoff, and it belongs on the list in [06-honest-tradeoffs.md](06-honest-tradeoffs.md) like every other one.

## The one-line summary

If your app is local-first everywhere except the AI feature, it is not local-first; BYOK is how the AI feature keeps the promise the rest of the architecture makes. For the promises themselves, see [02-why-it-matters.md](02-why-it-matters.md); for what still requires servers no matter what, see [06-honest-tradeoffs.md](06-honest-tradeoffs.md).
