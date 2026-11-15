# Research: hosted in-app notification services

Ticket: #2 (map #1). Researched 2026-11-15 from vendor pricing pages, docs and the Novu GitHub repo. Pricing pages change often, so recheck the numbers before committing to a plan.

**Question.** What do hosted in-app notification services offer a small React Router v7 + SQLite app (in-app only, event-driven, real-time preferred, hundreds to low thousands of MAU), and what do they cost? Vendors compared: Knock, Novu (cloud and self-hosted OSS), MagicBell, Courier and SuprSend.

## Answer

**Shortlist: Knock and Novu Cloud.** Both free tiers include an in-app feed, preferences and batching/digest, with enough volume for our scale. Both also ship headless React hooks, so we can build our own UI and the service only does delivery.

- **Knock:** the most thorough docs (documented WebSocket real-time service, a headless feed and a preferences hook). The free tier has every feature we need. The catch: there is no documented EU data residency, and the first paid tier costs $250/mo.
- **Novu Cloud:** the free tier includes EU data residency, Pro is cheap ($30/mo), and the MIT-licensed core gives a real exit path. The catches: the free tier caps us at 5 workflows, and self-hosting is heavy.

The other three are out. **Courier** limits digest/batching to Enterprise. **SuprSend** puts preferences and batching on its $250/mo Business plan. **MagicBell**'s free tier allows only 1,000 deliveries a month and its pricing page doesn't say which features each plan includes.

Worth noting for the build-vs-buy ticket: at our scale every vendor still means sending user profiles and notification content to a third party. And none of them saves us from writing the event-triggering code on our server.

## Comparison

| | Knock | Novu Cloud | MagicBell | Courier | SuprSend |
|---|---|---|---|---|---|
| React inbox package | `@knocklabs/react` | `@novu/react` (React 18+) | `@magicbell/react` | `@trycourier/courier-react` v9 | `@suprsend/react` |
| Restyling | `--rnf-` CSS variables, replace all CSS, `renderItem`; headless hooks | Override styles, replace any element; headless hooks + `@novu/js` | CSS variables + class names, `ItemComponent`; no headless option documented | `CourierInboxTheme`, custom views, low-level APIs | Prebuilt or headless |
| Real-time | Documented WebSocket | Real-time unread count (transport not documented) | Not documented on the React page | `wss://realtime.courier.io` | Not documented on the React page |
| Preferences | Yes, free tier; React prefs component + `usePreferences` hook | Yes; Preferences component | `UserPreferences` component | Preference Center (plan not stated) | Business plan ($250) and up |
| Batching / digest | Batch function, free tier | Digest step (regular/timed, keyed) | Not on pricing page | **Enterprise only** | Business plan and up |
| Free tier | 10,000 messages/mo, no hard cap | 10,000 workflow runs/mo, 5 workflows, 3 seats | 1,000 deliveries/mo | 10,000 sends/mo | 10,000 notifications/mo, branded inbox |
| First paid tier | Starter $250/mo (50k messages, then $0.005) | Pro from $30/mo (30k runs, then $1.20/1k) | Startup $249/mo (50k, then $0.0025) | Business $0.005/send, every send billed | Essentials $100/mo (no prefs/digest) |
| EU residency | Not documented (GDPR statement only) | US + EU on Free | Not stated | EU (Ireland) on paid plans | Enterprise "custom data center region" |
| Self-host | No | MIT core (community edition) or Enterprise | No | No | Enterprise / BYOC beta |
| Client auth | Signed `userToken` (enhanced security mode; without it, anyone with the feed ID can read the feed) | HMAC `subscriberHash` | User JWT | Server-minted JWT | Signed JWT |

### Scale check

At 1,000-2,000 MAU, each getting about 5 in-app notifications a month, we'd use 5,000-10,000 messages a month. That fits the Knock, Novu, Courier and SuprSend free tiers, but not with much headroom. Knock and SuprSend say they don't hard-stop a free account that goes over; they get in touch about upgrading instead. Courier's Business plan bills every send, so 10k sends would cost about $50/mo. Our numbers are an estimate, not a measurement.

## Per-vendor notes

### Knock
- Pricing: Free is $0 with up to 10,000 messages; feeds, preferences and batching are included. Starter is $250/mo with 50,000 messages, then $0.005 each. Free and Starter keep logs for 30 days. Going over the Free limit gets you contacted about billing. [knock.app/pricing](https://knock.app/pricing)
- Feed: `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton` and `NotificationFeedPopover`. CSS variables prefixed `--rnf-`, a `colorMode` prop, and `renderItem`. [React feed docs](https://docs.knock.app/in-app-ui/react/feed)
- Headless: [custom feed UI with hooks](https://docs.knock.app/in-app-ui/react/headless/feed) and a [`usePreferences` hook](https://docs.knock.app/in-app-ui/react/headless/preferences). Both are listed in [docs.knock.app/llms.txt](https://docs.knock.app/llms.txt).
- Real-time: "Knock provides a real-time web socket connection for your users to subscribe to new messages being produced on an in-app channel." Socket behaviour can be overridden. [In-app API overview](https://docs.knock.app/in-app-ui/api-overview)
- Batching: windows can be fixed, dynamic or sliding, with an optional batch key and a cap of 90 days or 1,000 activities. [Batch function](https://docs.knock.app/designing-workflows/batch-function)
- **Uncertain:** the batch doc doesn't explicitly say batches render as a single feed item. Knock's own materials only make a GDPR statement, so data residency remains unconfirmed. A competitor (Courier) says Knock has no EU region, which is secondhand.

### Novu
- Pricing: Free is $0 with 10,000 runs, 5 workflows, 24h activity-log retention and US + EU residency. Pro starts at $30/mo with 30k runs, then $1.20 per 1k. Team starts at $250/mo. The pricing page lists self-hosting under Enterprise. [novu.co/pricing](https://novu.co/pricing)
- **Uncertain:** the pricing table didn't extract cleanly, so which inbox, preference and digest features each plan includes couldn't be confirmed beyond the plan bullets.
- Inbox: `@novu/react` `<Inbox />` is built from Bell, Notifications, InboxContent and Preferences parts. You can "override styles, replace every UI elements and icons", or go headless with hooks or `@novu/js`. Updating a subscriber needs an HMAC `subscriberHash`. [Inbox overview](https://docs.novu.co/platform/inbox/overview)
- Digest: either `regular` (time window, optional look-back) or `timed` (cron), grouped by a key such as `payload.projectId`. [Digest step reference](https://docs.novu.co/.mintlify/skills/dashboard-workflows/references/digest-step.md)
- Self-host (community edition): the repo has `LICENSE-MIT` plus `LICENSE-ENTERPRISE` / `EE-PACKAGES-LICENSE` (open core) ([github.com/novuhq/novu](https://github.com/novuhq/novu)). It needs API, Worker, WS and Dashboard services plus MongoDB, Redis (with AOF) and object storage; the minimum is one VM with 4 vCPU / 8 GB plus MongoDB and Redis ([self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview)). Compared with Cloud it has no snooze, can't remove Novu branding, and Novu describes it as "for testing and experimenting" ([self-hosted vs cloud](https://docs.novu.co/community/self-hosted-and-novu-cloud)). That is a lot of infrastructure next to an app that runs on a single SQLite file.

### MagicBell
- Pricing: Builder is $0 for 1,000 deliveries/mo. Startup is $249/mo for 50,000, then $0.0025 per delivery. The pricing page doesn't say which plans include the inbox, real-time, preferences, digest or regions. [magicbell.com/pricing](https://www.magicbell.com/pricing)
- `@magicbell/react`: needs a server-generated User JWT. It provides `Inbox`, `FloatingInbox` and a `UserPreferences` component, styled with CSS variables (`--magicbell-*`). The React page doesn't document headless hooks or the real-time transport. [React docs](https://www.magicbell.com/docs/libraries/magicbell-react)

### Courier
- Pricing: Developer is free for 10,000 sends/mo. Business is $0.005 per send, with every send billed. **Digest & batching is listed only under Enterprise.** EU data residency comes with Business. [courier.com/pricing](https://www.courier.com/pricing)
- Inbox: `@trycourier/courier-react` v9. The backend mints a JWT and the client calls `signIn({ userId, jwt })`. Theming goes through `CourierInboxTheme`, custom views, or low-level APIs. [Inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)
- Regions: US and a separate EU (Ireland) datacenter with its own keys, including `wss://realtime.eu.courier.io`, available on all paid plans. [EU datacenter](https://www.courier.com/docs/platform/workspaces/eu-datacenter)

### SuprSend
- Pricing: Free is 10k notifications with a branded inbox. Essentials is $100/mo for 50k, then $2 per 1k. **Business is $250/mo and is the first tier with preferences and batching/digest.** Self-hosting, BYOC and a custom region are Enterprise only. Going over the Free limit isn't hard-stopped. [suprsend.com/pricing](https://www.suprsend.com/pricing)
- **Uncertain:** the comparison table's checkmarks didn't extract, so the gating above comes from the plan cards.
- React: `@suprsend/react` (the older `@suprsend/react-inbox` is end-of-support). It offers a prebuilt UI or headless mode, and auth moved from HMAC to signed JWT. [React inbox docs](https://docs.suprsend.com/docs/inbox-react)

## Data residency and lock-in (all vendors)

What leaves our database: each recipient's user ID (and any profile fields we sync), the payload of every triggered event, the rendered notification content, read/seen/archived state, and preferences. The vendor becomes the system of record for the inbox. Our SQLite database would no longer know what a user has been notified about unless we mirror it ourselves.

Migrating off means:
- re-creating workflows, templates and batch rules, which live in the vendor's dashboard or DSL;
- exporting notification history and preferences through the vendor's API, if it allows that;
- replacing the client SDK and auth-token minting.

Two things reduce the lock-in. Using headless hooks keeps our UI ours. With Novu, the MIT core is a theoretical fallback, though the infrastructure is heavy.

## Integration effort (what our server must do)

All five vendors follow the same pattern:
1. **Identify users:** upsert recipients/subscribers with an ID (and a name for avatars).
2. **Trigger:** when a domain event happens (lesson published, comment reply, and so on), call `trigger(workflowKey, { recipients, actor, data })` from a React Router action or server module.
3. **Mint a client token:** sign a per-user JWT or HMAC hash (Knock `userToken`, Novu `subscriberHash`, Courier/MagicBell/SuprSend JWT) and pass it to the browser through a loader.
4. **Mount the provider and feed component** in the root layout. Real-time arrives over the vendor's socket, so we don't host a WebSocket server.

The secret-key signing step is mandatory in production. Without Knock's enhanced security mode, for example, anyone who has a user's feed ID can read their feed.

## Open questions
- Does Knock offer EU residency on any plan? Ask Knock directly.
- Exact Novu per-plan feature gating, since the pricing table didn't extract.
- Whether the free tiers' "no hard stop" policy is something we can count on.
- If the deciding goal is learning and understanding the system, a hosted service hides the most interesting parts (fan-out, real-time transport, digest logic). That trade-off belongs to the build-vs-buy decision on the map.
