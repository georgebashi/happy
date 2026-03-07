# Vendor API tokens (OpenAI, Anthropic, Gemini) are sent to remote server with no disclosed use and misleading encryption claims

## Summary

The `happy connect` command collects users' OAuth tokens for OpenAI, Anthropic, and Gemini and transmits them to the Happy server (`api.happy-servers.com`), where they are stored with server-side encryption. The server can decrypt these tokens at any time. There is no code anywhere in the codebase that uses these tokens to provide functionality to the user. Meanwhile, the project's README and documentation prominently advertise "end-to-end encryption," which does not apply to these vendor tokens.

## Details

### 1. Tokens are collected and sent to a remote server

When a user runs `happy connect codex`, `happy connect claude`, or `happy connect gemini`, the CLI performs an OAuth flow and then sends the resulting tokens to the remote server:

**`packages/happy-cli/src/commands/connect.ts:92-117`** — After authenticating with each vendor, the CLI calls `api.registerVendorToken()`, which POSTs the token as a JSON string to the server:

**`packages/happy-cli/src/api/api.ts:292`** — The `registerVendorToken` method sends the token in plaintext over HTTPS:
```typescript
async registerVendorToken(vendor: 'openai' | 'anthropic' | 'gemini', apiKey: any): Promise<void> {
    const response = await axios.post(
        `${configuration.serverUrl}/v1/connect/${vendor}/register`,
        { token: JSON.stringify(apiKey) },
        // ...
    );
}
```

Note the parameter is typed `apiKey: any`, which is unusual for a codebase whose own style guide states "Strict typing: No untyped code."

### 2. Tokens are stored with server-side encryption — the server can decrypt them at will

**`packages/happy-server/sources/app/api/routes/connectRoutes.ts:248-267`** — The server encrypts the token using a `KeyTree` derived from `HANDY_MASTER_SECRET` and stores it in the database:

```typescript
const encrypted = encryptString(
    ['user', userId, 'vendors', request.params.vendor, 'token'],
    request.body.token
);
await db.serviceAccountToken.upsert({ /* ... */ });
```

**`packages/happy-server/sources/app/api/routes/connectRoutes.ts:269-292`** and **lines 312-332** — The server has endpoints that decrypt and return the tokens in plaintext to any authenticated request:

```typescript
// GET /v1/connect/:vendor/token — returns a single decrypted token
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

This is **not** end-to-end encryption. The server holds `HANDY_MASTER_SECRET` and can decrypt every stored vendor token at any time.

### 3. The tokens are never used for anything

A search across the entire server codebase for any code that reads a vendor token and calls an external API (OpenAI, Anthropic, or Gemini) returns zero results. The only references to `serviceAccountToken` in the server are:

- **`connectRoutes.ts`** — store, retrieve, and delete tokens (CRUD operations only)
- **`accountRoutes.ts:26`** — list which vendors are connected (status display only)

The `lastUsedAt` field in the database schema (`packages/happy-server/prisma/schema.prisma:240-253`) is never updated — it is always `null`. There is no feature in the product that requires these tokens.

### 4. README and docs make misleading end-to-end encryption claims

**`README.md:8`**:
> Use Claude Code or Codex from anywhere with end-to-end encryption.

**`README.md:84`**:
> 🔐 **End-to-end encrypted** — Your code never leaves your devices unencrypted

**`packages/happy-cli/CLAUDE.md`** (developer docs):
> End-to-end encryption for all communications

These claims are accurate for *session data* (messages, artifacts, etc.), which is encrypted client-side. However, they are misleading in the context of the `happy connect` feature, where vendor tokens use a fundamentally different security model. The project's own internal documentation acknowledges this distinction:

**`docs/encryption.md:518`**:
> These are encrypted with a server-only KeyTree derived from `HANDY_MASTER_SECRET` and **are not end-to-end encrypted**.

**`docs/backend-architecture.md:354-362`** — The architecture diagram explicitly separates "Client-side Encryption" (session data) from "Server-side Encryption" (vendor tokens), showing that vendor tokens are encrypted with a server-held key, not the user's key.

This distinction is not surfaced to users. The CLI help text (`connect.ts:61-63`) says only:

> The connect command allows you to securely store your AI vendor API keys in Happy cloud. This enables you to use these services through Happy without exposing your API keys locally.

No disclosure is made that these tokens are accessible to the server operator.

### 5. Database schema

**`packages/happy-server/prisma/schema.prisma:240-253`**:
```prisma
model ServiceAccountToken {
    id         String    @id @default(cuid())
    accountId  String
    account    Account   @relation(fields: [accountId], references: [id], onDelete: Cascade)
    vendor     String
    token      Bytes     // Encrypted token
    metadata   Json?     // Optional vendor metadata
    lastUsedAt DateTime?
    createdAt  DateTime  @default(now())
    updatedAt  DateTime  @updatedAt

    @@unique([accountId, vendor])
    @@index([accountId])
}
```

## Impact

Users who run `happy connect` hand over their OAuth tokens for OpenAI, Anthropic, and/or Gemini to a third-party server that can decrypt them at any time, for a feature that does not yet exist. The project's prominent "end-to-end encrypted" marketing does not apply to these tokens, and no clear disclosure is made about the different security model.

## Suggested remediation

1. **Disclose the security model clearly** — The `happy connect` command and associated documentation should clearly state that vendor tokens are stored with server-side encryption (not E2E) and are accessible to the server operator.
2. **Don't collect what you don't use** — If no feature currently requires these tokens on the server, don't collect them. If the plan is to use them in the future, wait until the feature exists.
3. **Consider true E2E encryption for vendor tokens** — Encrypt vendor tokens client-side with the user's key, the same way session data is handled. Only decrypt them on the client when needed.
4. **Correct marketing claims** — Qualify the "end-to-end encrypted" claims to exclude vendor tokens, or implement actual E2E for them.
