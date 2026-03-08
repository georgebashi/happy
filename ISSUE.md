# `happy connect` requests broad GCP scope and stores tokens with server-side encryption

## Summary

The `happy connect` command's Google/Gemini OAuth flow requests the `cloud-platform` scope, which grants access to all GCP services on the user's account, not just Gemini. This token, along with tokens for Anthropic and OpenAI, is encrypted with a server-held secret rather than the user's end-to-end encryption key. No code in the server codebase currently uses any of these tokens.

Specifically:

- The Google/Gemini OAuth scope (`cloud-platform`) grants access to all GCP services, not just Gemini.
- Vendor tokens are stored with server-side encryption rather than the user's E2E encryption key. This is documented in [internal architecture docs](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/docs/encryption.md#L518) but not surfaced to users during the connect flow.
- No server code currently calls any vendor API with these tokens.
- The three providers use very different scope levels (full GCP access vs inference-only vs identity-only), which makes the intended use case unclear.

## The Gemini flow requests broad GCP access

The [Gemini OAuth flow](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateGemini.ts#L22-L26) requests `cloud-platform`, `userinfo.email`, and `userinfo.profile`. The `cloud-platform` scope grants access to all GCP services the authenticated account can reach — Cloud Storage, BigQuery, Compute Engine, IAM, Cloud SQL, Secret Manager, etc. The flow also sets [`access_type: 'offline'`](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateGemini.ts#L236), which requests a refresh token that can mint new access tokens indefinitely without further user interaction.

For a Gemini integration, a narrower scope like `generativelanguage.googleapis.com` would be sufficient. The `cloud-platform` scope additionally covers Cloud Storage, Compute Engine, IAM, and every other GCP service the account can reach.

## Vendor tokens are not end-to-end encrypted

The project's [README](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/README.md#L8) describes "end-to-end encryption," and the [server README](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/README.md#L11) describes "Zero Knowledge - The server stores encrypted data but has no ability to decrypt it."

Session data (messages, code, artifacts) is encrypted client-side with the user's key, consistent with these descriptions. Vendor tokens from `happy connect` use a different model — they are encrypted with a server-held `HANDY_MASTER_SECRET` via a `KeyTree` from `privacy-kit`. The [encryption documentation](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/docs/encryption.md#L518) notes this: "These are encrypted with a server-only KeyTree derived from `HANDY_MASTER_SECRET` and are not end-to-end encrypted."

The server can decrypt these tokens and exposes endpoints that [return individual tokens](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts#L269-L292) or [all vendor tokens at once](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts#L312-L332) in plaintext. The [CLI help text](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect.ts#L61-L63) describes the feature as: "The connect command allows you to securely store your AI vendor API keys in Happy cloud," without distinguishing the encryption model from the E2E encryption used for session data.

## Tokens are stored but not used

No code in the server calls any vendor API using stored tokens. The `ServiceAccountToken` model is referenced only for CRUD operations in [`connectRoutes.ts`](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/connectRoutes.ts) and a status check in [`accountRoutes.ts`](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/sources/app/api/routes/accountRoutes.ts#L26). The [`lastUsedAt` field](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-server/prisma/schema.prisma#L249) in the database schema is never updated.

The three providers request very different levels of access. The [Gemini flow](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateGemini.ts#L22-L26) requests `cloud-platform` (all GCP services). The [Claude flow](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateClaude.ts#L18) requests `user:inference` (inference only). The [Codex flow](https://github.com/slopus/happy/blob/d343330c86ab966969aecd82be4aecbad7ec4238/packages/happy-cli/src/commands/connect/authenticateCodex.ts#L253) requests `openid profile email offline_access` (identity only, no API access). It would be helpful to understand the intended use case, since these different scope levels would each support different features.
