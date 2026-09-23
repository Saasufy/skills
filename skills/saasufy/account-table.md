# Saasufy Account Table

## When to Use This Skill

Use this skill when you need to:
- Read or display the logged-in user's profile (username, email, signup source)
- Store extra per-user data (a display name, avatar, preferences) alongside their account
- Let users deactivate their own account
- Build a view over accounts (e.g. look a user up by username or email)

The `Account` table is created and populated by Saasufy's auth layer, not by your app. It is not visible to your service or to frontend components until you *expose* it, which is what this guide covers. For the `Group`/`GroupMembership` equivalents, see [access-control-groups.md](access-control-groups.md).

## Authentication

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## Concept Overview

Every time a user authenticates — via blockchain, OAuth, an external JWT or admin credentials — Saasufy upserts a record in your service's `Account` table, keyed by the user's `accountId` (the same value that appears as `socket.authToken.accountId`). The service owns this table:

- **Records are never created by clients.** A `create` action on `Account` is always rejected with *"The operation was not supported"*; records appear only as a side effect of a successful login.
- **The service maintains a fixed set of fields on it** (`id`, `username`, `email`, `authSource`, `walletAddress`, `lastWalletBalance`, `lastIpAddress`, `isDeactivated`, `createdAt`, `updatedAt`, `updatedByIp`) and keeps database indexes on `username`, `email`, `walletAddress`, `lastIpAddress`, `createdAt`, `updatedAt` and `updatedByIp` regardless of what you declare.
- **These fields are frozen** and cannot be written by a client: `id`, `username`, `walletAddress`, `authSource`, `lastWalletBalance`, `lastIpAddress`, `createdAt`, `updatedAt`, `updatedByIp`. A client may update `email`, `isDeactivated` and any custom fields you add.
- **Update and delete are restricted to the record's owner** (`socket.authToken.accountId === record.id`), except for admins. Deleting a whole `Account` record is admin-only; a non-admin owner may only clear individual non-frozen fields.
- **`isDeactivated`** is checked at login: a deactivated account is refused with *"Account is deactivated"*.

Exposing the table is therefore only about making it readable and extendable from your app — you declare the `Model`, the `ModelField` records for the fields you want your app to see, and the `ModelIndex` records for the lookups you want views to use.

## Step 1: Expose the Account Table

Create a `Model` named exactly `Account`:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/Model' \
  -d '{"name": "Account", "position": 0}'
```

Note the response's `id` — it is the `{ACCOUNT_MODEL_ID}` used below.

Because the name is `Account`, Saasufy applies safer defaults than for a normal model: `accessModelAuthField` defaults to `"id"` (the record's own ID is the ownership field, since there is no `accountId` field on an account) and `accessRead` defaults to `"restrict"`, so each user can read only their own account record. Leave both alone unless you intend accounts to be publicly readable; if you do want a public directory, prefer exposing only the fields you want public and setting `accessRead` on the model to `"allow"` while restricting sensitive fields individually (see [access-control.md](access-control.md)).

## Step 2: Expose the Standard Fields

Create one `ModelField` per field you want your app to read. These nine are the standard set (the same ones the control panel's **Expose standard fields** button on the model's `/fields` page creates), with the types and positions Saasufy uses:

```bash
ACCOUNT_MODEL_ID={ACCOUNT_MODEL_ID}

for FIELD in \
  'id string 0' \
  'username string 1' \
  'email string 2' \
  'authSource string 3' \
  'lastWalletBalance number 4' \
  'lastIpAddress string 5' \
  'isDeactivated boolean 6' \
  'createdAt number 7' \
  'updatedAt number 8'
do
  set -- $FIELD
  curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
    -H "Content-Type: application/json" \
    -XPOST 'https://saasufy.com/api/ModelField' \
    -d "{\"modelId\": \"$ACCOUNT_MODEL_ID\", \"name\": \"$1\", \"type\": \"$2\", \"position\": $3}"
done
```

| Field | Type | Meaning |
|-------|------|---------|
| `id` | string | The account ID; matches `socket.authToken.accountId` |
| `username` | string | Wallet address, or `{name}.{hash}.{provider}` for OAuth logins |
| `email` | string | Set from the OAuth provider's email field, where one is configured |
| `authSource` | string | `blockchain`, `external`, `simple`, or the OAuth provider name |
| `lastWalletBalance` | number | Balance at last blockchain login |
| `lastIpAddress` | string | IP address of the last login |
| `isDeactivated` | boolean | When true, login is refused |
| `createdAt` | number | Signup timestamp (ms) |
| `updatedAt` | number | Last login/update timestamp (ms) |

Two further service-maintained fields can be exposed the same way if your app needs them: `walletAddress` (string) and `updatedByIp` (string).

Do not declare constraints (`required`, `min`, `email`, …) on these fields — the service already validates them, and a conflicting constraint can make a legitimate login fail. You can add your own extra fields (e.g. `displayName`, `avatarUrl`) to the same model as normal `ModelField` records; those are writable by the owning user under the rules above.

## Step 3: Expose the Standard Indexes

The service keeps the underlying database indexes in place either way, but declaring them as `ModelIndex` records is what makes them visible in the schema and safe to reference from a `ModelView`'s `transformIndex`. The standard pair (matching the **Expose standard indexes** button on the model's `/indexes` page) is `username` and `email`:

```bash
for INDEX in username email; do
  curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
    -H "Content-Type: application/json" \
    -XPOST 'https://saasufy.com/api/ModelIndex' \
    -d "{\"modelId\": \"$ACCOUNT_MODEL_ID\", \"name\": \"$INDEX\", \"fields\": \"$INDEX\"}"
done
```

These let you build views that look an account up by username or email — for example a view with `paramFields: "username"`, `primaryFields: "username"` and `transformIndex: "username"`. See [views-and-indexing.md](views-and-indexing.md) before declaring any view.

## Step 4: Deploy

Schema changes take effect only after a deployment. See [deployment-and-testing.md](deployment-and-testing.md).

## Using the Account Record in the Frontend

With `accessRead` left at `restrict`, a user can read their own record directly by ID — `socket.authToken.accountId` is both the token field and the record's `id`:

```html
<model-viewer
  model-type="Account"
  model-id="{{socket.authToken ? socket.authToken.accountId : ''}}"
  model-fields="username,email,createdAt"
>
  <template slot="item">
    <div>{{Account.username}}</div>
    <div>{{Account.email}}</div>
  </template>
</model-viewer>
```

To let a user edit their own non-frozen fields, bind a `model-input` to the same `model-type`/`model-id`. See [model-viewer.md](model-viewer.md) and [model-input.md](model-input.md).

## Common Pitfalls

1. **Trying to create Account records.** Signup happens through authentication only; there is no client-side create. See [authentication.md](authentication.md).
2. **Adding constraints to the standard fields.** The service validates them already; a stricter constraint (e.g. `required` on `email` when the OAuth provider gives none) will break logins.
3. **Renaming the model.** The name must be exactly `Account` — the auth layer writes to that table name, and the safer access defaults are keyed off it.
4. **Expecting `accountId` on the model.** There is no `accountId` field; ownership is the record's own `id`, which is why `accessModelAuthField` is `"id"`.
5. **Trying to update a frozen field.** Writing `username`, `authSource`, `lastIpAddress` etc. fails with *"Some of the specified account fields cannot be updated"*.
