# NeoServer

![Uploading image.png…]()

A complete OG Fortnite private server backend built with **TypeScript, Express, and MongoDB**.

NeoServer combines the best features from multiple open-source Fortnite backends into a single, fully typed TypeScript project — covering authentication, player profiles, cosmetics, gifting, refunds, friends, XMPP chat, party system, arena, XP progression, battle pass, matchmaking, tournaments, events, shop rotation, Support-a-Creator, EOS compatibility, and a full Discord bot.

> [!WARNING]
> Never publish API keys, tokens, passwords, or other secrets. Back up your configuration and database before making changes. Use this project responsibly.

---

## Features

<details>
<summary><strong>Authentication & sessions</strong></summary>

### Supported grant types

| Grant type | Description |
| --- | --- |
| `password` | Email + password login with bcrypt verification |
| `refresh_token` | Renews an existing session |
| `exchange_code` | Short-lived code → full session (used after Discord OAuth2) |
| `client_credentials` | Launcher and server-to-server authentication |

### Token format

Tokens match Epic's format exactly — `eg1~` prefix, RS256 signed JWT with the full Epic permissions payload. Three token types: access (8h), refresh (24h), client (4h). Stored in a typed in-memory Map with file fallback.

### Session management

- Checks ban status on every request via `verifyToken` middleware
- `DELETE /account/api/oauth/sessions/kill/:token` — kills a specific token
- `DELETE /account/api/oauth/sessions/kill?killType=OTHERS_ACCOUNT_CLIENT_CREDENTIALS` — kills all client tokens

### Discord OAuth2 web login

`GET /oauth2/login` redirects to Discord authorization. After the player authorizes, `/oauth2/callback` exchanges the code, finds or creates a game account linked to their Discord ID, and returns a page with an exchange code they paste into the launcher. No manual account creation needed.

**Implementation:** `src/routes/auth.ts`, `src/routes/discord.ts`, `src/discord/oauth2.ts`, `src/tokenManager/`

</details>

<details>
<summary><strong>Player profiles & MCP</strong></summary>

### Profile types

| Profile | Stored data |
| --- | --- |
| `athena` | Cosmetics, loadouts, quests, XP, season stats |
| `common_core` | V-Bucks, gifts, purchase history, SAC code |
| `profile0` | Alternate V-Bucks mirror |
| `creative` | Creative mode data |
| `collections` | Collection data |

### Profile commands (17 total)

| Command | Description |
| --- | --- |
| `QueryProfile` | Reads a profile, merges per-season XP data |
| `EquipBattleRoyaleCustomization` | Equips cosmetic to slot (Character, Backpack, Pickaxe, Glider, Contrail, Music, Loading, Emote, Wrap) |
| `BulkEquipBattleRoyaleCustomization` | Equips multiple cosmetics at once |
| `SetBattleRoyaleBanner` | Sets banner icon and color |
| `SetCosmeticLockerSlot` | Updates a locker slot |
| `SetCosmeticLockerBanner` | Updates locker banner |
| `CopyCosmeticLoadout` | Copies one loadout preset to another |
| `PurchaseCatalogEntry` | Buys a catalog item with V-Bucks |
| `GiftCatalogEntry` | Gifts a catalog item to one or more friends |
| `RefundMtxPurchase` | Refunds a V-Bucks purchase (max 3 lifetime refunds) |
| `SetAffiliateName` | Sets or clears a Support-a-Creator code |
| `SetReceiveGiftsEnabled` | Toggles gift receiving on/off |
| `RemoveGiftBox` | Opens or removes a gift box |
| `OpenGiftBox` | Opens a gift box and grants its items |
| `MarkItemSeen` | Clears the "new" indicator on an item |
| `SetItemFavoriteStatusBatch` | Favorites or unfavorites multiple items |
| `ClientQuestLogin` | Daily quest login refresh |
| `SetBattleRoyaleSeasonData` | Updates season XP and tier |
| `IncrementNamedCounterStat` | Increments a named counter |
| `ClaimLoginReward` | Claims the daily login reward |
| `FortRerollDailyQuests` | Rerolls one daily quest |

**Implementation:** `src/routes/mcp/`, `src/routes/mcp/commands/`

</details>

<details>
<summary><strong>XP & Battle Pass progression</strong></summary>

### How it works

The XP engine handles full level-up progression for all supported seasons.

1. A trusted service calls `POST /api/v1/xp/:username` with `{ amount: N }`
2. The engine reads per-season XP threshold data from `Responses/Athena/Data/Season{N}/SeasonXP.json`
3. For each level crossed: awards free and paid Battle Pass tier rewards from `SeasonFreeBattlepass.json` and `SeasonPaidBattlepass.json`
4. For seasons 1–10: converts Battle Stars to tiers using `SeasonBP.json`
5. Items already owned are skipped — no duplicates
6. Saves atomically with a 3-attempt optimistic lock on the profile revision

### Season data coverage

Seasons 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, and 15.

### Season Umbrella

`POST /api/v1/rewards/season_umbrella/:username` — grants the season victory umbrella if not already owned. Idempotent.

**Implementation:** `src/xp/`, `src/routes/xp.ts`

</details>

<details>
<summary><strong>Gifting</strong></summary>

### In-game gifting (GiftCatalogEntry)

1. Validates the sender has enough `Currency:MtxPurchased`
2. Validates friendship between sender and all receivers
3. Checks each receiver has gifting enabled and does not already own the item
4. All receivers validated before any V-Bucks are deducted (no partial failures)
5. Deducts V-Bucks across multiple currency slots proportionally
6. Creates a `GiftBox` item in each receiver's `common_core`
7. Sends a `com.epicgames.gift.received` XMPP notification to each receiver

### Discord V-Bucks gifting (/giftvbucks)

Discord slash command. 30-second cooldown. Transfers `Currency:MtxPurchased` between accounts and sends an XMPP notification.

### Refunding (RefundMtxPurchase)

Up to 3 lifetime refunds per account. Verifies refund credits, removes the item, restores V-Bucks, and records the refund in `mtxpurchasehistory`.

**Implementation:** `src/routes/mcp/commands/GiftCatalogEntry.ts`, `src/routes/mcp/commands/RefundMtxPurchase.ts`

</details>

<details>
<summary><strong>Friends system</strong></summary>

Full bidirectional friends system backed by MongoDB. States: `ACCEPTED`, `PENDING` (incoming/outgoing), `BLOCKED`.

### Endpoints

| Method | Path | Action |
| --- | --- | --- |
| `POST` | `/friends/api/v1/:accountId/friends/:friendId` | Send or accept friend request |
| `DELETE` | `/friends/api/v1/:accountId/friends/:friendId` | Remove or decline |
| `POST` | `/friends/api/v1/:accountId/blocklist/:friendId` | Block |
| `DELETE` | `/friends/api/v1/:accountId/blocklist/:friendId` | Unblock |
| `GET` | `/friends/api/public/friends/:accountId` | Full friends list with status |
| `GET` | `/friends/api/v1/:accountId/summary` | Friends, incoming, outgoing, blocklist |
| `PUT` | `/friends/api/v1/:accountId/friends/:friendId/alias` | Set alias |
| `DELETE` | `/friends/api/v1/:accountId/friends/:friendId/alias` | Remove alias |

Smart logic: if the target already sent a request to the caller, it auto-accepts both sides instead of creating a duplicate outgoing request.

Every friend list change sends an XMPP presence notification in real time.

**Implementation:** `src/routes/friends.ts`, `src/structs/friend.ts`

</details>

<details>
<summary><strong>XMPP real-time messaging</strong></summary>

Full XMPP over WebSocket server (port 3552 by default). Fortnite's game client connects here for all real-time events.

### Authentication

SASL PLAIN using the player's `eg1~` access token in the base64 payload.

### Supported stanzas

| Stanza | Handling |
| --- | --- |
| `<auth>` | SASL PLAIN — verifies access token |
| `<iq>` | Bind, session, ping, roster (empty) |
| `<presence>` | Broadcasts `available`/`unavailable` to accepted friends only |
| `<message>` | Direct chat + MUC groupchat |

### MUC (Multi-User Chat)

Party rooms use the `party-{partyId}@muc.prod.ol.epicgames.com` JID format. Used by the party system for all party events.

### Real-time events delivered via XMPP

- Gift received (`com.epicgames.gift.received`)
- Friend request sent/accepted/removed
- Party member joined/left/kicked/promoted
- Party updated/disbanded
- Invite sent

**Implementation:** `src/xmpp/XMPPServer.ts`

</details>

<details>
<summary><strong>Party system</strong></summary>

In-memory party state. All party events are broadcast via XMPP MUC.

### Endpoints

| Method | Path | Action |
| --- | --- | --- |
| `GET` | `/party/api/v1/Fortnite/user/:accountId` | Get user's current party and pending invites |
| `POST` | `/party/api/v1/Fortnite/parties` | Create a new party |
| `GET` | `/party/api/v1/Fortnite/parties/:partyId` | Get party info |
| `PATCH` | `/party/api/v1/Fortnite/parties/:partyId` | Update party meta (privacy, config) |
| `POST` | `/party/api/v1/Fortnite/parties/:partyId/members/:accountId/join` | Join party |
| `DELETE` | `/party/api/v1/Fortnite/parties/:partyId/members/:accountId` | Leave or kick member |
| `POST` | `/party/api/v1/Fortnite/parties/:partyId/invites` | Send invite |
| `DELETE` | `/party/api/v1/Fortnite/parties/:partyId/invites/:inviteId` | Decline or cancel invite |
| `POST` | `/party/api/v1/Fortnite/parties/:partyId/members/:accountId/promote` | Promote to leader |

**Implementation:** `src/routes/party.ts`

</details>

<details>
<summary><strong>Arena & hype system</strong></summary>

### Automatic hype tracking (DataRouter)

`POST /datarouter/api/v1/public/data` — the game client sends telemetry events here automatically. NeoServer listens and updates arena hype with no client-side changes required.

| Event | Hype change (configurable) |
| --- | --- |
| `Athena.ClientWonMatch` | +25 |
| `Combat.AthenaClientEngagement` (kill) | +7 |
| `Combat.ClientPlayerDeath` (division ≥ 4) | −20 |

All updates use atomic `$inc` with a 3-attempt optimistic lock.

### Manual hype API

`POST /api/v1/rewards/managehype/:username` — grants hype from an external service for reasons: `win`, `kill`, `top3`, `top7`, `top12`, `death`, `busfare`.

### Division thresholds

| Division | Hype range |
| --- | --- |
| 0 — Open League | 0 – 499 |
| 1 — Open League | 500 – 999 |
| 2 — Contender League | 1 000 – 1 999 |
| 3 — Contender League | 2 000 – 3 499 |
| 4 — Champion League | 3 500 – 5 499 |
| 5 — Champion League | 5 500 – 7 999 |
| 6 — Champion League | 8 000 – 11 999 |
| 7 — Champion League | 12 000+ |

### Leaderboard

`GET /api/v1/leaderboard/:type` — types: `arena`, `wins`, `kills`, `matches`. Single MongoDB `$facet` aggregation. Supports `?page=`, `?limit=`, `?search=`, and ETag caching.

`GET /api/onlineplayers` — live player count and names from the XMPP client registry.

**Implementation:** `src/routes/datarouter.ts`, `src/routes/leaderboard.ts`, `src/arena/`

</details>

<details>
<summary><strong>Shop & catalog</strong></summary>

### Static catalog

Serves `Responses/catalog.json` on `GET /fortnite/api/storefront/v2/catalog`. Sections: Featured and Daily.

### Auto-rotator

`npm run rotate-shop` runs `scripts/shopRotate.ts`:

1. Fetches all BR cosmetics from `fortnite-api.com`
2. Filters by rarity (excludes Common) and recent rotation history
3. Groups by item set — selects N featured panels (full sets) + N daily items (mixed types)
4. Writes the new `Responses/catalog.json`
5. Posts a Discord webhook embed with today's items and next rotation time (if `shop.webhookUrl` is set)

**Implementation:** `src/shop/ShopRotator.ts`, `scripts/shopRotate.ts`

</details>

<details>
<summary><strong>Tournaments</strong></summary>

| Endpoint | Description |
| --- | --- |
| `GET /api/v1/events/Fortnite/download/:accountId` | Full tournament event list |
| `GET /api/v1/players/Fortnite/:accountId/bulk/progress` | Player tournament progress |
| `POST /api/v1/players/Fortnite/bulk/team` | Team tournament lookup |
| `GET /api/v1/leaderboards/Fortnite/:eventId/:eventWindowId` | Tournament leaderboard |
| `GET /api/v1/events/Fortnite/tokens/:accountId` | Tournament eligibility tokens |

**Implementation:** `src/routes/tournaments.ts`

</details>

<details>
<summary><strong>Matchmaking</strong></summary>

### WebSocket matchmaker (port 3553)

```
Connecting → Queued → SessionAssignment → Play
```

1. Client connects and sends their account ID
2. Server places them in queue and broadcasts queue count
3. Creates ticket, match, and session identifiers
4. After countdown, assigns all queued players together
5. Disconnected clients are cleaned up

### HTTP endpoints

| Endpoint | Description |
| --- | --- |
| `GET /fortnite/api/game/v2/matchmakingservice/ticket/player/:accountId` | Matchmaking ticket (signed JWT) |
| `GET /fortnite/api/game/v2/matchmaking/account/:accountId/session/:sessionId` | Session info |
| `GET /fortnite/api/v1/matchmaking/session/:sessionId` | Session lookup |

**Implementation:** `src/routes/matchmaker.ts`

</details>

<details>
<summary><strong>Winterfest & seasonal events</strong></summary>

The timeline endpoint (`GET /fortnite/api/calendar/v1/timeline`) activates the Winterfest event channel for seasons 7, 11, and 19. Winterfest reward data is served from `Responses/winterfestRewards.json` (cached in memory after first read).

### Winterfest MCP commands

| Command | Description |
| --- | --- |
| `OpenGiftBox` | Opens a gift box and grants its loot to the appropriate profile |
| `ClaimLoginReward` | Claims the daily login reward for the current day |
| `FortRerollDailyQuests` | Replaces one daily quest with a random new one |

**Implementation:** `src/routes/timeline.ts`, `src/routes/mcp/commands/`

</details>

<details>
<summary><strong>Support a Creator (SAC)</strong></summary>

Players set their creator code in-game via `SetAffiliateName`. The code is validated against the `currentSACCode` field on User documents — if no player has that code, it returns an error.

### Endpoints

| Endpoint | Description |
| --- | --- |
| `POST .../client/SetAffiliateName` | Set or clear a SAC code in-game |
| `GET /affiliate/api/public/affiliates/slug/:slug` | Creator lookup (used by the game client) |

### Discord SAC commands

| Command | Description |
| --- | --- |
| `/sac <code>` | Sets the caller's linked game account SAC code |
| `/clearsac` | Clears the SAC code |

**Implementation:** `src/routes/mcp/commands/SetAffiliateName.ts`, `src/routes/account.ts`

</details>

<details>
<summary><strong>EOS (Epic Online Services)</strong></summary>

Required for Chapter 4+ game versions.

| Endpoint | Description |
| --- | --- |
| `POST /epic/oauth/v2/token` | EOS token (client_credentials, external_auth) |
| `GET /epic/oauth/v2/userInfo` | EOS user info |
| `GET /api/v1/asset/metadata/:artifactId` | Launcher asset metadata |
| `POST /api/epic/v2/keys` | EOS encryption keys |
| `GET /sdk/v1/sdk-related-assets` | SDK assets |
| `GET /waitingroom/api/waitingroom` | Queue status (returns null) |
| `GET /eulatracking/api/shared/agreements/fn` | EULA status |
| `POST /eulatracking/.../accept` | Accept EULA |

**Implementation:** `src/routes/eos.ts`

</details>

<details>
<summary><strong>Discord bot</strong></summary>

Discord.js v14 bot. Loads all commands automatically from `src/discord/commands/admin/` and `src/discord/commands/user/`. Registers slash commands on ready. Started conditionally — only when `discord.bot_token` is set in config.

### Admin commands

| Command | Description |
| --- | --- |
| `/register` | Create a new game account |
| `/ban` | Ban a player by username or account ID |
| `/unban` | Unban a player |
| `/vbucks` | Grant V-Bucks to a player |
| `/skin` | Add a skin to a player's athena profile |
| `/info` | Show account info (account ID, V-Bucks, ban status) |
| `/kick` | Kick a player from matchmaking |
| `/donate` | Donate V-Bucks to a specific Discord user |
| `/grant-full-locker` | Give full locker to a specific Discord user |
| `/grant-xp` | Grant XP to a player |
| `/set-hype` | Set arena hype for a player |
| `/rotate-shop` | Force a shop rotation manually |

### User commands

| Command | Description |
| --- | --- |
| `/exchange` | Generate an exchange code for the caller's linked account |
| `/giftvbucks` | Gift V-Bucks to another Discord user (30s cooldown) |
| `/account` | Show your linked game account info |
| `/link` | Link your Discord account to a game account |
| `/stats` | Show your player stats |
| `/sac` | Set your Support-a-Creator code |
| `/clearsac` | Clear your SAC code |
| `/claim-full-locker` | Claim full locker (role-gated, one-time, atomic) |
| `/claim-legacy-bundle` | Claim legacy cosmetic bundle (role-gated, one-time, atomic) |
| `/claim-booster-rewards` | Claim booster pack rewards (role-gated) |
| `/username` | Change your in-game username |
| `/delete-account` | Delete your game account (with confirmation) |
| `/arena-leaderboard` | Show the current arena leaderboard |

### Audit log

Every modifying action posts a structured embed to the configured `discord.auditWebhookUrl` (if set). Actions are color-coded: grant (green), ban (red), delete (orange), username change (blue).

**Implementation:** `src/discord/bot.ts`, `src/discord/commands/`

</details>

<details>
<summary><strong>Security</strong></summary>

### Rate limiting

| Limiter | Applied to | Limit |
| --- | --- | --- |
| `authLimiter` | `/account/api/oauth/token` | 10 req / 15 min per IP |
| `strictLimiter` | Register, delete, sensitive routes | 5 req / min per IP |
| `apiLimiter` | General API routes | 100 req / min per IP |
| `generalLimiter` | All `/fortnite/` routes | 300 req / min per IP |

### Security headers

Every response includes: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`, `Referrer-Policy: no-referrer`. `X-Powered-By` is removed.

### Additional protections

- `requestSizeGuard` — rejects request bodies larger than 2 MB with an Epic-format error
- `ipNormalize` — reads the real client IP from `cf-connecting-ip`, `x-real-ip`, or `x-forwarded-for`

**Implementation:** `src/middleware/security.ts`

</details>

<details>
<summary><strong>Database & collections</strong></summary>

| Collection | Stored data |
| --- | --- |
| `users` | Account ID, username, email, bcrypt password, Discord ID, ban status, SAC code |
| `profiles` | All MCP profiles embedded per account (athena, common_core, profile0, etc.) |
| `friends` | Accepted, incoming, outgoing, blocked lists + aliases |
| `arena` | Hype, division, wins, kills, matches played |

**Implementation:** `src/database/models/`

</details>

<details>
<summary><strong>Technology stack</strong></summary>

| Technology | Role |
| --- | --- |
| TypeScript 5 | Application language |
| Node.js 20+ | Runtime |
| Express 4 | HTTP API framework |
| MongoDB + Mongoose 8 | Database and data modeling |
| `ws` | WebSocket — XMPP and matchmaker |
| `xmlbuilder2` + `fast-xml-parser` | XMPP XML parsing and building |
| Discord.js v14 | Discord bot |
| `jsonwebtoken` 9 | JWT tokens (`eg1~` format) |
| `bcrypt` | Password hashing |
| `axios` | HTTP client (Discord OAuth2, shop rotator) |
| `zod` | Request body validation |
| `uuid` | Account ID and token generation |
| `express-rate-limit` | Rate limiting |
| `node-cron` | Shop rotation scheduling |

</details>

---

## Installation

### Requirements

- Node.js 20 or later
- MongoDB (local or Atlas)
- A Discord application (for OAuth2 and bot features)

### Steps

```bash
cd NeoServer-main
npm install
```

Copy and fill in the configuration:

```json
// Config/config.json
{
  "mongodb": { "database": "mongodb://127.0.0.1/NeoServer" },
  "discord": {
    "clientId": "",
    "clientSecret": "",
    "redirectUri": "http://localhost:3551/oauth2/callback",
    "bot_token": "",
    "auditWebhookUrl": "",
    "fullLockerRoleId": "",
    "legacyBundleRoleId": "",
    "legacyBundleItems": [],
    "adminRoleId": ""
  },
  "port": 3551,
  "xmppPort": 3552,
  "matchmakerPort": 3553,
  "ssl": { "enabled": false, "cert": "", "key": "" },
  "jwt": { "secret": "replace_with_a_long_random_string" },
  "currentSeason": 10,
  "arena": {
    "winHype": 25,
    "killHype": 7,
    "deathHype": 20
  },
  "shop": {
    "webhookUrl": "",
    "featuredSets": 3,
    "dailyItems": 6
  }
}
```

Start the server:

```bash
npm run dev
```

Rotate the shop manually:

```bash
npm run rotate-shop
```

### SSL

Set `ssl.enabled` to `true` and provide paths to your certificate and key files in `ssl.cert` and `ssl.key`.

---

## Project structure

```
NeoServer-main/
├── src/
│   ├── index.ts                  Entry point
│   ├── api/                      External service APIs (XP, hype, umbrella)
│   ├── arena/                    Arena hype engine
│   ├── config/                   Config loader and constants
│   ├── database/
│   │   └── models/               Mongoose models (User, Profile, Friends, Arena)
│   ├── discord/
│   │   ├── bot.ts                Bot bootstrap and command loader
│   │   ├── oauth2.ts             Discord OAuth2 web flow
│   │   └── commands/
│   │       ├── admin/            Admin slash commands (12)
│   │       └── user/             User slash commands (13)
│   ├── middleware/               Auth, error handler, security
│   ├── routes/
│   │   ├── mcp/
│   │   │   └── commands/         One file per MCP command (21 commands)
│   │   └── ...                   auth, friends, party, xmpp, arena, shop, etc.
│   ├── scripts/                  Standalone scripts (shop rotator)
│   ├── shop/                     Shop rotation logic
│   ├── structs/                  Shared utilities (error, functions, friend, profile, rewardUtils)
│   ├── tokenManager/             JWT creation and verification
│   └── xmpp/                     XMPP WebSocket server and handlers
├── CloudStorage/                 Default .ini files served to clients
├── Config/
│   └── config.json               Main configuration file
├── Responses/
│   ├── Athena/Data/Season{N}/    Per-season battlepass and XP data (S1–S12, S15)
│   ├── catalog.json              Current shop catalog
│   ├── contentpages.json         Lobby content pages
│   ├── winterfestRewards.json    Winterfest event rewards
│   └── ...
├── package.json
└── tsconfig.json
```

---

*NeoServer — TypeScript backend for a custom OG Fortnite server environment.*
