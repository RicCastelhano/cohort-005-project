# Hosted Notification providers for Cadence

Research for issue #2 (child of map #1). Question: which hosted in-app notification providers fit Cadence (React Router 7 + Node, SQLite/Drizzle)?

Researched 2026-11-18 against vendor docs and pricing pages. Pricing and package names change often, so re-check them before committing to anything. Items marked **(?)** were not confirmed on a primary page.

This document does not pick a provider. That decision belongs to a separate grilling ticket.

## Fit with Cadence, in brief

All five work the same way. Cadence's Node server triggers a workflow or notification through a server SDK, using a secret key. The browser renders a React inbox that talks directly to the vendor, using a public key plus a per-user token. Cadence would mint that token on the server for the session user. None of them need anything from SQLite/Drizzle except a stable user ID to use as the recipient ID. All of them keep notification state (read, archived and so on) on the vendor side, so Cadence would not store it.

## Comparison table

| | **Knock** | **Novu Cloud** | **MagicBell** | **Courier** | **SuprSend** |
|---|---|---|---|---|---|
| **React inbox package** | `@knocklabs/react`: `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton` [k1] | `@novu/react`: `Inbox`, plus composable `Bell`, `Notifications`, `InboxContent`, `Preferences`. Needs React 18 or later; supports 19 [n1] | `@magicbell/react`: `ContextProvider`, `Inbox`, `FloatingInbox`, `InlineNotification`, `ToastContainer`, `UserPreferences` [m1] | `@trycourier/courier-react` (v9): `<CourierInbox />`, Toasts, Preferences [c1] | `@suprsend/react` (`@suprsend/react-inbox` is end-of-life): `Inbox` popover, plus full-screen and side-sheet feeds [s1][s2] |
| **Customizability** | `renderItem`, `renderHeader`, `--rnf-*` CSS variables, replace the whole stylesheet, `colorMode`, headless hooks [k2] | Override styles, replace sub-components and icons, change the layout; headless hooks, or framework-agnostic `@novu/js` [n1][n5] | CSS variables (`--magicbell-*`), class names, slot components (`ItemComponent`, `ButtonComponent`, ...), `unstyled` [m1] | `CourierInboxTheme`, custom views, or a fully custom UI on the low-level APIs [c1] | `theme`, `themeType`, replace bell, badge, card, header and empty state, CSS, headless hooks, multiple tabs through `stores` [s2] |
| **Free tier** | 10k messages/mo; feeds, Feed API and preferences included; Knock branding stays. Next tier is Starter at $250/mo [k3] | 10k workflow runs/mo, 5 workflows, 2 environments (Dev and Prod), 3 seats, **24h activity retention**. Next tier is Pro from $30/mo [n2] | **1k deliveries/mo**, counted per channel; unlimited users; 1 project. Next tier is Startup at $249/mo [m2] | 10k sends/mo. The pricing page does not say whether Inbox or Preferences are included (?). Next tier is pay-as-you-go at $0.005/send, billed from the first send [c2] | 10k notifications/mo; inbox included with SuprSend branding; **preferences not included**; no batching or digest; 30-day logs. Next tier is Essentials at $100/mo [s3] |
| **Node SDK** | `@knocklabs/node`, which includes `signUserToken` [k4] | `@novu/api`: `trigger` with a `secretKey` [n3] | `magicbell-js` (`ProjectClient`). Package naming is inconsistent across docs: `magicbell`, `@magicbell/core`, `magicbell-js` (?) [m3] | Courier Node SDK, e.g. `client.auth.issueToken(...)` [c3] | `@suprsend/node-sdk` [s4] |
| **Recipient ID and client auth** | Recipient ID is your user `id`. By default the public key alone can "act as any user". In **enhanced security mode**, the server signs an **RS256 JWT** (`sub` = user ID, default expiry 1h) with a signing key, passes it as `userToken` to `KnockProvider`, and refreshes it via `onUserTokenExpiring`. The docs say to enable this before production [k4] | Recipient is a `subscriberId` plus a public `applicationIdentifier`. With **HMAC** turned on, the server computes `HMAC-SHA256(secretKey, subscriberId)` and passes it as `subscriberHash`. HMAC is optional but strongly recommended; once on, the Inbox will not load without the hash [n4] | **User JWT, HS256**, signed by your server with the project Secret Key. Claims: `user_external_id` and/or `user_email`, plus `api_key`. Passed as the required `token` prop. The doc example uses a 1-year expiry [m4] | The server calls `POST /auth/issue-token` with the API key and scopes such as `user_id:<id> inbox:read:messages inbox:write:events` (plus `read:preferences`/`write:preferences`). The client calls `signIn({ userId, jwt })`. Without `expires_in` the token never expires, and the SDK does **not** refresh it [c3] | Recipient is a `distinctId`. In **enhanced security mode**, `SuprSendProvider` takes `publicApiKey`, `distinctId` and `userToken`, an **ES256 JWT** signed by your server. HMAC was replaced by JWT [s1][s5] |
| **Read / unread / archive** | unseen, seen, read, plus archived (`seen_at`, `read_at`, `archived_at`). The popover marks items seen on open by default [k5] | read, unread, archive (bulk `archiveAll` and `archiveAllRead`), plus **snooze** (cloud only, at least 3 min ahead) [n5] | read, unread, archive, delete. "Seen" is not mentioned on the in-app page (?) [m5] | unread counts, read, opened, archive, synced across devices [c1] | seen, read, unread, archive, plus **pinned** notifications (`is_pinned`) [s5] |
| **Real-time** | WebSocket. Optional cross-tab sync through BroadcastChannel (experimental). Sockets in inactive tabs are managed automatically [k2][k5] | Yes, through a socket (`socketUrl`, e.g. `wss://eu.socket.novu.co`) [n4] | Yes, a "persistent connection"; the transport is not named [m5] | Yes, real-time sync across devices; the transport is not named on the overview page [c1] | Yes, through a socket (`host.socketHost`), plus a connection-status indicator [s2] |
| **Preferences** | Included on free; preferences management and API [k3]. A pre-built React preference-center component was not confirmed (?) | Built into the Inbox (`Preferences`) per workflow and channel [n1] | `UserPreferences` component, per category and channel [m1][m5] | Preferences component and preference center; free-tier inclusion unclear (?) [c1][c2] | Supported, but **excluded from the free tier** [s3] |
| **Dev / test / sandbox** | Development and production environments plus custom ones (e.g. staging). Content is committed and promoted between environments, and data is isolated per environment. Whether development counts toward billing is not stated (?) [k6] | Development and production environments with separate keys; you publish from Dev to Prod. Custom environments are Team or Enterprise only [n3] | 1 project on free. No documented sandbox or test mode for in-app (?) [m2][m5] | Test and production environments, each with draft and published keys (`dk_`/`pk_`). **Mock keys** simulate sends, but sources disagree on which plan includes them (?) [c4] | Sandbox, Staging and Production workspaces with separate credentials. **Test Mode** works in Staging and custom workspaces [s6] |

## Trade-offs worth noting (not a recommendation)

- **Free-tier headroom:** Knock, Novu, Courier and SuprSend all give about 10k/mo. MagicBell gives only 1k deliveries, counted per channel. Novu's 24h activity retention makes debugging harder on free.
- **Cost of the first paid step:** Novu Pro starts at $30/mo, SuprSend at $100/mo, and Knock and MagicBell at about $250/mo. Courier charges per send from the first message once you leave free.
- **Client auth strength:** Knock, MagicBell, Courier and SuprSend all use server-minted JWTs. Novu uses a static HMAC of the subscriber ID, which never expires but is simple. Knock and SuprSend let the public key alone act as any user until you switch on enhanced security, so turning it on is mandatory work. Courier tokens need manual refresh. MagicBell's example tokens last a year.
- **Inbox polish:** Every provider ships a drop-in inbox, a headless option and real-time updates. Novu has snooze, SuprSend has pinning and tabs, and Knock has the richest seen/read/archived model.
- **Preferences:** On free, preferences are confirmed for Knock and Novu, unclear for Courier, and absent for SuprSend.
- **Lock-in:** Novu is also open source and self-hostable (outside this question's scope). The others are hosted only.

## Sources

- [k1] https://docs.knock.app/in-app-ui/react/overview
- [k2] https://docs.knock.app/in-app-ui/react/feed
- [k3] https://knock.app/pricing
- [k4] https://docs.knock.app/in-app-ui/security-and-authentication
- [k5] https://docs.knock.app/in-app-ui/feeds/overview, https://docs.knock.app/api-reference/messages
- [k6] https://docs.knock.app/concepts/environments
- [n1] https://docs.novu.co/platform/inbox/overview
- [n2] https://novu.co/pricing
- [n3] https://docs.novu.co/platform/developer/environments, https://docs.novu.co/platform/sdks/server/typescript
- [n4] https://docs.novu.co/platform/inbox/prepare-for-production
- [n5] https://docs.novu.co/platform/sdks/react/hooks/use-notifications, https://docs.novu.co/platform/inbox/features/snooze, https://docs.novu.co/platform/inbox/headless-mode
- [m1] https://www.magicbell.com/docs/libraries/magicbell-react
- [m2] https://www.magicbell.com/pricing
- [m3] https://www.magicbell.com/docs/libraries/magicbell-js
- [m4] https://www.magicbell.com/docs/api/authentication/user
- [m5] https://www.magicbell.com/docs/channels/in_app
- [c1] https://www.courier.com/docs/platform/inbox/inbox-overview
- [c2] https://www.courier.com/pricing
- [c3] https://www.courier.com/docs/platform/inbox/authentication
- [c4] https://www.courier.com/docs/workspaces/environments, https://www.courier.com/docs/workspaces/overview
- [s1] https://docs.suprsend.com/docs/inbox-react, https://docs.suprsend.com/docs/react-sdk
- [s2] https://docs.suprsend.com/docs/react-in-app-feed
- [s3] https://www.suprsend.com/pricing
- [s4] https://docs.suprsend.com/docs/integrate-node-sdk
- [s5] https://docs.suprsend.com/docs/react-sdk, https://www.npmjs.com/package/@suprsend/react-core
- [s6] https://docs.suprsend.com/docs/workspaces
