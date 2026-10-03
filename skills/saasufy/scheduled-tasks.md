# Saasufy Scheduled Tasks

A `ScheduledTask` invokes one of the account's cloud functions at a fixed interval. Use it for polling a third-party API, sending digests, expiring stale records, recomputing summaries and any other recurring server-side work.

Scheduled tasks are managed through the Admin HTTP API (or the `Scheduled Tasks` page of the dashboard).

## Authentication

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

The Admin API base URL is `https://saasufy.com/api/`. See [schema-management.md](schema-management.md).

## ScheduledTask Management

### List scheduled tasks

Available views:
- **`accountAlphabeticalView`** — Ordered by name ascending. Params: `accountId`.
- **`accountNameView`** — Find by exact name. Params: `accountId`, `name`.

```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ScheduledTask?view=accountAlphabeticalView'
```

### Get a scheduled task by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ScheduledTask/{TASK_ID}'
```

### Create a scheduled task

Required fields:
- `name` (string, alphanumeric, 1-50 chars) — identifies the task in the dashboard and the service log. Must be unique within the account and **cannot be changed afterwards**; the record ID is derived from it.

Optional fields:
- `cloudFunctionId` (UUID of a `CloudFunction`) — **null means the task is disabled** and is skipped silently.
- `taskInterval` (integer, min 1) — defaults to `1`.
- `intervalUnit` (string, one of `"milliseconds"`, `"seconds"`, `"minutes"`, `"hours"`, `"days"`, `"months"`) — defaults to `"minutes"`.

The interval is stored as the number and the unit which were given, and combined into a duration only where one is needed. **The combined interval must be at least 60000 ms (one minute)** or the request is rejected. A `months` unit is taken as 30 days, so a monthly task runs at an even interval rather than drifting with calendar months.

System-written fields, present on reads:
- `lastRunAt` (number) — when the task last started; the next run is counted from it.
- `lastError` (string, max 500 chars) — the failure of the last run, or null. Read this to find out why a task is not doing what you expect.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ScheduledTask' \
  -d '{"name": "syncInventory", "cloudFunctionId": "{FUNCTION_ID}", "taskInterval": 15, "intervalUnit": "minutes"}'
```

### Update a scheduled task

`name` cannot be modified. When you change one half of the interval, the minimum is checked against the other half as it is stored.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ScheduledTask/{TASK_ID}' \
  -d '{"taskInterval": 6, "intervalUnit": "hours"}'
```

Disable a task without deleting it by clearing its cloud function:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ScheduledTask/{TASK_ID}' \
  -d '{"cloudFunctionId": null}'
```

### Delete a scheduled task

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/api/ScheduledTask/{TASK_ID}'
```

## When Changes Take Effect

- **A new task needs a deployment.** The task list is read when the service starts.
- **Editing the `cloudFunctionId` or the interval of an existing task applies from its next run** — each task is re-read from the database before it runs.
- **Deleting a task takes effect without a deployment** — the task is dropped the next time it comes due.
- **Changing the `code` of the cloud function itself needs a deployment**, since functions are compiled at startup. For the same reason, a task pointing at a function which was created after the last deployment records a `lastError` instead of running.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

## Writing the Cloud Function of a Task

The function runs exactly as it does over HTTP, except that there is no HTTP caller. It receives a synthetic `POST` request whose body — and therefore `params` — is:

```json
{
  "scheduledTaskId": "...",
  "scheduledTaskName": "syncInventory",
  "triggeredAt": 1767225600000
}
```

Use those to tell a scheduled run from a normal call:

```js
if (params.scheduledTaskName) {
  console.log(`Scheduled run of ${params.scheduledTaskName}`);
  let staleProducts = await r.table('Product')
    .filter(r.row('updatedAt').lt(params.triggeredAt - 86400000))
    .run();
  return { staleCount: staleProducts.length };
}
```

Whatever the function responds with is collected rather than written to a socket. **A status of 400 or above is treated as a failure** and recorded in the task's `lastError`, as is a thrown error. All the usual cloud function limits apply, including the time limit — a task whose work cannot finish within 10 s must break it into batches across runs. See [cloud-functions.md](cloud-functions.md).

## Scheduling Behaviour

- Each task is run by **exactly one worker**. The tasks of an account are sorted by ID and dealt out one per worker in turn, so the schedule is spread over the fleet and a task is never run twice over. See [scalability.md](scalability.md).
- The next run is counted from `lastRunAt`, so **a schedule survives a redeployment** instead of starting over. A task which has never run is due immediately.
- The scheduler ticks once a minute (the shortest allowed interval), so an interval is honoured to within about that granularity.
- Due tasks on a worker run **in series**, longest-due first. Cycles are not stacked: a task which missed its turn is still due next cycle rather than running twice to catch up.
- Each run is metered like a cloud function call — see the statistics section of [cloud-functions.md](cloud-functions.md).

## Related

- [cloud-functions.md](cloud-functions.md) — writing the function a task invokes.
- [constants.md](constants.md) — configuration and secrets a task's function can read.
