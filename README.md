# Asteria

A separate, modular Discord bot for community management, moderation, security, and engagement. Built with TypeScript, discord.js, Node.js 24, and SQLite. **This project is independent of Kuroo** and does not read or migrate Kuroo data.

The first release supplies working baseline modules for the [full feature list](docs/features.md). Modules start disabled and can be enabled per server. Provider setup and the precise limits of each module are documented below; a Discord test server is required to verify actual permissions, gateway behavior, and voice playback.

## Quick start

Requirements: **Node 24.13+**, npm, a Discord application, and a persistent writable disk. Install **FFmpeg** on PATH for music. No database service is required.

```sh
npm ci --ignore-scripts
cp .env.example .env
# Fill in DISCORD_TOKEN and DISCORD_CLIENT_ID in .env.
npm run deploy
npm run build
npm start
```

Use `DISCORD_GUILD_ID` for immediate registration in a test server. Omit it for global registration; remove old guild commands when switching scopes to avoid duplicates. Redeploy commands after updates. `npm run dev` runs the TypeScript watcher.

In the Discord Developer Portal, enable **Server Members Intent**. For content filters/counting, also enable **Message Content Intent** and set `MESSAGE_CONTENT_ENABLED=true`. Slash commands do not require Message Content Intent. Invite tracking additionally needs `INVITE_TRACKING_ENABLED=true` and Manage Server. Restart after changing environment variables.

Install the bot with `bot` and `applications.commands` scopes. Grant only permissions used by enabled modules:

- Basic replies: View Channels, Send Messages, Embed Links, Attach Files, Read Message History.
- Moderation: Moderate Members, Kick Members, Ban Members, Manage Messages, Manage Channels, Manage Roles.
- Role panels and verification: Manage Roles, Add Reactions, Read Message History.
- Audit protection: View Audit Log; Ban Members only for owner-selected ban mode.
- Tickets/temporary rooms: Manage Channels, Manage Roles; Move Members for voice lobby routing.
- Music: View Channel, Connect, Speak. Webhooks: Manage Webhooks.

Put Asteria above members and roles it should manage. **Administrator is unnecessary.** Discord still gives administrators access to private channels. Self-assigned, automatic and verified roles must be below Asteria, unmanaged, and have no server permissions.

## First server setup

All commands use subcommands. Command replies are private, except deliberately published panels, polls, announcements and community messages.

```text
/help guide
/config set key:logChannel value:123456789012345678
/module enable name:logging
/module enable name:moderation
/config set key:staffRole value:123456789012345679
/config set key:ticketCategory value:123456789012345680
/module enable name:tickets
/ticket open topic:I need help
```

Settings accept strings, JSON booleans, integers, `null`, or arrays as appropriate. Snowflake IDs stay strings and retain full precision. `/config show` lists all keys. `/module list` shows every feature flag. The [configuration guide](docs/configuration.md) includes setup recipes.

## Architecture

- `src/store.ts`: SQLite transactions, durable jobs, server-scoped settings/records, profiles, retention, schema version.
- `src/commands.ts`, `src/interactions.ts`: slash schemas, shared routing, ephemeral error handling, buttons, select menus and modals.
- `src/moderation.ts`, `src/gateway.ts`: manual actions, event handlers, protection, logging, roles, XP and community automation.
- `src/community.ts`, `src/support.ts`, `src/music.ts`: community, private support, verification and voice workflows.
- `src/scheduler.ts`: durable delivery with leases, retries, giveaway draws, birthdays and RSS notifications.
- `src/http.ts`: readiness, authenticated statistics API and signed Stripe subscription webhooks.
- `src/database.ts`: process lock and physical database check/backup/restore tools.

Every database operation includes a guild scope except deliberate operator-only global tools. Guild managers can add role gates with `/access`; these never replace built-in Discord permission requirements. Operators are explicit IDs in `OPERATOR_IDS`, not arbitrary server administrators. Anti-nuke trust, enforcement policy and enablement are restricted to the server owner.

## External integrations

### Music

Set `MUSIC_TRACKS` to a JSON object mapping short names to HTTPS audio URLs that the operator controls and is entitled to play. Example: `{"ambient":"https://your-domain.example/audio.ogg"}`. Enable `music`, join voice, then `/music play track:ambient`. Queues allow 25 tracks. Manage Channels is required to skip, stop or disconnect shared playback. FFmpeg, Opus and DAVE encryption support are included/required by the voice stack. No search, arbitrary member URLs, YouTube scraping, Spotify DRM bypass, or Lavalink service is provided. Queues are transient; scheduled tasks and community data persist.

### Social notifications

Set `SOCIAL_FEEDS` to named HTTPS RSS/Atom feed URLs. Enable `social`, then `/social subscribe feed:news channel:#updates`. Feeds are polled every five minutes, have a 1 MB response limit and a 10 second timeout. Redirects are rejected. The initial item establishes a baseline without posting history; subsequent new latest items are published. This supports providers exposing feeds; OAuth-only social APIs need separate adapters. URLs are trusted operator configuration.

### Premium and subscriptions

Set `STRIPE_SECRET_KEY`, `STRIPE_PRICE_ID` (a **recurring** price), `STRIPE_WEBHOOK_SECRET`, and `BILLING_RETURN_URL`. Expose `POST /webhooks/stripe` behind HTTPS and register `customer.subscription.created`, `customer.subscription.updated`, and `customer.subscription.deleted` in Stripe. The server owner uses `/premium subscribe`; subscription metadata binds updates to the server ID. Signatures use the raw body, a five minute timestamp tolerance, durable event deduplication and per-subscription version checks. `/premium status` reports billing state. Operators can set a manual status. **All modules remain available without a subscription in this release**; there is no undisclosed paywall or automatic role billing.

`STRIPE_API_BASE_URL` supports local testing. Never send real Stripe keys to an emulator. The tagged CodeRabbit v0.0.1 Stripe emulator verifies checkout creation; its documented catalog does not provide subscription lifecycle APIs/events, so signed subscription state transitions are tested locally with synthetic events. Live checkout/payment remains a deployment verification step.

### API and health

Default bind: `127.0.0.1:3000`. `GET /healthz` returns 200 only with a ready Discord client and healthy SQLite connection; otherwise 503. Set `API_TOKEN` to at least 32 random characters to enable `GET /api/guilds/:guildId/stats` with `Authorization: Bearer …`. The token grants operator-level aggregated access across connected guilds. Protect it in a secret manager. For container/public use configure `HEALTH_HOST` and a TLS reverse proxy; expose only required paths. There is no public write/admin API.

## Operations and recovery

Run **one process per bot token and SQLite file**, on a persistent local disk; do not use a shared network filesystem. Rate windows and music queues reset on restart. The database process lock prevents concurrent bot/restore processes. After an unclean exit, inspect the recorded PID in `data/asteria.sqlite.lock` and verify the process has stopped before removing the stale lock.

```sh
# Stop the bot first for physical database operations.
npm run database -- check
npm run database -- backup /secure/backups/asteria.sqlite
npm run database -- restore /secure/backups/asteria.sqlite --force
```

Restore checks SQLite integrity/schema and creates a safety copy before replacing an existing database. Backups contain private data: restrict access, encrypt off-host copies and set an expiration policy. `/backup` provides **same-server configuration snapshots**, not Discord message/channel recovery. Owner-only restore validates channel/role references.

Jobs persist across restarts, lease for two minutes, retry three times, then appear in `/developer jobs`. Operators can `/developer retry`. Delivery is **at least once**: a crash after sending but before committing may duplicate a notification. Disabling a module suppresses and completes its due jobs. Giveaway draws are saved before delivery so retries preserve winners. Birthday announcements use UTC and one daily marker per member.

Moderation intent is persisted before Discord enforcement. An interrupted API action may leave a pending case: check Discord audit logs before repeating it. Ticket/voice reservations survive channel creation failures when a channel may exist; reconcile an interrupted `creating` record with Discord before deleting the record. Ticket close requires an explicit button confirmation and deletes the channel; no transcript is saved. Empty temporary rooms are removed on voice events and by daily recovery cleanup.

Anti-nuke protection detects thresholds **after** audit events. Alert is the default; optional banning affects only members Asteria can ban. It cannot guarantee prevention, restore deleted channels/roles/messages, or act above its role. Raid protection times out new, young accounts from the join threshold onward (not earlier joins), and exempts moderators/trusted users. Content filtering is deterministic text matching, not image classification. A verification math question is a basic entry barrier, not identity proof.

## Validation

```sh
npm run check
npm test
npm run build
npm audit --omit=dev
```

Tests use real in-memory/file SQLite, synthetic Discord boundaries, ephemeral HTTP listeners, and no live credentials. The optional `ASTERIA_STRIPE_TEST_URL` test runs against the tagged Stripe emulator with the documented local seed. See [validation notes](docs/validation.md). Real Discord command registration, UI rendering, gateway events, voice UDP connectivity and actual Stripe payment require staging credentials.

## Privacy and language

`/privacy policy`, `/privacy export`, and `/privacy delete` provide server-scoped tools. Anonymous messages retain a private author reference until deletion/retention; they are anonymous to members, not the operator. Moderation/audit records and active support ownership have documented accountability exceptions. [Read the privacy policy](docs/privacy.md) before enabling anonymous features.

English command descriptions/help are provided. English and Spanish core response strings are supported with server locale and profile language preferences; command descriptions and most feature-specific text currently remain English. No claim of complete translation coverage is made.
