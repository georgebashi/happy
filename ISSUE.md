# `happy connect` collects powerful OAuth tokens, sends them to a server that can read them, and doesn't use them

## If you have used `happy connect`, revoke your OAuth sessions now

Happy Coder's `happy connect` command asks users to authenticate with OpenAI, Anthropic, and Google via OAuth, then sends the resulting tokens to Happy's remote server (`api.happy-servers.com`). The project's prominent claims of "end-to-end encryption" do not apply to these tokens — they are encrypted with a server-held key, meaning the server operator can decrypt them at will. No code in the codebase actually uses these tokens to provide any feature. The tokens are collected, sent to a third party, and stored.

**If you have used `happy connect codex`, `happy connect claude`, or `happy connect gemini`, you should immediately revoke the OAuth sessions:**

- **OpenAI**: https://platform.openai.com/settings/authentication — revoke active sessions
- **Anthropic**: https://console.anthropic.com/settings/keys — revoke active sessions
- **Google (Gemini)**: https://myaccount.google.com/permissions — revoke access for the Happy Coder app

## What these tokens can actually do

The OAuth scopes requested by `happy connect` vary dramatically in power. Users consenting to the OAuth prompts may not fully appreciate what they are granting, especially given that the project markets itself as "end-to-end encrypted."

### Google/Gemini: `cloud-platform` — full access to all GCP services

**`packages/happy-cli/src/commands/connect/authenticateGemini.ts:22-26`**:
```typescript
const SCOPES = [
    'https://www.googleapis.com/auth/cloud-platform',
    'https://www.googleapis.com/auth/userinfo.email',
    'https://www.googleapis.com/auth/userinfo.profile',
].join(' ');
```

The `cloud-platform` scope is the broadest Google Cloud scope that exists. It grants full access to **all** GCP services the user's account can reach — not just Gemini. This includes:

- **Cloud Storage** — read/write/delete any GCS bucket
- **BigQuery** — query any dataset
- **Compute Engine** — create, modify, or delete VMs
- **IAM** — view and potentially modify permissions
- **Cloud SQL, Pub/Sub, Cloud Functions, Secret Manager**, and every other GCP API

With `access_type: 'offline'` (line 236), the flow also requests a **refresh token**, so the server can mint new access tokens indefinitely without further user interaction. An `offline` refresh token with `cloud-platform` scope is, in effect, persistent root access to a user's entire Google Cloud account.

### Anthropic/Claude: `user:inference` — make API calls as the user

**`packages/happy-cli/src/commands/connect/authenticateClaude.ts:18`**:
```typescript
const SCOPE = 'user:inference';
```

This token allows making Claude API calls charged to the user's Anthropic account. The server could consume the user's credits or quota.

### OpenAI/Codex: `openid profile email offline_access` — identity only (but with refresh)

**`packages/happy-cli/src/commands/connect/authenticateCodex.ts:253`**:
```typescript
['scope', 'openid profile email offline_access'],
```

These are identity scopes — they can read the user's OpenAI profile and email, but not make API calls. However, `offline_access` grants a refresh token, giving persistent access to identity information.

## The "end-to-end encryption" claim creates a false sense of safety

Happy prominently markets end-to-end encryption as a core feature:

**`README.md:8`**:
> Use Claude Code or Codex from anywhere with end-to-end encryption.

**`README.md:84`**:
> 🔐 **End-to-end encrypted** — Your code never leaves your devices unencrypted

**`packages/happy-server/README.md:11`**:
> 🔐 **Zero Knowledge** - The server stores encrypted data but has no ability to decrypt it

These claims are true for *session data* (messages, code, artifacts), which is encrypted client-side with the user's key. But `happy connect` vendor tokens follow a completely different model — they are encrypted server-side with a server-held secret (`HANDY_MASTER_SECRET`). The server can decrypt every stored vendor token at any time.

The project's own internal documentation acknowledges this:

**`docs/encryption.md:518`**:
> These are encrypted with a server-only KeyTree derived from `HANDY_MASTER_SECRET` and **are not end-to-end encrypted**.

But this distinction is never surfaced to users. When a user who chose Happy *because* of its E2E encryption claims is prompted to `happy connect gemini`, they have no reason to suspect these tokens are handled differently from everything else. The CLI help text reinforces this:

**`packages/happy-cli/src/commands/connect.ts:61-63`**:
> The connect command allows you to securely store your AI vendor API keys in Happy cloud. This enables you to use these services through Happy without exposing your API keys locally.

No disclosure is made that the server operator can read these tokens.

## Tokens are collected but never used

A search across the entire server codebase for any code that reads a vendor token and calls an external API returns zero results. The only references to `ServiceAccountToken` are CRUD operations:

- **`connectRoutes.ts`** — store, retrieve, and delete tokens
- **`accountRoutes.ts:26`** — list which vendors are connected (status display)

The `lastUsedAt` field in the database schema (`packages/happy-server/prisma/schema.prisma:249`) is never updated — it is always `null`. There is no feature in the product that requires these tokens on the server.

## How the tokens are stored and accessed

**`packages/happy-server/sources/app/api/routes/connectRoutes.ts:248-267`** — The server encrypts tokens with a key it holds:
```typescript
const encrypted = encryptString(
    ['user', userId, 'vendors', request.params.vendor, 'token'],
    request.body.token
);
await db.serviceAccountToken.upsert({ /* ... */ });
```

**`connectRoutes.ts:269-332`** — The server has endpoints that decrypt and return tokens in plaintext:
```typescript
// GET /v1/connect/:vendor/token
return reply.send({
    token: decryptString(['user', userId, 'vendors', request.params.vendor, 'token'], token.token)
});

// GET /v1/connect/tokens — returns ALL decrypted vendor tokens
for (const token of tokens) {
    decrypted.push({
        vendor: token.vendor,
        token: decryptString(['user', userId, 'vendors', token.vendor, 'token'], token.token)
    });
}
```

Both the CLI (`packages/happy-cli/src/api/api.ts:292`) and mobile app (`packages/happy-app/sources/sync/apiServices.ts`) send tokens to the same endpoint in plaintext JSON over HTTPS.

## Recommended actions for users

**Revoke your OAuth sessions immediately** if you have used any `happy connect` subcommand:

- **OpenAI**: Revoke active sessions at https://platform.openai.com/settings/authentication
- **Anthropic**: Revoke active sessions at https://console.anthropic.com/settings/keys
- **Google (Gemini)**: Revoke Happy Coder's access at https://myaccount.google.com/permissions

The Google revocation is especially important given the `cloud-platform` scope.

Until this is resolved, **do not use `happy connect`**.

## Suggested remediation for maintainers

1. **Don't collect what you don't use** — No feature currently requires these tokens on the server. Don't collect them until one exists.
2. **Use least-privilege scopes** — The Gemini flow should not request `cloud-platform`. If the intent is only Gemini API access, use `https://www.googleapis.com/auth/generative-language` or a similarly narrow scope.
3. **Disclose the security model** — If vendor tokens must be stored server-side, the `happy connect` command and documentation should clearly state they are not end-to-end encrypted and are accessible to the server operator.
4. **Consider true E2E for vendor tokens** — Encrypt vendor tokens client-side with the user's key, the same way session data is handled.
5. **Qualify the "end-to-end encrypted" claims** — The current blanket statements are misleading when a major feature uses server-side encryption.
