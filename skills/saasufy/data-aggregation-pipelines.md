# Data Aggregation Pipelines

## When to Use This Skill

Use this skill when you need a collection whose records are derived automatically from the records of another collection. For example:
- High-score tables and leaderboards (best score per user, per game, per week...)
- Time-series statistics (average/min/max of sensor readings per minute, per hour...)
- Counters and totals (number of orders and revenue per customer, per product, per day...)
- History tables which keep track of how the records of a collection change over time.

Aggregation pipelines run inside the user's deployed service. They keep the target collection up to date in realtime and scale linearly across the service's workers. You don't need any custom backend code or frontend processing.

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Base URL

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## Overview

An aggregation pipeline is made up of an `Aggregation` resource and a set of rules:

- The `Aggregation` belongs to a **target model** (`modelId`) and reads from a **source model** (`sourceModelId`).
- **Project rules** (`AggregationProjectRule`) copy the value of a source field straight into a target field.
- **Group rules** (`AggregationGroupRule`) put a source field's value into a bucket and write the bucket value into a target field. The value is either used as it is (`exact`) or rounded to a multiple of an operand, for example to group timestamps minute by minute.
- **Aggregate rules** (`AggregationAggregateRule`) combine the values of a source field across every source record in a group into a single value (`avg`, `sum`, `min`, `max` or `count`).
- **Constant rules** (`AggregationConstantRule`) write the same fixed value into a target field of every target record.

### How records are grouped

Each source record belongs to a group. The group is identified by the record's values for **every projected field** together with its bucketed values for **every grouped field**. Each group produces exactly **one target record**:

- Project and group fields decide which group a record belongs to. Their values are written as-is into the target record.
- Aggregate fields are computed across all source records in the group.
- Constant fields are the same on every target record.

If an aggregation has no project or group rules, all matching source records fall into a single group, so the target model ends up with a single record.

The ID of a target record is derived from the aggregation ID and the group, so it stays the same over time. The exception is when the `id` field is projected (and the sweep is not disabled): the target record ID is then derived from the source record ID, so the target record stays the same even when other projected values change.

### When aggregations are computed

- **Realtime:** Whenever a source record is created, updated or deleted, the service immediately recomputes the groups affected by the change. That means the group the record is now in and, if it moved or was deleted, the group it left. The target collection updates in realtime, so a `collection-viewer` bound to it updates live.
- **Periodic cycles:** Every `aggregationInterval` milliseconds (or at the service's aggregation interval, which is 60 seconds by default) the service walks through source records which were written since the last cycle and recomputes their groups. This catches anything the realtime path missed. New writes are left to settle for about 5 seconds before a cycle picks them up.
- **Sweep:** After a change to the grouping or a rebuild, the service walks through the target records once and deletes the ones which no longer have any source records behind them (unless `disableSweep` is set).

A group is always recomputed from scratch by reading all its source records. This keeps target records correct through edits and deletions, and makes the process safe to repeat. A target record is only rewritten if one of its values actually changed.

### Chaining pipelines

The target model of one aggregation can be the source model of another. For example, readings can be rolled up per minute, and those per-minute records rolled up again per hour. Cycles are not allowed: an aggregation is rejected if its source model is already fed by its target model through other aggregations.

## Schema

The following models are used to define aggregation pipelines. They are all accessible via the Admin HTTP API.

### Aggregation

Defines a pipeline which rolls up the records of a source model into the records of a target model.

Fields:
- `id` (UUID, read-only): Derived from the account, the target model and the `aggregationName`.
- `aggregationName` (string, alphanumeric, required): Must be unique among the aggregations of the target model. **Cannot be modified after creation.**
- `modelId` (UUID, required): The ID of the **target** model which the aggregation writes records into. **Cannot be modified after creation.**
- `sourceModelId` (UUID, required): The ID of the **source** model which the aggregation reads records from. Cannot be the same as `modelId` and cannot introduce a cycle between aggregations.
- `sourceFilterQuery` (string, optional): A query in the Saasufy query format (see [search-filtering-querying.md](search-filtering-querying.md)). Only source records which match the query take part in the aggregation; e.g. `status = active ~AND~ amount >= 10`. If left empty or null, then every source record takes part.
- `minSourceUpdatedAt` (integer, optional): A timestamp in milliseconds since the Unix epoch. Source records whose `updatedAt` is older than this value are left out. This avoids walking over a long history of old records. If null, then there is no lower bound.
- `aggregationInterval` (integer, optional): How often the periodic cycle runs for this aggregation, in milliseconds. Must be at least `10000`. If null, then the service's aggregation interval (60000 by default) is used.
- `disableSweep` (boolean, optional): If `true`, then the service never deletes target records. Every target record which has ever been written is kept, even when its source records are edited out of its group or deleted. This is what allows history tables to be built (see the examples below).
- `isPaused` (boolean, optional): If `true`, then the aggregation doesn't run at all. Takes effect on the next deployment.
- `rebuildRequestedAt` (number, optional): Set this to the current timestamp (`Date.now()`) to make the service recompute the aggregation from every source record and sweep the target model again. **This takes effect without a redeployment.**
- `lastRunAt` (number, written by the service): When the aggregation last processed records.
- `lastError` (string, written by the service): The last error encountered by the aggregation, or the reason why it was skipped (e.g. a missing target field). Null if there is no error. Check this field when an aggregation isn't producing the expected records.
- `lastErrorWorkerIndex` (integer, written by the service): Which worker reported `lastError`.
- `createdAt`, `updatedAt` (numbers, timestamps, automatic)

Limits: A model can have at most 100 aggregations.

Available views:
- **`accountModelAlphabeticalView`**: Aggregations of a target model ordered by name. Params: `accountId`, `modelId`.
- **`accountModelAggregationNameView`**: Find an aggregation by name. Params: `accountId`, `modelId`, `aggregationName`.

### AggregationProjectRule

Copies the value of a source field straight into a target field. Source records are only combined into the same target record when they share the same value for every projected field, so a project rule also acts as an "exact match" grouping.

Fields:
- `id` (UUID, read-only): Derived from the `aggregationId` and `sourceField`, so there can be only one project rule per source field in an aggregation.
- `aggregationId` (UUID, required): **Cannot be modified after creation.**
- `sourceField` (string, required): A field of the source model. It can also be one of the automatic fields: `id`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy` or `updatedByIp`. It cannot be a `multi` field. **Cannot be modified after creation.**
- `targetField` (string, optional): The field of the target model to write the value into. Defaults to the same name as `sourceField`. **The target field must be declared as a ModelField on the target model, otherwise the whole aggregation is skipped.**
- `createdAt`, `updatedAt` (numbers, timestamps, automatic)

Notes:
- Only strings, numbers and booleans can be projected. Source records holding an object or array in a projected field are ignored.
- A source record with no value for a projected field is grouped under `null`. Such groups can't be read through an index, so they are much slower to recompute. Prefer projecting fields which are `required`.
- Projecting `id` produces one target record per source record.

### AggregationGroupRule

Puts the value of a source field into a bucket and groups source records which land in the same bucket. The bucket value is written into the target field.

Fields:
- `id` (UUID, read-only): Derived from the `aggregationId` and `sourceField`, so there can be only one group rule per source field in an aggregation.
- `aggregationId` (UUID, required): **Cannot be modified after creation.**
- `sourceField` (string, required): A field of the source model, or one of the automatic fields (e.g. `createdAt` or `updatedAt`). It cannot be a `multi` field. **Cannot be modified after creation.**
- `operation` (string, required): One of:
  - `exact`: Groups by the value as it is. Does not require an operand. Works with strings, numbers and booleans.
  - `round-down`: Rounds the number down to a multiple of `operand` (`Math.floor(value / operand) * operand`).
  - `round`: Rounds the number to the nearest multiple of `operand`.
  - `round-up`: Rounds the number up to a multiple of `operand`.
- `operand` (number): Required and must be greater than 0 for the `round-down`, `round` and `round-up` operations. The source field must be declared as a `number` field for these operations. For example, `round-down` with an operand of `60000` on the `createdAt` field groups records minute by minute. `3600000` groups by hour and `86400000` groups by day (UTC).
- `targetField` (string, optional): Defaults to the same name as `sourceField`. **Must be declared as a ModelField on the target model, otherwise the whole aggregation is skipped.**
- `createdAt`, `updatedAt` (numbers, timestamps, automatic)

Notes:
- Source records whose grouped field doesn't hold a valid value (e.g. a missing or non-numeric value with a rounding operation) are left out of the aggregation.

### AggregationAggregateRule

Combines the values which every source record in a group holds for a source field into a single value on the target record.

Fields:
- `id` (UUID, read-only): Derived from the `aggregationId`, `sourceField` and `operation`. You can have several aggregate rules on the same source field as long as each uses a different operation (e.g. `min`, `max` and `avg` of `temperature`).
- `aggregationId` (UUID, required): **Cannot be modified after creation.**
- `sourceField` (string, required): A field of the source model. **Cannot be modified after creation.**
- `operation` (string, required): One of the following. **Cannot be modified after creation.**
  - `avg`: The average of the numeric values.
  - `sum`: The sum of the numeric values.
  - `min`: The smallest numeric value.
  - `max`: The largest numeric value.
  - `count`: The number of source records in the group which have a non-null value for the source field (of any type). Use `id` as the source field to count all records in the group.
- `operand` (number, optional): Only used by the `avg` operation. It is the number of decimal places to round the result to. If not set, then the result is not rounded.
- `modifier` (string, optional): Only used by the `avg` operation when `operand` is set. Can be `floor`, `round` or `ceil`. Defaults to `round`.
- `targetField` (string, optional): Defaults to the same name as `sourceField`. If multiple rules read from the same source field, then each of them must specify a distinct `targetField`. If the target field is not declared on the target model, then that rule's value is not written (a warning is logged but the rest of the aggregation still runs).
- `createdAt`, `updatedAt` (numbers, timestamps, automatic)

Notes:
- `avg`, `sum`, `min` and `max` ignore values which are not finite numbers. `avg` and `sum` produce `null` when the group has no numeric values.
- A group can contain at most 10000 source records by default. Beyond that, only the first 10000 are aggregated and a warning is logged. Design groups to be granular enough; e.g. group readings per sensor per minute rather than per sensor.

### AggregationConstantRule

Writes a fixed value into a field of every target record. This is useful for adding a value which no source record carries, such as a `type`, `category` or flag used to filter the target collection through a view.

Fields:
- `id` (UUID, read-only): Derived from the `aggregationId` and `targetField`, so there can be only one constant rule per target field.
- `aggregationId` (UUID, required): **Cannot be modified after creation.**
- `targetField` (string, required): The field of the target model to write into. **Cannot be modified after creation.**
- `valueType` (string, required): Can be `string`, `number` or `boolean`.
- `stringValue` (string, max 500 characters): The value if `valueType` is `string`. Defaults to `""`.
- `numberValue` (number): The value if `valueType` is `number`. Defaults to `0`.
- `booleanValue` (boolean): The value if `valueType` is `boolean`. Defaults to `false`.
- `createdAt`, `updatedAt` (numbers, timestamps, automatic)

Notes:
- The constant takes no part in the grouping. Changing it only affects target records as their groups get recomputed, so request a rebuild to apply it to all existing records.

Limits: An aggregation can have at most 50 rules of each kind.

Available views (same for all four rule models):
- **`accountAggregationView`**: Rules of an aggregation ordered by creation time. Params: `accountId`, `aggregationId`.

## Managing Aggregation Pipelines

### Setting Up a Pipeline

1. Create the source model and its fields (if it doesn't already exist).
2. Create the target model and **declare a ModelField for every target field** which the rules will write to, using a matching type. The target fields of project and group rules are mandatory, otherwise the aggregation is skipped.
3. Set access control on the target model. Target records are written by the service itself, so users typically only need read access. Set `accessCreate`, `accessUpdate` and `accessDelete` to `"block"` (see [access-control.md](access-control.md)).
4. Create the `Aggregation` on the target model.
5. Create the rules for the aggregation.
6. Create ModelViews (and indexes) on the target model to display the aggregated data (see [schema-management.md](schema-management.md)).
7. Deploy the service. **Aggregation changes only take effect once the service has been deployed.**
8. Check the `lastRunAt` and `lastError` fields of the `Aggregation` to confirm that it is running.

### Create an Aggregation

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Aggregation' \
  -d '{"aggregationName": "bestScores", "modelId": "{TARGET_MODEL_ID}", "sourceModelId": "{SOURCE_MODEL_ID}"}'
```

### List Aggregations of a Target Model

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Aggregation?view=accountModelAlphabeticalView&viewParams[modelId]={TARGET_MODEL_ID}'
```

### Get an Aggregation (e.g. to check its status)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}'
```

### Update an Aggregation

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}' \
  -d '{"sourceFilterQuery": "isVerified = true", "aggregationInterval": 30000}'
```

### Create Rules

```bash
# Project rule
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "userId"}'

# Group rule
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationGroupRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "createdAt", "operation": "round-down", "operand": 86400000, "targetField": "day"}'

# Aggregate rule
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "points", "operation": "avg", "operand": 2, "modifier": "round", "targetField": "averagePoints"}'

# Constant rule
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationConstantRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "targetField": "period", "valueType": "string", "stringValue": "daily"}'
```

### List Rules of an Aggregation

Replace `AggregationGroupRule` with `AggregationProjectRule`, `AggregationAggregateRule` or `AggregationConstantRule` as needed.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/AggregationGroupRule?view=accountAggregationView&viewParams[aggregationId]={AGGREGATION_ID}'
```

### Update or Delete a Rule

Only the non-frozen fields of a rule can be updated (e.g. `targetField`, `operand`, `modifier`, or the value of a constant rule). To change a frozen field such as `sourceField` or an aggregate rule's `operation`, delete the rule and create a new one.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/AggregationAggregateRule/{RULE_ID}' \
  -d '{"operand": 1}'

curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/AggregationAggregateRule/{RULE_ID}'
```

### Rebuild an Aggregation

Recomputes the aggregation from every source record and sweeps the target model again. This does not require a deployment.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}' \
  -d "{\"rebuildRequestedAt\": $(date +%s%3N)}"
```

### Pause or Resume an Aggregation

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}' \
  -d '{"isPaused": true}'
```

Then deploy the service for the change to take effect.

### Delete an Aggregation

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}'
```

Then deploy the service to stop it from running. Existing target records are not deleted.

### Deploy

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

### What Happens When an Aggregation is Changed

- **Adding, removing or changing project/group rules** (or their operations or operands) changes how records are grouped. After the next deployment, the aggregation is automatically recomputed from the beginning.
- **Changing `sourceFilterQuery`, `minSourceUpdatedAt`, aggregate rules or constant rules** only affects groups as they get recomputed. Rebuild the aggregation (after deploying) to apply the change to all existing target records. Note that lowering `minSourceUpdatedAt` only brings in the older records after a rebuild.
- **Changing `disableSweep`** changes how target records are identified, so records which have already been aggregated are left as they are.

## Examples

The examples below assume that the models and fields have already been created as described in [schema-management.md](schema-management.md). `{...}` placeholders stand for the IDs returned when creating each resource.

### Example 1: High-Score Table

**Goal:** Show a leaderboard with the best score of each player in each game, along with how many games they have played and their average score.

**Source model `Score`** (one record per game played):
- `gameId` (string, required)
- `userId` (string, required)
- `username` (string, required)
- `points` (number, required)

**Target model `HighScore`**:
- `gameId` (string)
- `userId` (string)
- `username` (string)
- `bestScore` (number)
- `averageScore` (number)
- `gamesPlayed` (number)

Set the `HighScore` model to be publicly readable but not writable by users:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{HIGH_SCORE_MODEL_ID}' \
  -d '{"accessCreate": "block", "accessRead": "allow", "accessUpdate": "block", "accessDelete": "block"}'
```

Create the aggregation and its rules:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Aggregation' \
  -d '{"aggregationName": "bestScores", "modelId": "{HIGH_SCORE_MODEL_ID}", "sourceModelId": "{SCORE_MODEL_ID}"}'

# One HighScore record per (gameId, userId, username)
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "gameId"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "userId"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "username"}'

# Aggregated values
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "points", "operation": "max", "targetField": "bestScore"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "points", "operation": "avg", "operand": 1, "targetField": "averageScore"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "points", "operation": "count", "targetField": "gamesPlayed"}'
```

Create a view on `HighScore` which lists the best scores of a game in descending order:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelView' \
  -d '{"modelId": "{HIGH_SCORE_MODEL_ID}", "name": "gameLeaderboardView", "paramFields": "gameId", "primaryFields": "gameId", "affectingFields": "bestScore", "transformOrderByField": "bestScore", "transformOrderByDesc": true}'
```

Deploy the service, then display the top 10 players with a `collection-viewer` (see [collection-viewer.md](collection-viewer.md)). It updates in realtime as new scores are submitted:

```html
<collection-viewer
  collection-type="HighScore"
  collection-fields="username,bestScore,averageScore,gamesPlayed"
  collection-view="gameLeaderboardView"
  collection-view-params="gameId=space-invaders"
  collection-page-size="10"
>
  <template slot="item">
    <div class="leaderboard-row">
      <span>{{HighScore.username}}</span>
      <span>{{HighScore.bestScore}}</span>
      <span>{{HighScore.averageScore}} avg over {{HighScore.gamesPlayed}} games</span>
    </div>
  </template>
  <div slot="viewport"></div>
</collection-viewer>
```

Variations:
- **Weekly or daily leaderboards:** Add a group rule on `createdAt` with the `round-down` operation and an operand of `604800000` (week) or `86400000` (day), targeting a `period` field. Then add `period` to the view's `paramFields` and `primaryFields`. Note that Unix epoch weeks start on Thursdays (UTC).
- **Only count valid scores:** Set `sourceFilterQuery` on the aggregation, e.g. `isVerified = true`.
- **Usernames that can change:** Because `username` is projected, a player who changes their username gets a new `HighScore` record for the scores submitted under the new name. If usernames can change, project only `userId` and look up the username separately (e.g. with a `model-text` bound to the user's record).

### Example 2: Time-Series Running Averages

**Goal:** Track the average, minimum and maximum temperature of each sensor per minute. Then roll the per-minute stats up into correct per-hour stats.

**Source model `SensorReading`**:
- `sensorId` (string, required)
- `temperature` (number, required)

**Target model `SensorMinuteStats`**:
- `sensorId` (string)
- `minute` (number): the start of the minute, as a timestamp
- `avgTemperature` (number)
- `minTemperature` (number)
- `maxTemperature` (number)
- `temperatureSum` (number)
- `readingCount` (number)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Aggregation' \
  -d '{"aggregationName": "perMinute", "modelId": "{SENSOR_MINUTE_STATS_MODEL_ID}", "sourceModelId": "{SENSOR_READING_MODEL_ID}"}'

curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "sensorId"}'

# Bucket readings by the minute in which they were created
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationGroupRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "createdAt", "operation": "round-down", "operand": 60000, "targetField": "minute"}'

curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "temperature", "operation": "avg", "operand": 2, "targetField": "avgTemperature"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "temperature", "operation": "min", "targetField": "minTemperature"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "temperature", "operation": "max", "targetField": "maxTemperature"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "temperature", "operation": "sum", "targetField": "temperatureSum"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "temperature", "operation": "count", "targetField": "readingCount"}'

# View: latest minutes of a sensor first
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelView' \
  -d '{"modelId": "{SENSOR_MINUTE_STATS_MODEL_ID}", "name": "sensorLatestView", "paramFields": "sensorId", "primaryFields": "sensorId", "transformOrderByField": "minute", "transformOrderByDesc": true}'
```

The record of the current minute is recomputed every time a new reading arrives. The average therefore updates in realtime as the minute progresses, and a `collection-viewer` bound to `sensorLatestView` with a page size of `60` shows a live chart of the last hour.

**Rolling up to hourly stats (chaining):** Create a `SensorHourStats` model with `sensorId`, `hour`, `minTemperature`, `maxTemperature`, `temperatureSum` and `readingCount` fields. Then create an aggregation on it whose source model is `SensorMinuteStats`:
- Project rule: `sensorId`
- Group rule: `minute` with `round-down` and an operand of `3600000`, into `hour`
- Aggregate rules: `minTemperature` with `min`, `maxTemperature` with `max`, `temperatureSum` with `sum` and `readingCount` with `sum`

Don't average the per-minute averages: minutes with few readings would carry as much weight as minutes with many. Instead, compute the hourly average on the frontend as `temperatureSum / readingCount`, for example: `{{Math.round(SensorHourStats.temperatureSum / SensorHourStats.readingCount * 100) / 100}}`.

Tips for time-series:
- Group on `createdAt` rather than `updatedAt`. `createdAt` never changes, so an edited reading stays in the same bucket.
- Use `round-down` so that the bucket value is the start of the period.
- Keep each group under 10000 source records. If a sensor produces more than that per hour, aggregate per minute and chain to hourly as shown above instead of aggregating hourly from raw readings.
- Set `minSourceUpdatedAt` if the source model holds a long history which doesn't need to be aggregated.
- A cumulative running average of all readings of a sensor can be produced with an aggregation which only has a `sensorId` project rule and an `avg` aggregate rule. A sliding-window average (e.g. over the last 10 minutes) can be computed on the frontend from the per-minute records; see [collection-reducer.md](collection-reducer.md).

### Example 3: History Tables

**Goal:** Keep a record of every version of each `Product` record, so the frontend can show how its price and stock changed over time.

This relies on `disableSweep`. With the sweep disabled, target records are never deleted. Each new version of a source record lands in a new group (because its `updatedAt` changed), so a new target record is written for it while the target records of previous versions are kept.

**Source model `Product`**:
- `name` (string, required)
- `price` (number, required)
- `stock` (number, required)

**Target model `ProductHistory`**:
- `productId` (string)
- `changedAt` (number)
- `changedBy` (string)
- `name` (string)
- `price` (number)
- `stock` (number)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Aggregation' \
  -d '{"aggregationName": "productHistory", "modelId": "{PRODUCT_HISTORY_MODEL_ID}", "sourceModelId": "{PRODUCT_MODEL_ID}"}'

# Never delete history records
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Aggregation/{AGGREGATION_ID}' \
  -d '{"disableSweep": true}'

# One target record per (product id, updatedAt) = one record per version
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "id", "targetField": "productId"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationGroupRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "updatedAt", "operation": "exact", "targetField": "changedAt"}'

# Copy the values of the version
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "updatedBy", "targetField": "changedBy"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationProjectRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "name"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "price", "operation": "max", "targetField": "price"}'
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/AggregationAggregateRule' \
  -d '{"aggregationId": "{AGGREGATION_ID}", "sourceField": "stock", "operation": "max", "targetField": "stock"}'

# View: history of a product, newest first
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelView' \
  -d '{"modelId": "{PRODUCT_HISTORY_MODEL_ID}", "name": "productHistoryView", "paramFields": "productId", "primaryFields": "productId", "transformOrderByField": "changedAt", "transformOrderByDesc": true}'
```

Why this works:
- `id` plus `updatedAt` identifies exactly one version of one product, so each group contains a single source record.
- Numeric values are copied with the `max` aggregate operation. The group holds a single record, so `max` is just its value. Unlike projecting them, this doesn't make every numeric value part of the group's identity or the backing index. String values have to be projected; project only fields which are always present, because missing values make groups slow to recompute.
- `updatedBy` (projected into `changedBy`) records which account made the change. The service sets it automatically on writes made by logged-in users. Writes made without a user session (e.g. by anonymous users) leave it empty, which makes those versions slower to capture. If that is common in your app, leave out the `changedBy` rule.

Caveats:
- History entries are captured when the service recomputes a group. If a record is updated several times within a few milliseconds, some intermediate versions may be missed. For a strict audit log where every write must be recorded, write explicit log records from the application instead.
- Deleting a source record keeps its history but doesn't add a "deleted" entry. To record deletions, use a soft-delete flag (e.g. set an `isDeleted` boolean field and project it) instead of deleting records.
- History tables grow forever since nothing is swept. Set `minSourceUpdatedAt` to limit how far back the history goes when first creating the aggregation on a model which already has a lot of data.

**Variation: daily snapshots.** Group on `updatedAt` with `round-down` and an operand of `86400000` (into a `day` field) instead of `exact`. You then get one history record per product per day, holding the product's last version for that day. Keep `disableSweep` enabled so that previous days are retained.

## Important Notes

1. **Aggregation changes only take effect after the service is deployed.** The one exception is `rebuildRequestedAt`.
2. **Declare every target field on the target model** with a matching type. Missing project/group target fields cause the whole aggregation to be skipped; the reason is written to the aggregation's `lastError` field.
3. **Target records are written by the service.** Block user create/update/delete access on the target model; manual edits to target records are overwritten the next time their group is recomputed.
4. **Every source field used by a rule must exist on the source model** (or be an automatic field). `multi` fields cannot be projected or grouped.
5. **Two rules cannot write to the same target field.** Use distinct `targetField` values when several rules read from the same source field.
6. **Groups are capped at 10000 source records by default**; design groups to be granular and chain aggregations to roll them up further.
7. **Aggregations can be chained** but cannot form cycles.
8. **Aggregation work counts towards service usage.** A shorter `aggregationInterval` keeps the periodic cycle more responsive but costs more database operations. Realtime updates from source record writes happen regardless of the interval.

## Troubleshooting

- **No target records appear:** Check that the service has been deployed and look at the aggregation's `lastRunAt` and `lastError` fields. A null `lastRunAt` with no error means that the aggregation hasn't run yet; the first cycle starts a few seconds after deployment.
- **An aggregate value is missing from target records:** The target field is probably not declared on the target model. Declare it, deploy, then rebuild the aggregation.
- **Target records don't reflect a changed filter query, constant or aggregate rule:** Deploy, then rebuild the aggregation.
- **Stale target records remain after source records were deleted:** Check that `disableSweep` is not set. Rebuild the aggregation to trigger a fresh sweep.
- **Records are unexpectedly split into several target records:** Every projected field is part of the group's identity. Remove project rules for fields which vary between records that should be combined.
