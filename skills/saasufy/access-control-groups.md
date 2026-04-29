# Saasufy Group-Based Access Control

## When to Use This Skill

Use this skill when you need to:
- Build multi-tenant applications where each tenant is represented as a Group (e.g. teams, organizations, workspaces, projects, channels)
- Allow users to create their own Groups and invite/add other users as members
- Restrict creation/modification of records (in arbitrary models) to a specific Group's owner or members
- Share data between a defined set of users (a Group) without exposing it to the rest of the platform
- Combine Saasufy's standard owner-based access control with shared, group-scoped collaboration

If you only need owner-based access (one user owns one record), use the standard rules in [access-control.md](access-control.md). Use this skill when records must be shared with multiple users via group membership.

## Concept Overview

Saasufy provides two special built-in models that, when present, unlock group-based authorization:

- **`Group`** — Represents a tenant/team/workspace. Each Group has a single owner (`accountId`).
- **`GroupMembership`** — Represents the link between an account and a Group. Each membership record references one `groupId` and one `accountId`.

When these collections exist, the Saasufy auth layer automatically attaches two extra fields to every authenticated user's JWT:

- `groupOwnerships` — Comma-separated list of group IDs where the user is the owner (`Group.accountId === user.accountId`).
- `groupMemberships` — Comma-separated list of group IDs where the user is either an owner *or* an active member (owners are implicitly members of their own groups).

These token fields can then be referenced from the access control settings of *any other* model via `accessTokenAuthField`. Saasufy verifies authorization by checking whether the model's auth field value (e.g. `groupId`) appears anywhere in that comma-separated list on the JWT.

The `Group` and `GroupMembership` collections themselves have hard-coded authorization rules baked into the Saasufy service — these rules cannot be relaxed via standard model access settings and are described below.

## Built-In Authorization Rules for `Group`

When a `Group` model exists, the Saasufy service enforces the following rules at the service layer (in addition to whatever standard access control you configure):

| Action | Rule |
|--------|------|
| `create` | The user must be authenticated. Standard `accessCreate` rules also apply. |
| `update` | The user must be authenticated **and** the target group's `id` must appear in `socket.authToken.groupOwnerships` (i.e. the user owns this group). Admins bypass. |
| `delete` | Same as `update` — only the owner (or an admin) can delete. |

The following fields are frozen and cannot be modified after creation: `id`, `createdAt`, `updatedAt`, `updatedByIp`.

### Required Schema for `Group`

The Saasufy service expects (and auto-extends) the `Group` model to include these fields:

- `id` — UUID v4
- `name` — string, required, 1–200 characters
- `accountId` — string, required (the owner's account ID)
- `isDeactivated` — boolean (optional)
- `createdAt`, `updatedAt` — integer timestamps (required)
- `updatedByIp` — string

Indexes auto-managed by the service: `accountId`, `createdAt`, `updatedAt`, `updatedByIp`.

### Recommended Access Settings for `Group`

A typical setup that lets users create their own groups and lets owners manage them (the service also enforces ownership on update/delete regardless):

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)

curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{GROUP_MODEL_ID}' \
  -d '{
    "accessTokenAuthField": "accountId",
    "accessModelAuthField": "accountId",
    "accessCreate": "restrict",
    "accessRead": "allow",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

## Built-In Authorization Rules for `GroupMembership`

`GroupMembership` is the heart of group-based access control. Its built-in rules guarantee that only group owners can manage their group's membership list:

| Action | Rule |
|--------|------|
| `create` | The user must be authenticated **and** the `groupId` being inserted must appear in `socket.authToken.groupOwnerships`. In other words, **only the owner of the target group can add members to it.** Admins bypass. |
| `update` | The user must be authenticated, and a post-check verifies that the existing record's `groupId` is in the user's `groupOwnerships`. Only the group owner can modify a membership row. |
| `delete` | Same as `update` — only the group owner (or an admin) can remove a member. |

The following fields are frozen: `id`, `createdAt`, `updatedAt`, `updatedByIp`.

These rules are enforced by Saasufy's service worker independently of the model-level access flags — even if `accessCreate` is set to `"allow"`, a non-owner attempting to create a `GroupMembership` for a group they do not own will be rejected with: *"Only the group owner can add members to the group"*.

### Required Schema for `GroupMembership`

- `id` — UUID v4
- `groupId` — UUID v4, required (the target group)
- `accountId` — UUID v4, required (the member being added)
- `createdAt`, `updatedAt` — integer timestamps (required)
- `updatedByIp` — string

Indexes auto-managed by the service: `groupId`, `accountId`, `createdAt`, `updatedAt`, `updatedByIp`, plus a compound index `groupIdAccountId` (used to look up a specific user's membership in a specific group).

### Recommended Access Settings for `GroupMembership`

The Saasufy admin UI configures this model with the following settings, which work cleanly with the built-in service rules:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{GROUP_MEMBERSHIP_MODEL_ID}' \
  -d '{
    "accessTokenAuthField": "groupOwnerships",
    "accessModelAuthField": "groupId",
    "accessCreate": "restrict",
    "accessRead": "allow",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

The `accessTokenAuthField=groupOwnerships` + `accessModelAuthField=groupId` combination tells Saasufy: "the JWT's `groupOwnerships` list must contain this record's `groupId`". This aligns the standard model-level checks with the service-level rule so behavior stays consistent if the service-level rule is ever relaxed.

## Setting Up the Models

Models can be created via the Admin HTTP API (see [schema-management.md](schema-management.md)) or, more conveniently, by the built-in widgets on the **Authentication** page in the Saasufy dashboard, which create them with the correct names and access settings in one click.

If creating manually:

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)

# Create the Group model
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Model' \
  -d '{"name": "Group", "position": -2}'

# Create the GroupMembership model
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Model' \
  -d '{
    "name": "GroupMembership",
    "position": -3,
    "accessTokenAuthField": "groupOwnerships",
    "accessModelAuthField": "groupId"
  }'
```

After creating the models, **deploy the service** by clicking the *Deploy service* button on the Saasufy Dashboard. The service worker only attaches the schema extensions and JWT token fields after the models are detected at deploy time.

The Saasufy service auto-extends the schemas (fields and indexes) on deploy — you don't need to add the standard fields manually. You may add custom fields to either model (e.g. a `description` on `Group`, or a `role` on `GroupMembership`) and they will coexist with the built-in fields.

### Token Field Limits

JWTs only carry up to **250** group ownerships and **250** group memberships per user (see `maxGroupOwnershipsPerAccount` / `maxGroupMembershipsPerAccount` in the Saasufy config). If a user exceeds the limit, only the most recently updated groups will appear in the JWT. Account for this when designing applications that anticipate users belonging to many groups.

## Enforcing Group-Scoped Access on Other Models

Once `Group` and `GroupMembership` are deployed, you can restrict any other model so that records are only readable/writable by the group's owner or members.

The key is to:

1. Add a `groupId` field to the model.
2. Set `accessTokenAuthField` to either `groupOwnerships` (owner-only) or `groupMemberships` (any member).
3. Set `accessModelAuthField` to `groupId`.

### Pattern A: Group Owner-Only (no shared access)

Only the user who owns the group can create/read/update/delete records belonging to it.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessTokenAuthField": "groupOwnerships",
    "accessModelAuthField": "groupId",
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

### Pattern B: Group-Wide Shared Access (members + owner)

Any member of the group (including the owner) can perform all CRUD operations on records linked to that group.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessTokenAuthField": "groupMemberships",
    "accessModelAuthField": "groupId",
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

### Pattern C: Members Read, Owner Writes

Members of a group can read records, but only the group owner may create/update/delete them. Use action-specific token fields:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessModelAuthField": "groupId",
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessCreateTokenAuthField": "groupOwnerships",
    "accessReadTokenAuthField": "groupMemberships",
    "accessUpdateTokenAuthField": "groupOwnerships",
    "accessDeleteTokenAuthField": "groupOwnerships"
  }'
```

### Pattern D: Mixing per-Account and per-Group Ownership

A record can have both an `accountId` (the individual creator) and a `groupId` (the group it belongs to). Use field-level access control or action-specific auth fields to combine both. For example, allow the original author to edit while letting any group member read:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessReadTokenAuthField": "groupMemberships",
    "accessReadModelAuthField": "groupId",
    "accessUpdateTokenAuthField": "accountId",
    "accessUpdateModelAuthField": "accountId",
    "accessDeleteTokenAuthField": "accountId",
    "accessDeleteModelAuthField": "accountId"
  }'
```

## Filtering Group-Scoped Views in the Frontend

When a record belongs to a group, frontend components must filter by the active `groupId` (and the access rules will reject any request that asks for records the user has no membership in).

```html
<collection-viewer
  collection-type="Document"
  collection-fields="title,content,groupId,accountId"
  collection-view="groupView"
  collection-view-primary-fields="groupId"
  collection-view-params="groupId={{currentGroupId}}"
  collection-page-size="20"
>
  <template slot="item">
    <div class="card">
      <h3>{{Document.title}}</h3>
      <p>{{Document.content}}</p>
    </div>
  </template>
  <div slot="viewport"></div>
</collection-viewer>
```

The corresponding `ModelView` on `Document` should filter by `groupId` so the access layer can verify that the requesting user has the matching `groupId` in their JWT (`groupOwnerships` or `groupMemberships` depending on your configuration). See [search-filtering-querying.md](search-filtering-querying.md) for view setup details.

## Realtime Reaction to Membership Changes

When a `Group` or `GroupMembership` record is created/updated/deleted, the Saasufy service publishes a message on the channel:

```
account-access-change/{accountId}
```

…for every account whose `groupOwnerships` or `groupMemberships` changed as a result. The frontend can subscribe to this channel to refresh the JWT (so the user picks up new group access without logging out and back in). The standard Saasufy auth components handle this automatically.

## Workflow

1. **Create the `Group` model** (admin UI Authentication page is the easiest path; manual `curl` shown above).
2. **Create the `GroupMembership` model** with `accessTokenAuthField=groupOwnerships` and `accessModelAuthField=groupId`.
3. **Deploy the service** so the Saasufy service worker attaches the built-in schema extensions and JWT fields.
4. **Have users re-authenticate** so their JWTs gain the `groupOwnerships` and `groupMemberships` fields.
5. **Configure your other models** with `accessTokenAuthField` set to `groupOwnerships` or `groupMemberships` and `accessModelAuthField=groupId`.
6. **Add a `groupId` field** to each model that should be group-scoped, and set up a `ModelView` that filters by `groupId` for the frontend.
7. **Deploy** again whenever you change any of the schema or access settings.

## Important Notes

1. **The built-in service rules are enforced even if standard model-level rules disagree.** You cannot make `GroupMembership.create` open to all authenticated users by setting `accessCreate=allow`; the service rejects non-owner inserts regardless.
2. **Group owners are implicit members.** `groupMemberships` includes both ownership groups and explicit `GroupMembership` rows, deduplicated.
3. **Adding the user to their own group** is unnecessary — owners are automatically counted in `groupMemberships`. Do not insert a `GroupMembership` row for the owner.
4. **JWT updates require re-authentication or a token-refresh flow.** When you create a new `Group` or add a user via `GroupMembership`, the affected user must obtain a refreshed JWT before the new permissions take effect. Subscribe to `account-access-change/{accountId}` to trigger this automatically in the frontend.
5. **JWT size limits.** Up to 250 ownerships and 250 memberships per user. Beyond that, only the most recently updated groups appear in the JWT.
6. **`groupId` and `accountId` on `GroupMembership` are required UUIDs** — validate inputs in the frontend before submitting to avoid 400-class errors.
7. **Always read the API key from `.saasufy-api-key` first.**
8. **After any schema or access control change, ask the user to deploy** by clicking the *Deploy service* button on their Saasufy Dashboard.

## Troubleshooting

### "Only the group owner can add members to the group"
The user creating a `GroupMembership` does not own the target `Group`. Verify:
- The `Group` record's `accountId` matches the requesting user's `accountId`.
- The user's JWT was issued *after* the `Group` was created (otherwise `groupOwnerships` may not include it). Re-authenticate.

### "The operation was not permitted" on Group update/delete
The user is not the owner. Same checks as above.

### Members can read group records but not modify them
This is expected if you used `accessTokenAuthField=groupMemberships` for reads but `groupOwnerships` for writes. Switch the write fields to `groupMemberships` (Pattern B) if you want any member to write.

### Added a user via `GroupMembership` but they still can't see records
Their JWT has not refreshed yet. They need to re-authenticate, or the frontend must listen on `account-access-change/{accountId}` to trigger a token refresh. See the auth components in [log-in-form.md](log-in-form.md) and the `socket-provider` reauthentication flow.

### "Some of the specified group fields cannot be updated"
You attempted to update one of the reserved fields: `id`, `createdAt`, `updatedAt`, `updatedByIp`. Remove them from the update payload.

### Group membership not appearing in `groupMemberships`
Check that the `GroupMembership` record was actually created. Also, note that a user cannot be a member of more than 250 groups; confirm the user has not exceeded the 250-membership cap — if they have, only the 250 most recently updated groups appear in the JWT.
