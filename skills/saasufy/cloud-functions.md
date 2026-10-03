# Saasufy Cloud Functions

A `CloudFunction` is a piece of JavaScript which Saasufy runs inside your service, in a sandboxed QuickJS VM, and exposes over HTTP at `{SERVICE_URL}/functions/{name}`. Use it for webhooks, server-side validation, third-party API calls and any logic which must not run in the browser (for example because it needs a secret).

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

**The endpoint is open to any caller — there is no built-in authentication.** A function which should only be reachable by some callers must check `request.headers` or `request.cookies` itself and answer accordingly (for example by comparing a shared secret held in a `Constant` — see [constants.md](constants.md)).

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
- **`fetch(url, options)`** — see below.
- **`console`** — `log`/`info`/`debug`/`warn`/`error` write to the service log (visible on the dashboard `Logs` page), prefixed with the function name.
- **`process.env`** — the account's `Constant` records, each typed as it was declared. See [constants.md](constants.md).

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

## Statistics and Analytics

Cloud function calls are metered. See [schema-management.md](schema-management.md) for the models and how to read them:

- `Usage`, `ServiceStats` and `ServiceAggregatedStats` carry `serviceCloudFunctionCount` and `serviceCloudFunctionProcessingTime`. A call is also counted in `serviceOpCount`, `serviceHTTPRequestCount` and `serviceRequestProcessingTime`, so those totals remain complete. A scheduled run is counted the same way except that it adds nothing to `serviceHTTPRequestCount`.
- `ServiceAggregatedCloudFunctionAnalytics` holds per-function `callCount`, `errorCount` and `processingTime`, bucketed by `interval` and by `daily`, and is readable over the Admin HTTP API. The dashboard charts it on its `Cloud Functions` stats page.

Database operations which a function performs are also counted as the service operations they are, and show up in the per-model analytics.

## Related

- [constants.md](constants.md) — values and secrets which a function reads through `process.env`.
- [scheduled-tasks.md](scheduled-tasks.md) — running a cloud function on a timer instead of over HTTP.
