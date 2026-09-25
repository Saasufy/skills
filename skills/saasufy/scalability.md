# Saasufy Scalability

## When to Use This Skill

Use this skill when an app needs to serve more traffic than a single process can handle, or when you need to reason about how a Saasufy service behaves once it runs on more than one worker. For example:
- Deciding how many workers to give a service and what the plan allows.
- Understanding why realtime updates, aggregations and CRUD requests keep working once the service is spread across several processes.
- Writing app-level logic which stays correct when several workers run it at the same time.
- Understanding what happens to connected clients when a multi-worker service is redeployed or restarted.

Scalability in Saasufy is mostly configuration rather than code: you raise the worker count and the service spreads itself. The parts which are not automatic are called out under [What You Have to Design For](#what-you-have-to-design-for).

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Base URL

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## How a Saasufy Service is Structured

Each account runs one **service**. A service is not a single process:

- On every Saasufy **host** which runs the account, a **primary** process is launched. The primary forks `serviceWorkerCount` **worker** processes and does no request handling itself; it exists to supervise them.
- Every worker listens on the same service port with `reusePort` enabled, and the cluster uses round-robin scheduling. Incoming HTTP requests and WebSocket connections are therefore spread over the workers by the operating system. There is no load balancer to configure and no sticky sessions.
- Saasufy itself may be deployed across several hosts. The number of hosts (`hostCount`) is a property of the Saasufy deployment, not of your account, so you don't set it. What it changes for you is the total: **total workers = `hostCount` × `serviceWorkerCount`**.
- Each worker is given a **worker index** and the **worker count**, numbered across the whole fleet rather than per host (`workerIndex = hostIndex × serviceWorkerCount + localWorkerId`). This is what lets the workers of one host pick up where the workers of another leave off instead of duplicating each other's work.

### The scc-broker requirement

Every worker runs its own in-process pub/sub broker. Nothing one worker publishes reaches another until those brokers are federated by an **scc-broker**. Saasufy therefore **refuses to start a service on more than one worker** (or on a multi-host deployment) when the Saasufy server it is running on has no scc-broker attached; the deploy fails with an `InvalidAccountSettingsError` telling you to set the worker count back to `1`.

This is a deliberate refusal rather than a degraded mode: without a federated broker a multi-worker service would deliver realtime updates only to the clients which happen to be connected to the worker that handled the write, and its aggregations could not hand work between workers.

## Setting the Worker Count

The one scaling lever on the account is `Account.serviceWorkerCount`.

- **Type:** integer, minimum `1`. Leave it empty or null to run on a single worker.
- **Limit:** capped by `Account.maxServiceWorkerCount`, which is set by your plan and defaults to `3`. A value outside `1..maxServiceWorkerCount` is rejected with *"The service worker count must be a number between 1 and N"*.
- **Takes effect on the next deployment.** Changing it does nothing to the running service.

### Via the Dashboard

Log in to `saasufy.com`, open **Settings**, and set **Service worker count**. The current plan limit is shown next to the field. Then deploy the service.

### Via the Admin HTTP API

Your account ID is on every resource the API returns, so read it from any listed resource:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Model?view=accountAlphabeticalView&pageSize=1'
```

Then read and update the account:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Account/{ACCOUNT_ID}'

curl -H "Authorization:Bearer $SAASUFY_API_KEY" -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Account/{ACCOUNT_ID}' \
  -d '{"serviceWorkerCount": 3}'
```

Deploy for it to take effect (see [deployment-and-testing.md](deployment-and-testing.md)):

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" -XPOST 'https://saasufy.com/api/service/start'
```

Relevant `Account` fields:
- `serviceWorkerCount` (integer, optional): Worker processes per host. Defaults to `1`.
- `maxServiceWorkerCount` (integer, read-only for the account): The plan's cap. Defaults to `3`.
- `serviceResponseTimeout` (integer, optional): The socket ack timeout in milliseconds used when talking to the service. Defaults to `30000`. Raise it if legitimate operations are slow enough to time out, but treat a timeout as a signal to index the query rather than to wait longer.
- An API credential may only update a fixed set of account fields: `email`, `serviceWorkerCount`, `serviceResponseTimeout`, `serviceAuthEnabled`, `serviceAllowOrigin`, the auth key and expiry fields, and `isInactive`. Anything else, `maxServiceWorkerCount` included, is rejected.

## What Scales Automatically

You don't need to do anything for these:

- **Client connections and CRUD requests.** Spread over the workers by `reusePort` and round-robin scheduling.
- **Realtime pub/sub.** Once the brokers are federated, a client subscribed on one worker receives updates published by any other worker, so `collection-viewer`, `model-viewer` and every other realtime component keep working unchanged across any worker count.
- **Aggregation pipelines.** Every worker runs every aggregation, but only over the share of its groups which its own worker index owns. Records of a source model carry a maintained `shardKey` field, buckets are divided into 16 shards by default, and each shard is owned by `(shard + offset) % workerCount`. The offset is derived per aggregation, so several aggregations don't all pile their first shard onto the same worker. Adding workers therefore spreads one aggregation across cores rather than pinning it to one; see [data-aggregation-pipelines.md](data-aggregation-pipelines.md).

## What You Have to Design For

### In-process state is per-worker

Each worker is a separate OS process with its own memory. A module-level counter, cache, rate-limit table or in-memory session map exists **once per worker**, and a given client's requests are not pinned to one worker. Anything which has to be consistent across requests must live in the database, not in a variable.

This is the single most common way an app which worked on one worker breaks on three.

### The database is the shared bottleneck

Workers scale request handling, not the database behind it. An unindexed view costs the same whether one worker or six issue it; raising the worker count just lets you issue more of them at once. Before adding workers, make sure views are backed by indexes and that filtering happens in the indexed first phase where possible — see [views-and-indexing.md](views-and-indexing.md) and [search-filtering-querying.md](search-filtering-querying.md).

### Aggregation groups are capped

A group can hold at most 10000 source records by default; beyond that only the first 10000 are aggregated and a warning is logged. This cap is per group and is unaffected by the worker count, so granularity has to come from the grouping. Group readings per sensor per minute rather than per sensor, and chain aggregations to roll them up further.

### Clients reconnect on every deploy

A deploy is a hard restart, not a rolling one (see below). Clients are disconnected and reconnect on their own. `socket-provider` reconnects automatically; tune the backoff with `socket-options` if you need to (`autoReconnectOptions.initialDelay:number=2000,...`) — see [socket-provider.md](socket-provider.md).

## Deploys and Restarts on a Multi-Worker Service

### What a deploy does

Deploying replaces the whole service rather than rolling workers one at a time:

1. The new instance is registered against the account, so that the outgoing instance's exit is recognized as an intentional handover rather than a crash.
2. The host kills the service it currently runs for the account. The **primary is killed first** so that it takes its own workers down with it, instead of relaunching them as they die.
3. The host then **sweeps the service port** and kills any remaining listeners which belong to this account, along with their primaries. This catches workers which an earlier launch left behind: because every worker listens with `reusePort`, a stale worker can keep accepting connections alongside the current ones instead of failing to bind, which would otherwise show up as a service that intermittently serves an old build.
4. The new primary is forked with the current `serviceWorkerCount`, and the service is reported as started only once **every** worker has reported in.

### Multi-host ownership

On a Saasufy deployment with more than one host, each host only stops the processes it launched itself:

- A host tracks the service tree it runs for each account locally, because the `servicePid` recorded on the `Account` record may belong to a different machine. That recorded pid is only used as a fallback when Saasufy runs on a single host.
- When sweeping the port, a host identifies its own workers by matching both the account and its own host index in the process arguments. The other hosts' services listen on the same port for the same account and are deliberately left alone.

The practical consequence: a deploy is safe to issue at any time regardless of how many hosts and workers are involved, and it never half-stops a service by leaving another host's workers serving an old build.

### Crashes

- A worker which exits unexpectedly is relaunched by its primary. A worker which stays up for at least 60 seconds is treated as stable, so the next failure starts the count again.
- After 3 consecutive failures to stay up, the primary gives up on that worker and the whole service exits with a failure, at which point Saasufy restarts it with an exponential backoff. This turns a worker which cannot start (a bad build, a missing model) into a clean, backed-off restart loop rather than a tight relaunch loop.
- A worker which was asked to stop (`SIGINT`/`SIGTERM`, or a deploy) is never relaunched.

## Choosing a Worker Count

- **Start at 1.** A single worker is the right setting for development and for most small apps, and it is the only setting which works if the Saasufy server has no scc-broker.
- **Raise it when the service is CPU-bound**, i.e. when requests queue while the host still has idle cores. Symptoms: rising response times under load, socket ack timeouts, aggregation cycles falling behind.
- **Don't raise it to fix slow queries.** If one request is slow because its view is unindexed, more workers make the database busier without making that request faster. Index first, then scale.
- **Raise it to spread aggregation work** when a service runs several aggregations, or one over a large source model: the shards divide across workers.
- Remember it is per host. On a three-host deployment, `serviceWorkerCount: 3` means nine workers in total.

## Troubleshooting

- **Deploy fails with "A service can only be spread across more than one worker or host when this Saasufy server is connected to an scc-broker":** The Saasufy server has no scc-broker. Set `serviceWorkerCount` back to `1` and deploy again.
- **"The service worker count must be a number between 1 and N":** The value exceeds the plan's `maxServiceWorkerCount`. Read the account to see the cap.
- **Some clients miss realtime updates after raising the worker count:** The brokers are not federated. Check that the deploy actually succeeded rather than failing back to the previous instance, and look at the service logs.
- **A counter, cache or session works on one worker and behaves erratically on several:** It is in-process state. Move it into the database.
- **Behaviour intermittently reverts to an older build:** A stale worker from an earlier launch was left listening on the port. Deploy again; the port sweep removes listeners which belong to the account. If it persists, stop the service and start it again.
- **The worker count change had no effect:** It only takes effect on the next deployment. Deploy the service.
- **Aggregations fall behind or one worker is much busier than the others:** Check the group granularity rather than the worker count; a single oversized group is handled by one worker no matter how many there are. See [data-aggregation-pipelines.md](data-aggregation-pipelines.md).

## Important Notes

1. **`serviceWorkerCount` changes only take effect after the service is deployed.**
2. **More than one worker requires an scc-broker** on the Saasufy server; the deploy is refused otherwise.
3. **Never keep state which must be shared in process memory.** Each worker has its own copy and requests are not pinned to a worker.
4. **The worker count is per host.** The total is `hostCount × serviceWorkerCount`, and the worker indexes are numbered across the whole fleet.
5. **A deploy disconnects clients.** They reconnect automatically, but in-flight requests are lost; design writes to be safe to retry.
6. **Index before you scale.** Workers multiply throughput against a shared database; they do not make an unindexed query cheaper.
