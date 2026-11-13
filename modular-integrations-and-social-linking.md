---
status: shipped
collections: [connected-accounts, users, customers, studios]
features: [integrations]
---

# Modular Integrations Engine & Social Account Linking

A universal provider-adapter architecture for third-party platform integrations (Discord, Twitter/X, Bluesky, Twitch, etc.) enabling automated community server onboarding, role synchronization, and external platform automations.

## Scope & Purpose

2monke artists and buyers interact across external social platforms. Rather than scattering one-off columns (`discordId`, `discordToken`, etc.) across multiple collections:
1. **Universal Adapter Pattern**: Third-party services implement a standard `IntegrationAdapter` contract.
2. **Automated Server Onboarding & Role Sync**: When linking Discord, users are automatically added to the 2monke Discord guild via the Discord Bot API and granted their respective community role (**`@Artist`** or **`@Customer`**).
3. **Extensibility**: Adding new providers (Twitter/X, Bluesky, Twitch) requires only implementing an adapter class without changing the database schema or OAuth route logic.

## Placement

Connected accounts live on **My Studio → Integrations** (`/admin/integrations`). Payouts (`/admin/payouts`) is deliberately money-only: Stripe Connect, payout status, and the Express dashboard. Customer and artist profile pages expose the same card from their own account settings. Keeping third-party account linking out of Payouts prevents “how I get paid” and “which community accounts are linked” from sharing one page.

The page is account-scoped, not tenant-scoped: OAuth state is signed with the signed-in user, so a superadmin viewing another studio cannot connect that artist's Discord from their own session.

## Architecture

```
User clicks "Connect Discord" on My Studio → Integrations (or the profile card)
  │
  ▼
GET /api/integrations/[provider]/connect
  │
  ├─ 1. Authenticate user from session headers (getAuthenticatedUser)
  ├─ 2. Resolve audience ('artist' if user in 'users', 'buyer' if in 'customers')
  ├─ 3. Generate HMAC SHA-256 signed state (createOAuthState: userId, collection, audience, returnUrl)
  ├─ 4. Lookup adapter from registry (getIntegrationAdapter(provider))
  └─ 5. Redirect to provider authorization URL (with identify + guilds.join scopes)
  │
  ▼
External OAuth Consent Screen (Discord, Twitter, etc.)
  │
  ▼
GET /api/integrations/[provider]/callback?code=...&state=...
  │
  ├─ 1. Verify tamper-proof state signature & timestamp (verifyOAuthState)
  ├─ 2. Exchange authorization code for access/refresh tokens (adapter.exchangeCode)
  ├─ 3. Fetch user profile & external ID (adapter.getProfile)
  ├─ 4. Run post-connection sync hook (adapter.onConnected)
  │     └─ Discord Adapter:
  │        PUT https://discord.com/api/v10/guilds/{GUILD_ID}/members/{USER_ID}
  │        ├─ If not in server: Joins user to guild with [@Artist or @Customer]
  │        └─ If already in server: Adds role to existing membership
  ├─ 5. Upsert connection record (saveConnectedAccount)
  └─ 6. Redirect back to client returnUrl (?integration=discord&status=connected)
```

## The `IntegrationAdapter` Contract

All third-party services implement the [`IntegrationAdapter`](file:///Users/eugenepucko/Projects/artpack-parallel/src/lib/integrations/types.ts) interface:

```typescript
export interface IntegrationAdapter {
  readonly id: IntegrationProviderId
  readonly name: string
  readonly icon: string
  readonly description: string

  getAuthUrl(params: {
    state: string
    redirectUri: string
    audience: 'artist' | 'buyer'
  }): string

  exchangeCode(code: string, redirectUri: string): Promise<OAuthTokens>

  getProfile(tokens: OAuthTokens): Promise<ProviderProfile>

  onConnected(params: {
    user: ConnectedAccountUser
    audience: 'artist' | 'buyer'
    tokens: OAuthTokens
    profile: ProviderProfile
  }): Promise<SyncResult>

  onDisconnected?(params: {
    user: ConnectedAccountUser
    accountId: string
  }): Promise<void>
}
```

## Security & Anti-CSRF Protection

The state parameter used during the OAuth handshake is protected against forgery:
- **Payload**: Contains `userId`, `userCollection`, `audience`, `returnUrl`, `nonce`, and `timestamp`.
- **HMAC SHA-256**: Signed using `process.env.PAYLOAD_SECRET`.
- **Expiration**: Rejected if age exceeds 15 minutes.
- **Audience Enforcement**: Server-signed during `/connect` and cannot be tampered with by client query parameters.

## Payload `connected-accounts` Collection

- **Slug**: `connected-accounts`
- **Primary Key**: Text prefixed ID (`acc_...` via `prefixedIdField('connectedAccount')`)
- **Ownership**: Polymorphic relationship to either `users` (artists) or `customers` (buyers).
- **Tenant Scope**: Optional relationship to `studios` when connected for a specific studio.
- **Access Control**: Users can only read, update, and revoke their own connected accounts. Super admins retain administrative visibility.
- **Fields**:
  - `provider`: Select (`discord`, `twitter`, `bluesky`, `twitch`, `google`).
  - `providerAccountId`: Unique snowflake or platform user ID.
  - `providerUsername`: Display handle on the external platform.
  - `scopes`: JSON array of granted OAuth permissions.
  - `accessToken` & `refreshToken`: System-only read tokens.
  - `expiresAt`: Token expiration date.
  - `metadata`: JSON tracking server membership, assigned roles, and avatar URLs.
  - `status`: Select (`active`, `expired`, `revoked`).
  - `lastSyncedAt`: Timestamp of the latest provider sync.

## Domain Server Actions

Located in [`src/features/integrations/`](file:///Users/eugenepucko/Projects/artpack-parallel/src/features/integrations):
- [`getConnectedAccounts`](file:///Users/eugenepucko/Projects/artpack-parallel/src/features/integrations/actions/get-connected-accounts.action.ts): Returns token-free `ConnectedAccountSummary` rows for the signed-in user (or an admin-targeted user). Raw OAuth tokens never cross the server/client boundary.
- [`saveConnectedAccount`](file:///Users/eugenepucko/Projects/artpack-parallel/src/features/integrations/actions/save-connected-account.action.ts): Server-only OAuth callback writer. Rejects callers without a real Payload context so a browser cannot forge a connection.
- [`disconnectAccount`](file:///Users/eugenepucko/Projects/artpack-parallel/src/features/integrations/actions/disconnect-account.action.ts): Revokes an integration and executes adapter-specific cleanup hooks. The browser sends only the provider; the server resolves the owner from the session.

## Environment Configuration

```bash
# Discord Integration
DISCORD_CLIENT_ID="your_discord_app_client_id"
DISCORD_CLIENT_SECRET="your_discord_app_client_secret"
DISCORD_BOT_TOKEN="your_discord_bot_token"
DISCORD_PUBLIC_KEY="your_discord_application_public_key"
DISCORD_GUILD_ID="your_community_server_id"
DISCORD_ARTIST_ROLE_ID="role_id_for_verified_artists"
DISCORD_CUSTOMER_ROLE_ID="role_id_for_verified_buyers"
DISCORD_ARTIST_CHANNEL_ID="optional_channel_id_for_artist_inquiries"
```

## Discord Slash Commands & HTTP Interactions (`/ask`)

Rather than hosting a persistent WebSocket bot process, the platform handles Discord slash commands via **Serverless HTTP Interactions**:

```
Discord User types /ask question: "..." in Discord server
  │
  ▼
Discord sends POST https://<platform-domain>/api/discord/interactions
  │
  ├─ 1. Ed25519 Signature Verification (X-Signature-Ed25519 + X-Signature-Timestamp)
  │     └─ Uses native Node.js crypto.verify with DISCORD_PUBLIC_KEY
  │     └─ Returns HTTP 401 if signature is missing, invalid, or tampered
  │
  ├─ 2. Discord Handshake:
  │     └─ If interaction type === 1 (PING): Returns { type: 1 } (PONG)
  │
  └─ 3. Slash Command Execution:
        └─ If interaction type === 2 (APPLICATION_COMMAND) & name === 'ask':
           ├─ Acknowledges immediately with { type: 5 } (DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE)
           │  to satisfy Discord's strict 3-second timeout.
           ├─ Resolves audience ('artist' or 'buyer'):
           │  1. Explicit slash command option 'audience' (if provided)
           │  2. Channel context: checks if channel matches DISCORD_ARTIST_CHANNEL_ID
           │  3. Member role context: checks if member has DISCORD_ARTIST_ROLE_ID
           │  4. Fallback: defaults to 'buyer'
           ├─ Forwards to the help service via askHelpService({ question, audience, source: 'discord' })
           │  (src/lib/help/client.ts → POST {HELP_SERVICE_URL}/api/help/ask/service, HMAC-signed).
           │  Answering lives in 2monke/help-service; this route is only a proxy.
           ├─ Formats Discord Embed with branded colors, answer text, relevant doc citations,
           │  and human support escalation links.
           └─ Patches original Discord interaction response via:
              PATCH https://discord.com/api/v10/webhooks/{APP_ID}/{TOKEN}/messages/@original
```

### Slash Command Registration

Application commands are registered via the CLI script:

```bash
# Register to your development Discord Guild (instant update, no cache delay)
yarn discord:register

# Or register globally across all Discord servers (up to 1 hour propagation)
yarn discord:register --global
```

## Component Breakdown & Files Changed

| Path | Purpose |
|---|---|
| `src/collections/ConnectedAccounts.ts` | Collection config for external OAuth accounts |
| `src/components/admin/IntegrationsViewServer.tsx` | Server wrapper for `/admin/integrations`; resolves accounts before first paint |
| `src/features/integrations/components/IntegrationsView.tsx` | Admin Integrations page (admin-native, token-free UI) |
| `src/features/integrations/components/ConnectedAccountsCard.tsx` | Customer/artist profile card, shared state hook |
| `src/features/integrations/hooks/use-connected-accounts.ts` | Shared OAuth-return, connect, disconnect state machine |
| `src/features/integrations/lib/connected-account-summary.ts` | Token-stripping projection for client components |
| `src/features/integrations/lib/catalog.ts` | Client-safe integration names/descriptions |
| `src/lib/utils/id.ts` | Added `connectedAccount: 'acc_'` ID prefix |
| `src/payload.config.ts` | Registered `ConnectedAccounts` in `withAdminLeaveGuard` |
| `src/lib/integrations/types.ts` | Core types, OAuth interfaces, and adapter contracts |
| `src/lib/integrations/state.ts` | HMAC SHA-256 OAuth state generator and verifier |
| `src/lib/integrations/registry.ts` | Central adapter registry with automatic provider registration |
| `src/lib/integrations/providers/discord.adapter.ts` | Discord OAuth and Guild Join / Role Sync adapter |
| `src/lib/integrations/discord/interactions.ts` | Ed25519 signature validation, audience resolution, and embed formatting |
| `src/lib/integrations/index.ts` | Barrel export for integrations library |
| `src/app/api/discord/interactions/route.ts` | HTTP Interactions endpoint verifying Ed25519 and proxying `/ask` to the help service |
| `2monke/discord-gateway` | Slash command registration and the forum websocket listener (separate repository) |
| `src/features/integrations/actions/save-connected-account.action.ts` | Action for saving/updating connected accounts |
| `src/features/integrations/actions/get-connected-accounts.action.ts` | Action for querying user/studio integrations |
| `src/features/integrations/actions/disconnect-account.action.ts` | Action for disconnecting and revoking an integration |
| `src/features/integrations/index.ts` | Barrel export for integrations feature actions |
| `src/app/api/integrations/[provider]/connect/route.ts` | OAuth initiation route with server-side audience resolution |
| `src/app/api/integrations/[provider]/callback/route.ts` | OAuth callback route handling token exchange & guild sync |
| `tests/int/integrations-engine.int.spec.ts` | 7 automated integration tests verifying registry, state HMAC, and Discord guild join |
| `tests/int/discord-interactions.int.spec.ts` | 12 automated tests for Ed25519 verification, PING/PONG, `/ask` deferred response, and embeds |

## Verification Reference

```bash
# Run integration tests for Discord interactions and integrations engine
yarn test:int tests/int/discord-interactions.int.spec.ts tests/int/integrations-engine.int.spec.ts

# Connected-accounts surfaces: admin view rendering, OAuth return contract,
# connect/callback route behaviour
yarn test:int tests/int/integrations-view.int.spec.ts tests/int/integrations-return.int.spec.ts \
  tests/int/integrations-connect.int.spec.ts tests/int/integrations-callback.int.spec.ts

# Admin design contract (local literals, !important budget, mobile flattening)
yarn test:int tests/int/admin-theme-contract.int.spec.ts tests/int/admin-mobile-borderless-views.int.spec.ts

# Ensure customer help assistant and internal ask tests stay green
yarn test:int tests/int/help-assistant.int.spec.ts tests/int/ask-pipeline.int.spec.ts

# Validate TypeScript compilation
yarn tsc --noEmit
```

## Manual QA

1. `/admin/integrations` — connect Discord in an environment with Discord credentials;
   confirm the callback returns to this page, the row shows `@handle`, `Joined`, and the
   role, and the URL no longer carries `integration`/`status` params.
2. Disconnect — the confirmation appears inline (no browser `confirm()`), the row is removed,
   and a success notice is shown.
3. Without `DISCORD_CLIENT_ID`/`SECRET`, the Connect button returns to the page with a
   “not configured yet” notice instead of hitting Discord.
4. Payouts (`/admin/payouts`) shows Stripe only, with a “Manage integrations” pointer.
5. Customer profile — the same card renders from server-resolved accounts and survives a
   reload in the connected state.
