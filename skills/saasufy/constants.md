# Saasufy Constants

A `Constant` is a named, typed value stored on the account which cloud functions read through `process.env`. Use it for API tokens, shared secrets, feature flags, thresholds and anything else a cloud function should not have hard-coded.

Constants are managed through the Admin HTTP API (or the `Constants` page of the dashboard).

## Authentication

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

The Admin API base URL is `https://saasufy.com/api/`. See [schema-management.md](schema-management.md).

## Constant Management

### List constants

Available views:
- **`accountAlphabeticalView`** — Ordered by name ascending. Params: `accountId`.
- **`accountSearchView`** — Search by name. Params: `accountId`, `nameText`.
- **`accountNameView`** — Find by exact name. Params: `accountId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Constant?view=accountAlphabeticalView'
```

### Get a constant by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Constant/{CONSTANT_ID}'
```

### Create a constant

Required fields:
- `name` (string, `[A-Za-z0-9_]+`, 1-50 chars) — the key under `process.env`. Must be unique within the account and **cannot be changed afterwards**; the record ID is derived from it, so creating a second constant with the same name fails.
- `valueType` (string, one of `"string"`, `"number"`, `"boolean"`).

The value is stored in the column matching its type, and only that column is read:
- `stringValue` (string, max 10000 chars)
- `numberValue` (number)
- `booleanValue` (boolean)

Optional:
- `isSecret` (boolean) — only meaningful for a string; it masks the value in the dashboard input. **It does not restrict reads**: an Admin API credential with read access can read the value back.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Constant' \
  -d '{"name": "API_TOKEN", "valueType": "string", "stringValue": "sk-abc123", "isSecret": true}'
```

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Constant' \
  -d '{"name": "MAX_ITEMS_PER_ORDER", "valueType": "number", "numberValue": 25}'
```

### Update a constant

`name` cannot be modified. To change the type of a constant, set the value column of the new type and update `valueType` in the same request.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Constant/{CONSTANT_ID}' \
  -d '{"stringValue": "sk-def456"}'
```

### Delete a constant

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/Constant/{CONSTANT_ID}'
```

### Deploy

**A service reads its constants only when it starts.** Adding, editing or deleting a constant takes effect on the next deployment:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

## Reading Constants from a Cloud Function

Every cloud function is handed the same `process.env` object, with each value kept as the type it was declared with — a `number` constant arrives as a number, not as its digits.

```js
if (request.headers['x-webhook-secret'] !== process.env.WEBHOOK_SECRET) {
  response.status(403).json({ error: 'Forbidden' });
  return;
}

if (params.items.length > process.env.MAX_ITEMS_PER_ORDER) {
  response.status(400).json({ error: 'Too many items' });
  return;
}
```

Each invocation gets its own copy, so writing to `process.env` affects only the function doing the writing and only while it runs. A constant whose value column is empty reads as `null`.

This `process.env` holds the account's constants and the platform constants below, and nothing else — the real environment of the service process is never reachable from inside a cloud function.

Constants are **not** exposed to the frontend or to the WebSocket/data API. They are only readable by cloud functions and by Admin API credentials.

## Platform Constants

Alongside your own constants, `process.env` carries the settings of the account itself — the values on the `Authentication` and `Settings` pages of the dashboard, which are fields of the `Account` record:

| Name | Type | Value |
| --- | --- | --- |
| `SAASUFY_ACCOUNT_ID` | string | The account ID (UUID) which owns the service. |
| `SAASUFY_ACCOUNT_EMAIL` | string | The account's email address. |
| `SAASUFY_ACCOUNT_IS_INACTIVE` | boolean | Whether the account is marked inactive. |
| `SAASUFY_SERVICE_AUTH_KEY` | string | The `serviceAuthKey`: the secret which signs and verifies the JWTs of your own users. See [cloud-functions.md](cloud-functions.md). |
| `SAASUFY_SERVICE_AUTH_KEY_EXPIRY` | number | `serviceAuthKey` JWT expiry in milliseconds. |
| `SAASUFY_EXTERNAL_AUTH_KEY` | string | The `externalAuthKey`: a separate secret for tokens handed to systems outside Saasufy. |
| `SAASUFY_EXTERNAL_AUTH_KEY_EXPIRY` | number | `externalAuthKey` JWT expiry in milliseconds. |
| `SAASUFY_SERVICE_AUTH_ENABLED` | boolean | Whether service authentication is enabled. |
| `SAASUFY_SERVICE_ALLOW_ORIGIN` | string | The service's allowed-origin list. |
| `SAASUFY_SERVICE_RESPONSE_TIMEOUT` | number | The service's admin client response timeout in milliseconds. |
| `SAASUFY_SERVICE_WORKER_COUNT` | number | The total worker count across the fleet, which is the per-host count multiplied by the host count — not the number set on the `Settings` page. See [scalability.md](scalability.md). |

Two things to know about them:

- A setting which is not set is **absent** rather than `null`, so it can be tested the way a missing environment variable would be: `if (process.env.SAASUFY_EXTERNAL_AUTH_KEY) { ... }`.
- They are applied **after** your own constants, so a `Constant` named `SAASUFY_SERVICE_AUTH_KEY` cannot shadow the real one and feed the function a key of its own choosing. Avoid the `SAASUFY_` prefix for your own constants.

`SAASUFY_SERVICE_AUTH_KEY` is the key behind the `auth` global, which is the supported way to verify and sign user tokens; a function rarely needs to read the key itself. Treat all of these as secrets: never return one in a response or log it.

Note that an aggregation pipeline has its own, separate constant mechanism (`AggregationConstantRule`); see [data-aggregation-pipelines.md](data-aggregation-pipelines.md).

## Related

- [cloud-functions.md](cloud-functions.md) — the functions which read these constants.
