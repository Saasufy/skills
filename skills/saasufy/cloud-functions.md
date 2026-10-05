# Saasufy Cloud Functions

A `CloudFunction` is a piece of JavaScript which Saasufy runs inside your service, in a sandboxed QuickJS VM, and exposes over HTTP at `{SERVICE_URL}/functions/{name}`.

## Use Cloud Functions as a Last Resort

Prefer a declarative Saasufy feature whenever one can do the job. Views, aggregations and access-control rules are built so that unscalable patterns are hard to express; a cloud function has no such guardrails. It does run in parallel across the workers and cores of the service, but parallelism cannot undo an inefficiency written into the code — an unindexed scan, an N+1 loop of per-record reads, or a function polled in place of a subscription will only get slower as the data grows.

Check first whether the job is already covered:

- Filtering, sorting, searching, paginating → a `ModelView` backed by an index. See [views-and-indexing.md](views-and-indexing.md) and [search-filtering-querying.md](search-filtering-querying.md).
- Totals, averages, high-score tables, history tables, summary fields joined onto related records → an `Aggregation`, which is incremental and shards across workers. See [data-aggregation-pipelines.md](data-aggregation-pipelines.md).
- Who may read or write what → access-control rules. See [access-control.md](access-control.md).
- Login, signup, user profiles → [authentication.md](authentication.md) and [account-table.md](account-table.md).
- Field validation → `ModelField` constraints. See [schema-management.md](schema-management.md).
- Reacting to data changes → realtime components bound to a view or record, not a function on a timer.

If none of those seem to fit, it is usually the shape of the data rather than a missing feature: store the answer instead of computing it per request. A function which counts a collection on every call is an `Aggregation`; one which loops over related records is an aggregation writing a summary field onto the related record; one which filters in JavaScript usually just needs a precomputed field and a compound index so that a view can express the filter.

Reach for a cloud function when there is genuinely no declarative equivalent — a third-party webhook, an external API call, work needing a secret which must not reach the browser, an imperative multi-step operation — and then keep it short and bound the work it does per invocation.

Cloud functions are managed through the Admin HTTP API (or the `Cloud Functions` page of the dashboard) and invoked on the deployed service URL.

## Authentication

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
SAASUFY_SERVICE_URL=$(cat .saasufy-service-url)
```

The Admin API base URL is `https://saasufy.com/api/`. See [schema-management.md](schema-management.md).

## CloudFunction Management

### List cloud functions

Available views:
- **`accountAlphabeticalView`** — Ordered by name ascending. Params: `accountId`.
- **`accountSearchView`** — Search by name. Params: `accountId`, `nameText`.
- **`accountNameView`** — Find by exact name. Params: `accountId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/CloudFunction?view=accountAlphabeticalView'
```

### Get a cloud function by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/CloudFunction/{FUNCTION_ID}'
```

### Create a cloud function

Required fields:
- `name` (string, alphanumeric, 1-50 chars) — becomes the last segment of the invocation URL and must be unique within the account. **It cannot be changed afterwards**; the record ID is derived from it, so creating a second function with the same name fails.

Optional fields:
- `code` (string, max 100000 chars) — the **body** of an async function (see below).
- `timeout` (integer, 50-10000 ms) — null means the platform default of 10000.
- `memoryLimit` (integer, 1048576-33554432 bytes) — null means the platform default of 33554432. A function may only narrow these limits, never raise them.
- `isDisabled` (boolean) — leaves the function unreachable without deleting its code.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/CloudFunction' \
  -d '{"name": "greet", "code": "let { name } = params;\nreturn { greeting: `Hello ${name || \"world\"}` };", "timeout": 5000}'
```

### Update a cloud function

`name` cannot be modified.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/CloudFunction/{FUNCTION_ID}' \
  -d '{"code": "return { ok: true };", "isDisabled": false}'
```

### Delete a cloud function

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/CloudFunction/{FUNCTION_ID}'
```

### Deploy

**A service reads its cloud functions only when it starts.** Creating a function, changing its `code` or limits, or toggling `isDisabled` takes effect on the next deployment:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

## Invoking a Cloud Function

Any HTTP method is accepted on `{SERVICE_URL}/functions/{name}`:

```bash
curl -XPOST "$SAASUFY_SERVICE_URL/functions/greet" \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice"}'

curl -XGET "$SAASUFY_SERVICE_URL/functions/greet?name=Alice"
```

**The endpoint is open to any caller — there is no built-in authentication.** Nothing is checked before your code runs: the function itself must enforce access control before it does anything sensitive. See [Authenticating a Caller](#authenticating-a-caller) below.

## Writing the Code

`code` is the body of an `async function`, so `await` and top-level `return` both work. Whatever is returned becomes the JSON body of the response (status `200`, or `204` when nothing is returned).

```js
let { productId } = params;

let product = await crud.read({ type: 'Product', id: productId });
if (!product) {
  response.status(404).json({ error: 'No such product' });
  return;
}

console.log('Looked up', productId);

return { name: product.name, price: product.price };
```

### Globals

- **`params`** — the query string of the request merged with its JSON body (body wins), so the same function can be called by a link, a form or a webhook.
- **`request`** — `{ method, path, url, headers, cookies, ip, query, body, params }`.
- **`response`** — `status(code)`, `setHeader(name, value)` / `set(...)`, `send(body)`, `json(body)`, `end()`, `isSent()`. Chainable. Use it when you need a status code, a header or a non-JSON body; otherwise just `return` a value.
- **`crud`** — `create`, `read`, `update`, `delete`, each taking the same query object as the WebSocket CRUD API minus the `action` property (see [websocket-api.md](websocket-api.md)). Runs as the account admin, so **access-control rules do not restrict it** and realtime subscribers are notified of writes.
- **`r`** — a ReQL query builder scoped to the service's own database, for queries which the CRUD API cannot express. Finish a chain with `.run()`. Terms which would leave the database, run code on it or stream changes (`db`, `js`, `http`, `changes`, `grant`, ...) are blocked, and a chain may be at most 100 terms long.
- **`auth`** — `verifyToken(signedToken)` and `signToken(token)`, for the JWTs of your own users. See [Authenticating a Caller](#authenticating-a-caller).
- **`fetch(url, options)`** — see below.
- **`console`** — `log`/`info`/`debug`/`warn`/`error` write to the service log (visible on the dashboard `Logs` page), prefixed with the function name.
- **`process.env`** — the account's `Constant` records, each typed as it was declared, plus the platform constants (`SAASUFY_ACCOUNT_ID`, `SAASUFY_SERVICE_AUTH_KEY`, ...). See [constants.md](constants.md).

Only standard ECMAScript builtins are available otherwise. There is no `require`/`import`, no Node API, no timers (`setTimeout`) and no filesystem.

### fetch

```js
let result = await fetch('https://api.example.com/items', {
  method: 'POST',
  headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },
  body: { sku: params.sku }
});
if (!result.ok) throw new Error(`Upstream failed with ${result.status}`);
let data = await result.json();
```

The response exposes `url`, `status`, `statusText`, `ok`, `headers`, `text()` and `json()`. A non-string `body` is JSON-stringified for you.

Restrictions: `http`/`https` only; methods `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`; redirects are **not** followed; the Saasufy platform host, your own service host and private address ranges are blocked.

### Per-invocation limits

| Limit | Value |
| --- | --- |
| Wall-clock time | the function's `timeout`, up to 10000 ms |
| Memory | the function's `memoryLimit`, up to 32 MB |
| `fetch` calls | 20, each with a 10 s timeout and a 1 MB response cap |
| Database operations (`crud` + `r`) | 200, each result at most 1 MB |
| Log lines | 100 (further lines are dropped) |

Invocations run on a pool of 4 VMs per service worker; a call which waits more than 10 s for a free VM is rejected with `503`. Raise the worker count to raise cloud function concurrency (see [scalability.md](scalability.md)).

### Error statuses

- `404` — no enabled function of that name (remember: a new function needs a deployment).
- `503` — the service is not ready, or the VM pool was saturated.
- `504` — the function exceeded its time limit.
- `507` — the function exceeded its memory limit.
- `500` — the function threw; the message is returned.

## Authenticating a Caller

The hook runs your code for anyone who calls it, so **every function must enforce its own access control before performing a sensitive operation**. This matters more than usual because `crud` and `r` run as the account admin: access-control rules do not restrict them, so a function which writes based on unverified input is an open door to the whole dataset.

Which check to use depends on who the caller is:

- **A user of your app** → verify their JWT with `auth.verifyToken`, below.
- **A third-party webhook** → compare a shared secret from the request against a `Constant`. See [constants.md](constants.md).
- **Nobody in particular** (a genuinely public endpoint) → still validate and bound the input, and never let `params` choose which record or model is touched.

### The auth global

`auth` signs and verifies JSON Web Tokens with your `serviceAuthKey` — the same key the service signs its WebSocket clients' tokens with. A token your function verifies is therefore the very token your logged-in users already hold, and a token it signs is one those users can log in with.

```js
let claims = await auth.verifyToken(incomingToken);
let outgoingToken = await auth.signToken({ accountId: claims.accountId });
```

Both return a promise, so both must be awaited.

- `verifyToken(signedToken, key, options)` resolves with the token's claims, or rejects if the signature, the expiry or any constraint in `options` fails.
- `signToken(token, key, options)` resolves with the signed string. It expires after your `serviceAuthKey` JWT expiry unless `options.expiresIn` (in seconds) says otherwise.
- `key` and `options` are both optional. Left out, the key is your `serviceAuthKey` and the options are the service's own. Pass a key to work with a token of a different secret, such as `process.env.SAASUFY_EXTERNAL_AUTH_KEY`.

A rejection carries the reason in its message — `TokenExpiredError: jwt expired`, `JsonWebTokenError: invalid signature` — so a function can tell an expired token from a forged one. Do not pass that message back to the caller.

### Verifying the Authorization header

Send the user's signed JWT as a bearer token. Header names arrive lower-cased, so read `request.headers.authorization`:

```js
let header = request.headers.authorization || '';
let signedToken = header.replace(/^Bearer /i, '');

let token;
try {
  token = await auth.verifyToken(signedToken);
} catch (error) {
  response.status(401).json({ error: 'Unauthorized' });
  return;
}

// Scope every read and write to the verified identity, never to params.
let account = await crud.read({ type: 'Account', id: token.accountId });
return { email: account.email };
```

The claims on `token` are the same ones your access-control rules see — `accountId` above all, plus whatever your auth provider put there. See [authentication.md](authentication.md) and [access-control.md](access-control.md).

### Getting the JWT on the frontend

A user who is logged in over WebSockets **already has a signed JWT**; there is no need to mint a new one. It is held by the socket of the `socket-provider`, which is also where the `/files` endpoint takes it from (see [file-hosting.md](file-hosting.md)):

```js
let socket = document.querySelector('socket-provider').saasufySocket;

// The token only exists once the socket has authenticated.
if (socket.authState !== 'authenticated') {
  for await (let event of socket.listener('authStateChange')) {
    if (event.newState === 'authenticated') break;
  }
}

let response = await fetch('https://saasufy.com/sid7999/functions/my-orders', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${socket.signedAuthToken}`
  },
  body: JSON.stringify({})
});
```

The same token is persisted in `localStorage`, which is how it survives a page reload and syncs across tabs. Read it from there when no socket is at hand:

```js
let signedToken = localStorage.getItem(socket.authTokenName);
```

The key is whatever `auth-token-name` was set to on the `socket-provider`; by default it is `socketcluster.authToken.{hostname}` (with `:{port}` appended when the socket URL names one), so a provider on `wss://saasufy.com/sid7999/socketcluster/` stores it under `socketcluster.authToken.saasufy.com`. Prefer `socket.signedAuthToken` or `socket.authTokenName` over a hard-coded key.

Cross-origin calls are fine: Saasufy returns `Access-Control-Allow-Origin: *` on service paths, so the preflight the `Authorization` header triggers passes. Do not set `credentials: 'include'`.

From a script or another server, the same header works with any JWT you hold:

```bash
curl -XPOST "$SAASUFY_SERVICE_URL/functions/my-orders" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $SIGNED_AUTH_TOKEN" \
  -d '{}'
```

This is the user's JWT, not your admin `SAASUFY_API_KEY`; the API key is only for the Admin HTTP API under `https://saasufy.com/api/`, and must never be sent to a cloud function hook or embedded in a frontend.

### Signing a token

`signToken` is for handing an identity to something which cannot log in through the normal flow — a magic link, an invite, a token for an external system:

```js
// Only ever sign for an identity the function has already verified.
let signedToken = await auth.signToken(
  { accountId: token.accountId, scope: 'download' },
  null,
  { expiresIn: 300 }
);
return { signedToken };
```

Because the default key is the `serviceAuthKey`, a token signed this way authenticates a WebSocket client as that `accountId`. Only ever put claims into one that the function has itself established, and keep the expiry short. For a token meant for a system outside Saasufy, sign it with `process.env.SAASUFY_EXTERNAL_AUTH_KEY` instead so it cannot be used to log into the service.

## Statistics and Analytics

Cloud function calls are metered. See [schema-management.md](schema-management.md) for the models and how to read them:

- `Usage`, `ServiceStats` and `ServiceAggregatedStats` carry `serviceCloudFunctionCount` and `serviceCloudFunctionProcessingTime`. A call is also counted in `serviceOpCount`, `serviceHTTPRequestCount` and `serviceRequestProcessingTime`, so those totals remain complete. A scheduled run is counted the same way except that it adds nothing to `serviceHTTPRequestCount`.
- `ServiceAggregatedCloudFunctionAnalytics` holds per-function `callCount`, `errorCount` and `processingTime`, bucketed by `interval` and by `daily`, and is readable over the Admin HTTP API. The dashboard charts it on its `Cloud Functions` stats page.

Database operations which a function performs are also counted as the service operations they are, and show up in the per-model analytics.

## Related

- [constants.md](constants.md) — values and secrets which a function reads through `process.env`, including the platform constants.
- [authentication.md](authentication.md) — how a user gets the JWT which a function verifies.
- [file-hosting.md](file-hosting.md) — the same `Authorization: Bearer` header, used on the `/files` endpoint.
- [scheduled-tasks.md](scheduled-tasks.md) — running a cloud function on a timer instead of over HTTP.
