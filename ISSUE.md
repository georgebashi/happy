# `happy connect` OAuth tokens are not end-to-end encrypted and use overly broad scopes

## Summary

The `happy connect` command authenticates users with OpenAI, Anthropic, and Google via OAuth, then sends the resulting tokens to the Happy server (`api.happy-servers.com`). Unlike session data, which is end-to-end encrypted with a user-held key, these vendor tokens are encrypted server-side with a server-held secret (`HANDY_MASTER_SECRET`). The server can decrypt them at any time.

The Google/Gemini OAuth flow requests the `cloud-platform` scope with a refresh token, which is effectively root access to the user's entire GCP account — not just Gemini.

No code in the server codebase currently uses these tokens to call any vendor API.

## OAuth scopes requested

The three `happy connect` subcommands request OAuth scopes with very different levels of access.

The [Gemini flow](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateGemini.ts#L22-L26) requests `cloud-platform`, `userinfo.email`, and `userinfo.profile`. The `cloud-platform` scope grants access to all GCP services the authenticated account can reach — Cloud Storage, BigQuery, Compute Engine, IAM, Cloud SQL, Secret Manager, etc. The flow also sets [`access_type: 'offline'`](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateGemini.ts#L236), which requests a refresh token that can mint new access tokens without further user interaction.

The [Claude flow](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateClaude.ts#L18) requests `user:inference`, which allows making Claude API inference calls on behalf of the user.

The [Codex flow](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateCodex.ts#L253) requests `openid profile email offline_access` — identity-only scopes (profile and email), plus `offline_access` for a refresh token.

## Encryption model differs from documented E2E

The project documents end-to-end encryption as a core property. The [README](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/README.md#L8) states "Use Claude Code or Codex from anywhere with end-to-end encryption," and further [describes](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/README.md#L84) the project as "End-to-end encrypted — Your code never leaves your devices unencrypted." The [server README](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/README.md#L11) claims "Zero Knowledge - The server stores encrypted data but has no ability to decrypt it."

Session data (messages, code, artifacts) is encrypted client-side with the user's key, consistent with these claims. Vendor tokens from `happy connect` are not — they are encrypted with a server-held `HANDY_MASTER_SECRET` via a `KeyTree` from `privacy-kit`. The project's own [encryption documentation](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/docs/encryption.md#L518) notes this: "These are encrypted with a server-only KeyTree derived from `HANDY_MASTER_SECRET` and are not end-to-end encrypted."

This distinction is not communicated to users. The [CLI help text](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect.ts#L61-L63) describes the feature only as: "The connect command allows you to securely store your AI vendor API keys in Happy cloud."

## Server-side token handling

Tokens are [encrypted with the server-held key and stored](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts#L248-L267) on registration. The server exposes endpoints that [decrypt and return individual tokens](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts#L269-L292) or [all vendor tokens at once](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts#L312-L332) in plaintext. Both the [CLI](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/api/api.ts#L292) and [mobile app](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-app/sources/sync/apiServices.ts) transmit tokens to the same server endpoint.

## Tokens are not used by any server-side feature

No code in the server calls any vendor API using stored tokens. The `ServiceAccountToken` model is referenced only for CRUD operations in [`connectRoutes.ts`](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts) and a status check in [`accountRoutes.ts`](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/accountRoutes.ts#L26). The [`lastUsedAt` field](https://github.com/georgebashi/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/prisma/schema.prisma#L249) in the database schema is never updated.
