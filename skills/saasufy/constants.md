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

This `process.env` holds the account's constants and nothing else — the real environment of the service process is never reachable from inside a cloud function.

Constants are **not** exposed to the frontend or to the WebSocket/data API. They are only readable by cloud functions and by Admin API credentials.

Note that an aggregation pipeline has its own, separate constant mechanism (`AggregationConstantRule`); see [data-aggregation-pipelines.md](data-aggregation-pipelines.md).

## Related

- [cloud-functions.md](cloud-functions.md) — the functions which read these constants.
