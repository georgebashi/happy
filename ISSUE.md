# `happy connect` OAuth tokens are not end-to-end encrypted and use overly broad scopes

## Summary

The `happy connect` command authenticates users with OpenAI, Anthropic, and Google via OAuth, then sends the resulting tokens to the Happy server (`api.happy-servers.com`). Unlike session data, which is end-to-end encrypted with a user-held key, these vendor tokens are encrypted server-side with a server-held secret (`HANDY_MASTER_SECRET`). The server can decrypt them at any time.

The Google/Gemini OAuth flow requests the `cloud-platform` scope with a refresh token, which is effectively root access to the user's entire GCP account — not just Gemini.

No code in the server codebase currently uses these tokens to call any vendor API.

## OAuth scopes requested

### Google/Gemini: `cloud-platform`

**`packages/happy-cli/src/commands/connect/authenticateGemini.ts:22-26`**:
```typescript
const SCOPES = [
    'https://www.googleapis.com/auth/cloud-platform',
    'https://www.googleapis.com/auth/userinfo.email',
    'https://www.googleapis.com/auth/userinfo.profile',
].join(' ');
```

The `cloud-platform` scope grants access to all GCP services the authenticated account can reach — Cloud Storage, BigQuery, Compute Engine, IAM, Cloud SQL, Secret Manager, etc. The flow also sets `access_type: 'offline'` (line 236), which requests a refresh token that can mint new access tokens without further user interaction.

### Anthropic/Claude: `user:inference`

**`packages/happy-cli/src/commands/connect/authenticateClaude.ts:18`**:
```typescript
const SCOPE = 'user:inference';
```

This scope allows making Claude API inference calls on behalf of the user.

### OpenAI/Codex: `openid profile email offline_access`

**`packages/happy-cli/src/commands/connect/authenticateCodex.ts:253`**:
```typescript
['scope', 'openid profile email offline_access'],
```

These are identity-only scopes (profile and email), plus `offline_access` for a refresh token.

## Encryption model differs from documented E2E

The project documents end-to-end encryption as a core property:

**`README.md:8`**:
> Use Claude Code or Codex from anywhere with end-to-end encryption.

**`README.md:84`**:
> 🔐 **End-to-end encrypted** — Your code never leaves your devices unencrypted

**`packages/happy-server/README.md:11`**:
> 🔐 **Zero Knowledge** - The server stores encrypted data but has no ability to decrypt it

Session data (messages, code, artifacts) is encrypted client-side with the user's key, consistent with these claims. Vendor tokens from `happy connect` are not — they are encrypted with a server-held `HANDY_MASTER_SECRET` via a `KeyTree` from `privacy-kit`.

The project's internal documentation notes this distinction:

**`docs/encryption.md:518`**:
> These are encrypted with a server-only KeyTree derived from `HANDY_MASTER_SECRET` and **are not end-to-end encrypted**.

This distinction is not communicated to users. The CLI help text describes the feature as:

**`packages/happy-cli/src/commands/connect.ts:61-63`**:
> The connect command allows you to securely store your AI vendor API keys in Happy cloud.

## Server-side token handling

Tokens are stored encrypted and can be decrypted by the server on request:

**`packages/happy-server/sources/app/api/routes/connectRoutes.ts:248-267`** — storage:
```typescript
const encrypted = encryptString(
    ['user', userId, 'vendors', request.params.vendor, 'token'],
    request.body.token
);
await db.serviceAccountToken.upsert({ /* ... */ });
```

**`connectRoutes.ts:269-332`** — retrieval (decrypts and returns plaintext):
```typescript
// GET /v1/connect/:vendor/token
return reply.send({
    token: decryptString(['user', userId, 'vendors', request.params.vendor, 'token'], token.token)
});

// GET /v1/connect/tokens — returns all decrypted vendor tokens
for (const token of tokens) {
    decrypted.push({
        vendor: token.vendor,
        token: decryptString(['user', userId, 'vendors', token.vendor, 'token'], token.token)
    });
}
```

Both the CLI (`packages/happy-cli/src/api/api.ts:292`) and mobile app (`packages/happy-app/sources/sync/apiServices.ts`) transmit tokens to the same server endpoint.

## Tokens are not used by any server-side feature

No code in the server calls any vendor API using stored tokens. The `ServiceAccountToken` model is referenced only for CRUD operations in `connectRoutes.ts` and a status check in `accountRoutes.ts:26`. The `lastUsedAt` field in the database schema (`packages/happy-server/prisma/schema.prisma:249`) is never updated.
