# Saasufy Schema Management Skill

## When to Use This Skill

Use this skill when you need to:
- Create, read, update, or delete Models (database tables)
- Create, read, update, or delete ModelFields (table columns with constraints)
- Create, read, update, or delete ModelViews (query views for filtering/sorting data)
- Create, read, update, or delete ModelIndexes (database indexes for performance)
- Deploy schema changes.

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Base URL

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## Paginating Listing Endpoints

Listing endpoints return one page of records at a time — 10 by default. Tooling which enumerates a model's fields, views or indexes needs to page through the results, or it will act on a partial schema; a 26-field model, for example, would otherwise expose only its first 10 fields alphabetically.

Listing endpoints accept `pageSize` and `offset` query parameters, which work the same way as in the WebSocket CRUD read operation (see [websocket-api.md](websocket-api.md)):

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelField?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}&pageSize=100&offset=0'
```

If a response contains exactly the page-size number of records, request the next page. When paging by `offset`, check that each page contains different records from the previous one before continuing.

## Model Management

Models represent collections (like database tables) in your Saasufy service.
An admin can access the following resources via the admin HTTP API:

- Model
- ModelField
- ModelIndex
- ModelView
- BlockchainAuthProvider
- OAuthProvider
- APICredential
- Usage
- ServiceAggregatedStats
- ServiceAggregatedModelAnalytics

### List All Models

Available views:
- **`accountAlphabeticalView`** — Ordered by name ascending. Params: `accountId`.
- **`accountCustomPositionView`** — Ordered by position. Params: `accountId`.
- **`accountModelWithNameView`** — Find by name. Params: `accountId`, `name`.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XGET 'https://saasufy.com/api/Model?view=accountAlphabeticalView'
```

### Get a Specific Model by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XGET 'https://saasufy.com/api/Model/{MODEL_ID}'
```

### Create a New Model

Required fields:
- `name` (string, alphanumeric, 1-max length)
- `position` (integer, for ordering in UI)

**Avoid naming a `Model` after a browser global** (`Credential`, `Notification`, `Event`, `Request`, `Response`, `Location`, `File`, `Comment`, `Text`, `Option`, `Screen`, ...). Frontend template placeholders fall through to `window`, so `{{Credential.id}}` would resolve against `window.Credential` and render empty. Prefix instead — `UserCredential`. See [debugging-and-troubleshooting.md](debugging-and-troubleshooting.md).

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Model' \
  -d '{"name": "Product", "position": 1}'
```

### Update a Model

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{"name": "UpdatedName"}'
```

### Delete a Model

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/Model/{MODEL_ID}'
```

## ModelField Management

Fields define the structure of data within a Model (like table columns).

### List All Fields for a Model

Available views:
- **`accountModelAlphabeticalView`** — Ordered by name ascending. Params: `accountId`, `modelId`.
- **`accountModelCustomPositionView`** — Ordered by position. Params: `accountId`, `modelId`.
- **`accountModelNameView`** — Find by name. Params: `accountId`, `modelId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelField?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'
```

### Get a Specific Field by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelField/{FIELD_ID}'
```

### Create a New Field

Required fields:
- `modelId` (UUID of the parent model)
- `name` (string, alphanumeric)
- `type` (string: "string", "number", or "boolean")
- `position` (integer, for ordering)

Optional constraint fields:
- `required` (boolean)
- `allowNull` (boolean)
- `min` (number, for strings=min length, for numbers=min value)
- `max` (number, for strings=max length, for numbers=max value)
- `length` (number, exact length for strings)
- `alphanum` (boolean, alphanumeric only)
- `regex` (string, regex pattern)
- `regexFlags` (string, regex flags)
- `email` (boolean, email validation)
- `lowercase` (boolean, force lowercase)
- `uppercase` (boolean, force uppercase)
- `enum` (string, comma-separated allowed values)
- `uuid` (boolean, UUID validation)
- `integer` (boolean, integer only for numbers)
- `multi` (boolean, array of values)
- `blob` (boolean, binary data)
- `defaultValue` (string)
- `visibilityCondition` (string, expression for conditional visibility)
- `maxCardinality` (integer, max number of related items)
- `referentialIntegrityModel` (string, model name for foreign key)

Examples:

**String field with email validation:**
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelField' \
  -d '{"modelId": "{MODEL_ID}", "name": "email", "type": "string", "required": true, "email": true, "position": 1}'
```

**Number field with min/max:**
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelField' \
  -d '{"modelId": "{MODEL_ID}", "name": "age", "type": "number", "integer": true, "min": 0, "max": 150, "position": 2}'
```

**Boolean field:**
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelField' \
  -d '{"modelId": "{MODEL_ID}", "name": "isActive", "type": "boolean", "position": 3}'
```

**Enum field:**
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelField' \
  -d '{"modelId": "{MODEL_ID}", "name": "status", "type": "string", "enum": "draft,published,archived", "position": 4}'
```

### Update a Field

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{FIELD_ID}' \
  -d '{"required": false, "allowNull": true}'
```

### Delete a Field

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/ModelField/{FIELD_ID}'
```

## ModelView Management

Views define how to query and filter data from a Model.

### List All Views for a Model

Available views:
- **`accountModelAlphabeticalView`** — Ordered by name ascending. Params: `accountId`, `modelId`.
- **`accountModelNameView`** — Find by name. Params: `accountId`, `modelId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelView?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'
```

### Get a Specific View by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelView/{VIEW_ID}'
```

### Create a New View

Required fields:
- `modelId` (UUID of the parent model)
- `name` (string, view name)

Optional fields for configuring the view:
- `paramFields` (string, comma-separated field names used as parameters); aim to make them match field names on the model if there is a corresponding field. Note that a param only filters if it is consumed — by `transformIndexOperationInputA`/`transformIndexOperationInputB`, by `transformFilterQuery`, or by `transformFilterOperationInput`; matching the name of a model field is not sufficient on its own.
- `primaryFields` (string, comma-separated primary index fields); can reference any field on the model.
- `affectingFields` (string, fields that affect the view results); not needed most of the time but can be used to ensure that changing a specific non-primary field will trigger realtime notifications for that view.
- `transformIndex` (string, index to use for the transform); can reference any field on the model (or an index); if an index doesn't yet exist on a **single field**, it will be created by Saasufy automatically on the next deployment. Compound indexes are not inferred this way — create the `ModelIndex` first and reference it by its canonical name. See [views-and-indexing.md](views-and-indexing.md).
- `transformIndexOperation` (string, can be "equals" or "between")
- `transformIndexOperationInputA`: (string, the index key to scan from; typically referencing a viewParam passed into the view from the frontend, for example "$paramFields.companyEmployeeCountLow")
- `transformIndexOperationInputB`: (string, the index key to scan to, if the "between" index operation is used)
- For a compound index, each of `transformIndexOperationInputA` and `transformIndexOperationInputB` specifies a *complete index key*, so each must supply a value for every component, comma-separated in index order — e.g. `"$paramFields.clinicianId,$paramFields.fromAt"` and `"$paramFields.clinicianId,$paramFields.toAt"` for an index over `clinicianId,startAt`. Note also that `between` is half-open (`[from, to)`), so a record whose key equals the upper bound is excluded.
- `transformOrderByField` (string, field to sort by); you can provide a field name or a param field such as "$paramFields.sortBy" to allow the client to specify the sorting order dynamically. The field name or the value passed in as the param field can be suffixed with " desc" or " asc" to control the direction; e.g. the value could be "updatedAt desc".
- `transformOrderByDesc` (boolean, descending order; otherwise defaults to ascending)
- `transformFilterType` (string, filter type); can be either "basic" or "advanced". If "basic" is chosen, then you will need to set the `transformFilterField`, `transformFilterOperation` and `transformFilterOperationInput` parameters below. On the other hand, if "advanced" is used, then these will be ignored and you just need to provide the query (or param field reference) via the `transformFilterQuery` field below.
- `transformFilterField` (string, field to filter on); msut refer to a field of the model.
- `transformFilterOperation` (string, filter comparison operator to apply to the `transformFilterField`). Any of the following operators can be used: "contains", "excludes", "=", "!=", ">", ">=", "<" or "<=".
- `transformFilterOperationInput` (string, the parameter or value to compare the `transformFilterField` against using the `transformFilterOperation` as the comparison operator).
- `transformFilterQuery` (string, advanced query to filter records for this view; can be set to "$paramFields.query" to provide maximum flexibility to allow the frontend to construct the query); this is a more flexible alternative to specifying the `transformFilterField`, `transformFilterOperation` and `transformFilterOperationInput` properties individually as it allows constructing more advanced queries with multiple parts.
- `transformDistinct` (boolean, distinct results only)
- `disableRealtime` (boolean, disable realtime updates)
- `maxOffset` (number, max pagination offset)

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelView' \
  -d '{"modelId": "{MODEL_ID}", "name": "activeItemsView", "paramFields": "status", "primaryFields": "status", "affectingFields": "createdAt", "transformOrderByField": "createdAt", "transformOrderByDesc": true}'
```

### Update a View

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelView/{VIEW_ID}' \
  -d '{"transformOrderByDesc": false}'
```

### Delete a View

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/ModelView/{VIEW_ID}'
```

## ModelIndex Management

Indexes improve query performance on specific fields. They also determine what a `ModelView` can return, since a view's `transformIndex` names the index it scans. See [views-and-indexing.md](views-and-indexing.md) for how views use them.

**Index names are assigned by Saasufy**, derived from `fields` in order, camelCased; a `name` supplied at creation is replaced on deployment (`fields: "status,priceAmount"` → `statusPriceAmount`). Refer to an index by this canonical name. If a view's `transformIndex` names neither an existing field nor an existing index, Saasufy creates a placeholder index over a field of that name; where no such field exists, the index matches nothing and the view returns no results.

**Declare compound indexes only.** For a single-field `transformIndex`, name the field in the view and Saasufy creates the index on deploy. It also creates indexes for `multi` fields automatically.

### List All Indexes for a Model

Available views:
- **`accountModelAlphabeticalView`** — Ordered by name ascending. Params: `accountId`, `modelId`.
- **`accountModelNameView`** — Find by name. Params: `accountId`, `modelId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelIndex?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'
```

### Get a Specific Index by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelIndex/{INDEX_ID}'
```

### Create a New Index

Required fields:
- `modelId` (UUID of the parent model)
- `fields` (string, comma-separated field names to index, in order)

Optional fields:
- `format` (string, index format)
- `maxCardinality` (integer, max cardinality for the index)
- `name` (string); **do not set this.** Saasufy derives the canonical name from `fields` and overwrites whatever you supply on deployment.

Compound index — specify `fields` only:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelIndex' \
  -d '{"modelId": "{MODEL_ID}", "fields": "status,createdAt"}'
```

After deployment this index is named `statusCreatedAt`, and that is the name a view's `transformIndex` should use. You can either predict the canonical name (first field as-is, each subsequent field appended with its first letter capitalised) or deploy and read the assigned name back before setting `transformIndex`.

For a single-field index, point the view's `transformIndex` at the field name instead and Saasufy will create the index on the next deployment.

### Update an Index

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelIndex/{INDEX_ID}' \
  -d '{"maxCardinality": 1000}'
```

### Delete an Index

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/ModelIndex/{INDEX_ID}'
```

## BlockchainAuthProvider Management

BlockchainAuthProviders configure blockchain-based authentication for your service.

### List All BlockchainAuthProviders

Available views:
- **`accountAlphabeticalView`** — Ordered by networkSymbol ascending. Params: `accountId`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/BlockchainAuthProvider?view=accountAlphabeticalView'
```

### Get a Specific BlockchainAuthProvider by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/BlockchainAuthProvider/{PROVIDER_ID}'
```

### Create a New BlockchainAuthProvider

Optional fields:
- `networkSymbol` (string, blockchain network identifier e.g. "clsk")
- `chainModuleName` (string, chain module name)
- `hostname` (string, node hostname)
- `port` (integer, node port)
- `secure` (boolean, use secure connection)
- `minAccountBalance` (integer, minimum account balance required for authentication)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/BlockchainAuthProvider' \
  -d '{"networkSymbol": "clsk", "chainModuleName": "capitalisk_chain", "hostname": "node.example.com", "port": 8001, "secure": true, "minAccountBalance": 0}'
```

### Update a BlockchainAuthProvider

Note: `networkSymbol` cannot be modified after creation.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/BlockchainAuthProvider/{PROVIDER_ID}' \
  -d '{"minAccountBalance": 100}'
```

### Delete a BlockchainAuthProvider

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/BlockchainAuthProvider/{PROVIDER_ID}'
```

## OAuthProvider Management

OAuthProviders configure OAuth-based authentication for your service.

### List All OAuthProviders

Available views:
- **`accountAlphabeticalView`** — Ordered by createdAt ascending. Params: `accountId`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/OAuthProvider?view=accountAlphabeticalView'
```

### Get a Specific OAuthProvider by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/OAuthProvider/{PROVIDER_ID}'
```

### Create a New OAuthProvider

Required fields:
- `providerName` (string, name of the OAuth provider)

Optional fields:
- `enabled` (boolean, whether the provider is active)
- `providerClientId` (string, OAuth client ID)
- `providerClientSecret` (string, OAuth client secret)
- `tokenRequestPayloadType` (string, "json" or "form")
- `redirectURI` (string, allowed URI/URL to redirect users to; wildcards may be supported, depending on the provider)
- `getAccessTokenURL` (string, URL to exchange code for access token)
- `getUserInfoURL` (string, URL to fetch user info with access token)
- `providerClientIdField` (string, field name for client ID in token request)
- `providerClientSecretField` (string, field name for client secret in token request)
- `codeField` (string, field name for the code in the token request; if not specified, defaults to 'code')
- `accessTokenField` (string, field name for access token in response)
- `accountIdField` (string, field name for account ID in user info response)
- `usernameField` (string, field name for username in user info response)
- `emailField` (string, field name for email in user info response)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/OAuthProvider' \
  -d '{"providerName": "github", "enabled": true, "providerClientId": "abc123", "providerClientSecret": "secret456", "getAccessTokenURL": "https://github.com/login/oauth/access_token", "getUserInfoURL": "https://api.github.com/user"}'
```

### Update an OAuthProvider

Note: `providerName` cannot be modified after creation.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/OAuthProvider/{PROVIDER_ID}' \
  -d '{"enabled": false}'
```

### Delete an OAuthProvider

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/OAuthProvider/{PROVIDER_ID}'
```

## APICredential Management

APICredentials manage API keys for programmatic access to your service.

### List All APICredentials

Available views:
- **`accountLatestView`** — Ordered by createdAt descending. Params: `accountId`.
- **`accountSearchView`** — Search by name. Params: `accountId`, `nameText`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/APICredential?view=accountLatestView'
```

### Get a Specific APICredential by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/APICredential/{CREDENTIAL_ID}'
```

### Create a New APICredential

Required fields:
- `name` (string, credential name)
- `secretKeyHash` (string, hashed secret key)

Optional fields:
- `serviceAccountId` (UUID, associated service account)
- `allowAdminRead` (boolean, allow admin read access)
- `allowAdminWrite` (boolean, allow admin write access)
- `allowRead` (boolean, allow read access)
- `allowWrite` (boolean, allow write access)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/APICredential' \
  -d '{"name": "my-api-key", "secretKeyHash": "{HASH}", "allowAdminRead": true, "allowAdminWrite": true}'
```

### Update an APICredential

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/APICredential/{CREDENTIAL_ID}' \
  -d '{"allowRead": true, "allowWrite": false}'
```

### Delete an APICredential

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/APICredential/{CREDENTIAL_ID}'
```

## Usage Management

Usage records track per-account API consumption and billing periods. These records are generated automatically by the system and are read-only.

### List Usage Records

Available views:
- **`accountLatestView`** — Ordered by createdAt descending. Params: `accountId`.
- **`accountLatestClosedUnpaidView`** — Closed usage periods that have not been paid yet. Params: `accountId`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Usage?view=accountLatestView&viewParams[accountId]={ACCOUNT_ID}'
```

### Get a Specific Usage Record by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Usage/{USAGE_ID}'
```

### Usage Fields

Read-only fields returned on Usage records:
- `id` (UUID)
- `accountId` (UUID)
- `adminOpCount`, `adminTransmitCount`, `adminInvokeCount`, `adminPublishInCount`, `adminPublishOutCount`, `adminSubscribeCount`, `adminAuthenticateCount`, `adminHTTPRequestCount`, `adminRequestProcessingTime` (numbers, admin API usage counters)
- `serviceOpCount`, `serviceTransmitCount`, `serviceInvokeCount`, `servicePublishInCount`, `servicePublishOutCount`, `serviceSubscribeCount`, `serviceAuthenticateCount`, `serviceHTTPRequestCount`, `serviceRequestProcessingTime` (numbers, service API usage counters)
- `serviceBytesWritten`, `serviceBytesRead`, `serviceBytesStored` (numbers, storage/IO usage)
- `paidAt` (number, timestamp when the usage period was paid; `-1` if unpaid)
- `closedAt` (number, timestamp when the usage period was closed; `-1` if still open)
- `createdAt`, `updatedAt` (numbers, timestamps)

Note: Usage records cannot be created, updated, or deleted via the Admin HTTP API.

## ServiceAggregatedStats Management

ServiceAggregatedStats records store aggregated service usage statistics (interval or daily). These records are generated automatically by the system and are read-only.

### List ServiceAggregatedStats Records

Available views:
- **`accountLatestView`** — Ordered by createdAt descending. Params: `accountId`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ServiceAggregatedStats?view=accountLatestView&viewParams[accountId]={ACCOUNT_ID}'
```

### Get a Specific ServiceAggregatedStats Record by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ServiceAggregatedStats/{STATS_ID}'
```

### ServiceAggregatedStats Fields

Read-only fields returned on ServiceAggregatedStats records:
- `id` (UUID)
- `type` (string, aggregation bucket type; can be `"interval"` or `"daily"`)
- `accountId` (UUID)
- `serviceOpCount`, `serviceTransmitCount`, `serviceInvokeCount`, `servicePublishInCount`, `servicePublishOutCount`, `serviceSubscribeCount`, `serviceAuthenticateCount`, `serviceHTTPRequestCount`, `serviceRequestProcessingTime` (numbers, service API usage counters)
- `serviceBytesWritten`, `serviceBytesRead`, `serviceBytesStored` (numbers, storage/IO usage)
- `createdAt`, `updatedAt` (numbers, timestamps)

Note: ServiceAggregatedStats records cannot be created, updated, or deleted via the Admin HTTP API.

## ServiceAggregatedModelAnalytics Management

ServiceAggregatedModelAnalytics records store aggregated per-model CRUD analytics (interval or daily). These records are generated automatically by the system and are read-only.

### List ServiceAggregatedModelAnalytics Records

Available views:
- **`accountLatestView`** — Latest records across all models, ordered by timestamp descending. Params: `accountId`.
- **`accountLatestIntervalModelView`** — Latest `interval`-type records for a specific model, ordered by timestamp descending. Params: `accountId`, `modelName`.
- **`accountLatestDailyModelView`** — Latest `daily`-type records for a specific model, ordered by timestamp descending. Params: `accountId`, `modelName`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ServiceAggregatedModelAnalytics?view=accountLatestView&viewParams[accountId]={ACCOUNT_ID}'
```

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ServiceAggregatedModelAnalytics?view=accountLatestIntervalModelView&viewParams[accountId]={ACCOUNT_ID}&viewParams[modelName]={MODEL_NAME}'
```

### Get a Specific ServiceAggregatedModelAnalytics Record by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ServiceAggregatedModelAnalytics/{ANALYTICS_ID}'
```

### ServiceAggregatedModelAnalytics Fields

Read-only fields returned on ServiceAggregatedModelAnalytics records:
- `id` (UUID)
- `type` (string, aggregation bucket type; can be `"interval"` or `"daily"`)
- `accountId` (UUID)
- `modelName` (string, name of the model the analytics relate to)
- `createCount`, `readCount`, `updateCount`, `deleteCount` (numbers, per-model CRUD operation counters)
- `fieldInfo` (string, serialized per-field analytics information)
- `viewInfo` (string, serialized per-view analytics information)
- `timestamp` (number, timestamp of the aggregation bucket)
- `createdAt`, `updatedAt` (numbers, timestamps)

Note: ServiceAggregatedModelAnalytics records cannot be created, updated, or deleted via the Admin HTTP API.

## Deploy Schema Changes

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

## Important Notes

1. **Always read the API key from `.saasufy-api-key` file first**
2. **Field types** are limited to: "string", "number", "boolean"
3. **Timestamps** (`createdAt`, `updatedAt`) are automatically managed by the system
4. **When creating fields**, consider the data type and constraints carefully
5. **Views and Indexes** should be created after the Model and Fields are defined
6. After making schema changes, you must deploy/start the service for the changes to take effect.
7. **Index names are assigned by Saasufy** from the indexed fields; declare single-field indexes via a view's `transformIndex` and create `ModelIndex` records only for compound indexes. See [views-and-indexing.md](views-and-indexing.md).
8. **Listing endpoints paginate** (10 records by default) — page through them before acting on the results.
9. **Do not name a Model after a browser global** (`Credential`, `Event`, `Request`, ...) — frontend template placeholders would resolve against `window` and render empty.

## Common Workflows

### Create a Complete Model Schema

1. Create the Model
2. Create Fields for the Model
3. Create Views for querying the Model. For a single-field `transformIndex`, name the **field** — the index will be created for you on deploy
4. Create a `ModelIndex` **only** for compound indexes, specifying `fields` only, and reference each one from a view by its canonical name
5. After making schema changes, you must deploy/start the service for the changes to take effect
6. After deploying, check that every index's `fields` names real fields on the model

### Modify an Existing Schema

1. List Models to find the one to modify
2. List Fields/Views/Indexes for that Model (paging through the results)
3. Update or create new Fields/Views/Indexes as needed
4. After making schema changes, you must deploy/start the service for the changes to take effect
