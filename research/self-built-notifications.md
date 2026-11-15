# Research: self-built in-app notifications in Cadence's stack

Ticket: #3 (map #1). Scope: in-app only, event-driven, leaning real-time, optimised for learning and understandability.

## TL;DR

- **Data model:** one `notifications` table holding **one row per recipient**: `recipientId`, `type`, `actorId`, a polymorphic target (`targetType` + `targetId`), a link `url`, `groupKey`, `readAt` (null means unread), and `createdAt`. Index on `(recipientId, readAt, createdAt)`. The unread badge is one `COUNT(*)` on that index.
- **Creation:** write the notification **inline, inside the same Drizzle transaction** as the action that caused it, through one `notify(tx, …)` function in a `notifications.server.ts` module. An outbox would only pay off once a second consumer appears (email, webhooks).
- **Delivery:** **Server-Sent Events (SSE) from a React Router resource route**, with an in-process `EventEmitter` as the "something changed" signal and `useRevalidator()` on the client to re-run loaders. The stream carries only a "go refetch" nudge, never the payload. Use polling or revalidation on focus as the fallback, which you get almost for free.
- **Key constraint:** the in-memory emitter only works while Cadence runs as **one Node process**. That is true today (`react-router-serve`, no cluster, SQLite file on local disk). If the app ever runs more than one process, the emitter must be replaced (for example by polling SQLite), but the table and the `notify()` seam stay the same.
- **WebSockets are not recommended.** `react-router-serve` gives the app no access to the HTTP server's `upgrade` event, so WebSockets would mean replacing it with a custom Express server, all for a client-to-server channel that notifications don't need.

## 1. What the repo constrains

| Fact | Evidence | Consequence |
| --- | --- | --- |
| Prod server is `react-router-serve ./build/server/index.js` | `package.json` `start` script | One Node process, Express under the hood. There is no custom server file. |
| `react-router-serve` 8.3.0 builds an Express app with `compression()`, static, `morgan`, then `createRequestHandler` | `node_modules/@react-router/serve/dist/cli.js` lines 111-127 | No hook to reach the `http.Server` (needed for WebSocket `upgrade`). Compression sits in front of every response. |
| The Express adapter calls `res.flushHeaders()` for `Content-Type: text/event-stream` and calls `res.flush()` after each chunk it writes | `node_modules/@react-router/express/dist/index.js` line 63, `node_modules/@react-router/node/dist/index.js` lines 107-121 | SSE works through `react-router-serve`, compression included, with no custom server. |
| The adapter aborts `request.signal` when the response closes | `@react-router/express/dist/index.js` lines 41-48 | A loader can clean up its subscription on `request.signal`'s `abort` event when the tab closes. |
| SQLite through `better-sqlite3` (synchronous), opened lazily, `journal_mode = WAL`, `foreign_keys = ON`, file `data.db` | `app/db/index.ts` | Single writer and concurrent readers. Synchronous transactions, so `notify()` can sit inside one with no async gaps. |
| No background or scheduled work anywhere in `app/`: no cron, queue, worker or server-side `setInterval` (the only `setInterval` is client-side in `youtube-player.tsx`) | `grep` over `app/` | Nothing to piggyback on, and so far no need for a worker. Inline creation fits the current shape. |
| No deployment config in the repo (no Dockerfile for the app, fly/render/railway files, or Procfile; `.sandcastle/Dockerfile` is agent tooling) | `git ls-files` | Hosting is undecided. The "single process" assumption is safe today but must be written down. |
| Auth is a cookie session (`createCookieSessionStorage`, `getCurrentUserId(request)`) | `app/lib/session.ts` | An SSE `EventSource` request is a same-origin GET and sends the cookie, so the stream route can authenticate the same way as any loader. |
| A bell with an unread count is already on the backlog, for "announcements, quiz results, coupon activity" | `_internal/feature-ideas.md` item 23 | Confirms the shape: a badge plus a list. |
| Candidate trigger sources already exist as tables: `comments` (with `parentId` replies), `enrollments`, `purchases`, `refunds`, `quizAttempts`, `teamMembers`, `coupons` | `app/db/schema.ts` | `actorId` and `targetType`/`targetId` cover all of them without per-type columns. |

## 2. The common data model

Self-built systems converge on the same few fields. They come from the activity-stream "actor / verb / object / target" vocabulary ([W3C Activity Streams 2.0](https://www.w3.org/TR/activitystreams-core/)), flattened for a relational table:

```ts
// app/db/schema.ts (sketch, not applied)
export const notifications = sqliteTable(
  "notifications",
  {
    id: integer("id").primaryKey({ autoIncrement: true }),
    recipientId: integer("recipient_id").notNull().references(() => users.id),
    type: text("type").notNull(),            // enum: "comment_reply" | "quiz_graded" | "course_announcement" | ...
    actorId: integer("actor_id").references(() => users.id), // null for system events
    targetType: text("target_type"),         // "lesson" | "course" | "comment" | ...
    targetId: integer("target_id"),
    url: text("url").notNull(),              // where the click goes; computed at write time
    groupKey: text("group_key"),             // e.g. "comment_reply:lesson:42" for collapsing
    data: text("data", { mode: "json" }),    // small per-type payload (snippet, score)
    readAt: text("read_at"),                 // null = unread
    createdAt: text("created_at").notNull().$defaultFn(() => new Date().toISOString()),
  },
  (t) => [index("notifications_inbox_idx").on(t.recipientId, t.readAt, t.createdAt)]
);
```

Design notes:

- **One row per recipient (fan-out on write)** keeps read state trivial: `readAt` lives on the row. The alternative is one `notification` row plus a `notification_recipients` join row holding read state. It saves space when one event reaches many people, but every read query becomes a join. At Cadence's scale (course announcements reach at most one course's enrolments), one row per recipient is simpler and fast enough.
- **`readAt` as a nullable timestamp**, not a boolean, matches the repo's existing conventions (`editedAt`, `deletedAt` on `comments`), and it records *when* the user read the notification for free.
- **Polymorphic target** (`targetType` + `targetId`) cannot have a foreign key. Store a precomputed `url` and treat the target as informational, so a deleted target degrades to a dead link instead of a broken join.
- **`groupKey`** enables "Ana and 3 others replied on Lesson 4" by grouping at read time, or upserting at write time (bump `createdAt`, clear `readAt`, increment a count in `data`). Grouping at read time is easier to understand, so start there.
- **Timestamps as ISO text** match every other table in `schema.ts`, and ISO strings sort correctly as text.
- Schema changes must follow the repo's `database-migrations` skill (generate the migration, update the seed, reseed).

## 3. Delivery options

### Option A: polling and revalidation (baseline)

React Router already revalidates loader data after every action. [`useRevalidator()`](https://reactrouter.com/api/hooks/useRevalidator) exists "for reasons outside of normal data mutations like Window focus or polling on an interval". Put the unread count in the `layout.app.tsx` loader, and it refreshes on every navigation and form submit with no extra code. Add `revalidate()` on `visibilitychange`/`focus` plus an interval for near-real-time.

- Server needs: nothing.
- SQLite: one indexed `COUNT(*)` per poll per tab. Cheap in WAL mode, where "readers do not block writers and a writer does not block readers" ([sqlite.org/wal.html](https://www.sqlite.org/wal.html)).
- Multi-process: works unchanged.
- Cost: latency equals the interval, and each poll re-runs *every* loader on the page unless `shouldRevalidate` narrows it.

### Option B: Server-Sent Events (recommended)

A resource route (a route module with only a `loader`) returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream", ... } })`. React Router resource routes may return any `Response`, including streamed bodies (see the [file-uploads how-to](https://reactrouter.com/how-to/file-uploads)). The browser consumes the stream with [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource).

Shape that fits this stack:

1. `app/lib/notifications.server.ts` owns a module-level `EventEmitter`. `notify()` inserts rows, then **after the transaction commits** emits `user:<recipientId>`.
2. `routes/api.notifications.stream.ts` loader: authenticate with `getCurrentUserId(request)`, subscribe to `user:<id>`, enqueue `data: changed\n\n` on each event, enqueue a `: ping` comment every ~15 s, and unsubscribe on `request.signal`'s `abort` event.
3. The client opens one `EventSource` in the app layout. On a message it calls `revalidator.revalidate()`, so the badge and list come from the loader exactly as in Option A.

Why it fits:

- It is plain HTTP and runs inside the existing `react-router-serve` server. The Express adapter flushes headers and each chunk for `text/event-stream` (repo evidence in section 1), so the default `compression()` middleware does not buffer events.
- Reconnection is built in. "By default, if the connection between the client and server closes, the connection is restarted", and the `retry:` field sets the delay ([MDN, Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)). On reconnect the browser sends a `Last-Event-ID` header if events carried `id:` ([WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html)). Because the stream only says "go refetch", missed events don't matter, and you can ignore `Last-Event-ID` entirely.
- Keepalive is a comment line. "Legacy proxy servers … may drop HTTP connections after a short timeout", so "authors can include a comment line (one starting with a ':' character) every 15 seconds or so" (WHATWG §9.2.7).

Constraints:

- **The single process is the key constraint.** The emitter is in-memory, so a `notify()` in process A never reaches a stream held by process B. SQLite's `sqlite3_update_hook` is registered "with the database connection identified by the first argument" ([sqlite.org](https://www.sqlite.org/c3ref/update_hook.html)), so it cannot signal across processes either. If Cadence ever scales out, the stream loader switches to polling `MAX(id)` for its user every few seconds (a per-process poll of SQLite instead of per-tab), or a broker is added. The client and the table stay the same.
- **HTTP/1.1 connection limit.** "When not used over HTTP/2, SSE suffers from a limitation to the maximum number of open connections … per browser … (6)", across all tabs of the domain (MDN). `react-router-serve` speaks HTTP/1.1, so a user with 6+ Cadence tabs open will stall other requests unless a proxy in front terminates HTTP/2. Mitigations: close the `EventSource` when `document.hidden` and rely on revalidation on focus, or share one connection across tabs (a `BroadcastChannel` plus a leader tab). Closing on hidden is the simplest and also cuts server load.
- **Dev server module reloads.** In `react-router dev`, editing `notifications.server.ts` re-evaluates the module and creates a second emitter while old streams hold the first. Stash the emitter on `globalThis` (the same trick commonly used for DB clients) so it survives HMR. Check this in dev before relying on it.
- Each open tab holds one socket and a few closures: fine for hundreds of users, and not something to worry about at this stage.

### Option C: WebSockets

[WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) are bidirectional, which notifications don't need: the client only ever *receives* a nudge, and marking a notification read is an ordinary `<Form>` or `useFetcher` action. WebSockets also need the Node `http.Server`'s `upgrade` event, which `react-router-serve` doesn't expose (section 1). Adopting them means migrating to the custom Express server (`server.js` + `server/app.ts` and a Vite config change, per [React Router adapter docs: "Migrating from the React Router App Server"](https://reactrouter.com/api/other-api/adapter)), adding a `ws` dependency, and authenticating the upgrade request by parsing the session cookie by hand. They share SSE's single-process limit. All of that cost is paid only for a direction of traffic notifications don't use.

### Comparison

| | Polling / revalidate | SSE | WebSockets |
| --- | --- | --- | --- |
| Latency | interval | ~instant | ~instant |
| Server change | none | one resource route plus an emitter | replace `react-router-serve` with a custom server, plus `ws` |
| Works with multiple processes | yes | only with a cross-process signal | only with a cross-process signal |
| Auth | existing loader | existing loader (cookie on GET) | manual cookie parse on upgrade |
| Reconnect | n/a | built into `EventSource` | hand-rolled |
| Learning value | low | high, and small enough to read end to end | high, but mostly plumbing |

## 4. Where notifications are created

**Inline (recommended now).** The action that causes the event calls `notify(tx, { recipientIds, type, actorId, target, url, groupKey })` inside its own `db.transaction`. better-sqlite3 transactions are synchronous ([better-sqlite3 API: `.transaction()`](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md#transactionfunction---function)), so the comment and its notification commit or roll back together. Emit the SSE signal after the commit returns, never inside the transaction. Pros: atomic, no infrastructure, easy to trace in a debugger. Cons: the action pays the fan-out cost on the request path, and every new trigger means editing that action.

**Outbox / event table.** The action writes a domain event row (`events(type, payload, createdAt, processedAt)`) in the same transaction. A consumer turns events into notifications (and later emails or webhooks). Pros: decoupled, one event feeds many consumers, retryable. Cons: Cadence has **no background worker** (section 1), so you'd have to add one (an in-process `setInterval` or a separate script). Notifications would also arrive slightly later, and "why didn't I get notified?" now spans two places.

**Recommendation.** Go inline, but behind a single deep `notify()` function so every call site looks the same. If email notifications (backlog item 29) or webhooks (item 24) arrive, `notify()` becomes the place that also writes to an outbox, and the call sites stay unchanged.

## 5. Pitfalls

- **Fan-out volume.** An instructor announcement to a 2,000-student course is 2,000 inserts. In SQLite that is fine *if* it is one transaction with a multi-row `insert().values([...])`. Outside a transaction, each insert is its own commit (see [SQLite FAQ #19](https://www.sqlite.org/faq.html#q19) on why many separate INSERTs are slow). Remember there is "only one writer at a time" (sqlite.org/wal.html), so a huge fan-out blocks other writes for its duration. If that ever hurts, chunk the inserts or move announcements to fan-out on read (one announcement row, joined against enrolments).
- **N+1 unread counts.** Compute the badge with one `SELECT COUNT(*) … WHERE recipient_id = ? AND read_at IS NULL` in the layout loader, served by the composite index, never by loading rows and counting in JS. For the list, join `users` for actor names in the same query instead of fetching the actor per row. The existing layout loader already makes several sequential queries, so don't add a per-item one.
- **Stale badges across tabs.** Marking read in tab A doesn't update tab B. Two easy fixes: (1) `notify` and mark-read both emit on the user's channel, so every tab's SSE nudges a revalidate; (2) revalidate on `visibilitychange`. Use both, because (2) also covers tabs whose stream was closed while hidden.
- **Notifying the actor.** Filter `actorId` out of `recipientIds` inside `notify()`, so a user never sees "you replied to your own comment".
- **Soft-deleted targets.** `comments.deletedAt` exists. Decide whether deleting a comment also deletes its notifications, or leaves them pointing at a "deleted" placeholder. Precomputing `url` keeps the latter safe.
- **Unbounded growth.** Add a retention rule (for example, delete read notifications older than 90 days) eventually. With no scheduler, run it opportunistically in `notify()` or as a script.

## Sources

- Repo: `package.json`, `app/db/index.ts`, `app/db/schema.ts`, `app/lib/session.ts`, `app/routes.ts`, `app/routes/layout.app.tsx`, `_internal/feature-ideas.md`
- Installed packages (8.3.0): `@react-router/serve/dist/cli.js`, `@react-router/express/dist/index.js`, `@react-router/node/dist/index.js`; `compression/index.js` and its README (SSE `res.flush()` requirement); `compressible/index.js` (`/^text\//` marks `text/event-stream` compressible)
- React Router docs: [useRevalidator](https://reactrouter.com/api/hooks/useRevalidator), [Server adapters / custom Express server](https://reactrouter.com/api/other-api/adapter), [File uploads (resource route returning a streamed `Response`)](https://reactrouter.com/how-to/file-uploads)
- MDN: [Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events), [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource), [WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- WHATWG HTML Living Standard [§9.2 Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- SQLite: [Write-Ahead Logging](https://www.sqlite.org/wal.html), [sqlite3_update_hook](https://www.sqlite.org/c3ref/update_hook.html), [FAQ #19](https://www.sqlite.org/faq.html#q19)
- [better-sqlite3 API docs](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md)
- [W3C Activity Streams 2.0](https://www.w3.org/TR/activitystreams-core/)
