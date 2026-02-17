# Saasufy Access Control

## When to Use This Skill

Use this skill when you need to:
- Configure who can create, read, update, or delete records in a Model
- Set up token-based or model-based authentication for access control
- Enable multiple owners, creators, readers, updaters, or deleters for a Model
- Configure field-level access control for sensitive data
- Set up default access permissions

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Base URL

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## Access Control Overview

Saasufy provides two levels of access control:
1. **Model-level access control**: Controls access to entire records in a Model
2. **Field-level access control**: Controls access to specific fields within records

### Access Modes

Access control properties use an enum with the following possible values:
- `"allow"` - Allow all users to perform this operation without restrictions
- `"block"` - Block all users from performing this operation
- `"restrict"` - Restrict access based on authentication fields and ownership rules (configured via `accessTokenAuthField`, `accessModelAuthField`, and related properties)

## Model-Level Access Control

### General Access Properties

These properties control overall access to the Model:

- `accessTokenAuthField` (string): Field name in the user's JWT token used for authentication (default: "accountId")
- `accessModelAuthField` (string): Field name in the Model record used for authentication/ownership verification (default: "accountId"; if it exists)
- `accessEnableMultipleOwners` (boolean): Allow multiple users to own a record
- `accessAllowByDefaultOwners` (boolean): Allow all users as owners by default
- `accessCreate` (enum): Who can create records
- `accessRead` (enum): Who can read records
- `accessUpdate` (enum): Who can update records
- `accessDelete` (enum): Who can delete records

### Action-Specific Access Properties

These properties provide fine-grained control over specific actions:

- `accessEnableMultipleCreators` (boolean): Allow multiple creators
- `accessAllowByDefaultCreators` (boolean): Allow all users as creators by default
- `accessEnableMultipleReaders` (boolean): Allow multiple readers
- `accessAllowByDefaultReaders` (boolean): Allow all users as readers by default
- `accessEnableMultipleUpdaters` (boolean): Allow multiple updaters
- `accessAllowByDefaultUpdaters` (boolean): Allow all users as updaters by default
- `accessEnableMultipleDeleters` (boolean): Allow multiple deleters
- `accessAllowByDefaultDeleters` (boolean): Allow all users as deleters by default

### Action-Specific Authentication Fields

These properties specify authentication fields for each action:

- `accessCreateTokenAuthField` (string): Token field for create operations
- `accessReadTokenAuthField` (string): Token field for read operations
- `accessUpdateTokenAuthField` (string): Token field for update operations
- `accessDeleteTokenAuthField` (string): Token field for delete operations
- `accessCreateModelAuthField` (string): Model field for create operations
- `accessReadModelAuthField` (string): Model field for read operations
- `accessUpdateModelAuthField` (string): Model field for update operations
- `accessDeleteModelAuthField` (string): Model field for delete operations

### Update Model Access Control

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{"accessCreate": "allow", "accessRead": "allow", "accessUpdate": "restrict", "accessDelete": "restrict"}'
```

### Example: Set Up Owner-Based Access Control

This example allows anyone to create and read records, but restricts updates and deletes to the owner. The `accessTokenAuthField` specifies the field in the user's JWT token (typically "accountId"), and `accessModelAuthField` specifies the field in the record that must match for restricted access to be granted.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessTokenAuthField": "accountId",
    "accessModelAuthField": "ownerId",
    "accessCreate": "allow",
    "accessRead": "allow",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

### Example: Enable Multiple Owners

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessEnableMultipleOwners": true,
    "accessAllowByDefaultOwners": false
  }'
```

### Example: Configure Action-Specific Access

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessEnableMultipleReaders": true,
    "accessAllowByDefaultReaders": true,
    "accessEnableMultipleUpdaters": true,
    "accessAllowByDefaultUpdaters": false
  }'
```

## ModelField-Level Access Control

Field-level access control allows you to restrict access to specific fields within a Model. This is useful for sensitive data like email addresses, phone numbers, or personal information.

### Field Access Properties

- `accessCreate` (enum): Who can set this field when creating records
- `accessRead` (enum): Who can read this field
- `accessUpdate` (enum): Who can update this field
- `accessDelete` (enum): Who can delete this field (for multi-valued fields)

### Update Field Access Control

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{FIELD_ID}' \
  -d '{"accessRead": "restrict", "accessUpdate": "restrict"}'
```

### Example: Protect Sensitive Field

Make a field (e.g., email or phone number) only readable and editable by the owner:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{FIELD_ID}' \
  -d '{
    "accessCreate": "allow",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": null
  }'
```

### Example: Public Read, Owner Write

Allow anyone to read a field but only owners to modify it:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{FIELD_ID}' \
  -d '{
    "accessRead": "allow",
    "accessUpdate": "restrict"
  }'
```

### Example: Admin-Only Field

Create a field that only specific users can read or modify:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{FIELD_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'
```

## Common Access Control Patterns

### Pattern 1: Public Read, Restricted Write

Use case: Blog posts, product listings where anyone can read but only the author can modify

```bash
# Model level
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "allow",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessTokenAuthField": "accountId",
    "accessModelAuthField": "authorId"
  }'
```

### Pattern 2: Owner-Only Access

Use case: Private user data, personal documents (only the account owner can access their own records)

```bash
# Model level
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessTokenAuthField": "accountId",
    "accessModelAuthField": "accountId"
  }'
```

### Pattern 3: Collaborative Documents

Use case: Shared documents with multiple editors

```bash
# Model level
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "restrict",
    "accessRead": "restrict",
    "accessUpdate": "restrict",
    "accessDelete": "restrict",
    "accessEnableMultipleReaders": true,
    "accessAllowByDefaultReaders": false,
    "accessEnableMultipleUpdaters": true,
    "accessAllowByDefaultUpdaters": false
  }'
```

### Pattern 4: Mixed Field-Level Security

Use case: User profiles with public and private fields

```bash
# Model level - public read, owner write
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/Model/{MODEL_ID}' \
  -d '{
    "accessCreate": "allow",
    "accessRead": "allow",
    "accessUpdate": "restrict",
    "accessDelete": "restrict"
  }'

# Field level - email field is owner-only
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{EMAIL_FIELD_ID}' \
  -d '{
    "accessRead": "restrict",
    "accessUpdate": "restrict"
  }'

# Field level - name field is public
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/api/ModelField/{NAME_FIELD_ID}' \
  -d '{
    "accessRead": "allow",
    "accessUpdate": "restrict"
  }'
```

## Workflow for Setting Up Access Control

1. **Identify the access pattern** you need (public, owner-based, role-based, etc.)
2. **Get the Model ID** by listing all models
3. **Update Model-level access control** with appropriate settings
4. **Get Field IDs** for any fields requiring special access control
5. **Update Field-level access control** for sensitive or restricted fields
6. **Deploy the changes** by asking the user to click the `Deploy service` button on their Dashboard page at saasufy.com

## Important Notes

1. **Always read the API key from `.saasufy-api-key` file first**
2. **Field-level access control** overrides Model-level settings for that specific field
3. **Setting a value to `null`** removes that specific access control constraint
4. **Token auth fields** must match field names in the user's JWT token (default field is "accountId")
5. **Model auth fields** must match field names in the Model being accessed
6. **Multiple owners/readers/updaters/deleters** requires enabling the corresponding flag
7. After configuring access control, always ask the user to **Deploy the schema changes** by clicking on the `Deploy service` button on their Dashboard page at saasufy.com

## Troubleshooting

### Access Denied Errors

If users are getting access denied errors:
1. Verify the `accessTokenAuthField` matches the token field name
2. Verify the `accessModelAuthField` matches the Model field name
3. Check that the access mode enum values are correct
4. Ensure the schema has been deployed after changes

### Field Not Visible

If a field is not appearing in responses:
1. Check the field's `accessRead` setting
2. Verify the user has the required role or ownership
3. Check if field-level access control is more restrictive than Model-level

### Filtering Results By Account ID

You can filter based on an account ID by specifying an `accountId` property to the relevant component's attribute; for example by adding the relevant `socket.authToken.accountId` value to the `collection-view-params` attribute of the `collection-viewer` component. Note that the `socket` object can be accessed inside template expressions with double or triple curly braces. Relevant access controls will be enforced by matching the `accountId` from the `socket.authToken` against the `accountId` view param passed to the view.

```html
<collection-viewer
  class="longlist-viewer"
  collection-type="Longlist"
  collection-fields="createdAt,accountId"
  collection-view="accountSearchView"
  collection-view-primary-fields="accountId"
  collection-view-params="accountId={{socket.authToken ? socket.authToken.accountId : ''}},query="
  collection-page-size="10"
  auto-reset-page-offset
>
  <template slot="item">
    <div class="card{{Longlist.accountId && Longlist.accountId.includes(',') ? ' card-shared' : ''}}">
      <!-- CONTENT -->
    </div>
  </template>

  <template slot="no-item">
    <div class="container-vertical container-centered-cross container-centered-main" style="height: 100px;">No projects were found.</div>
  </template>

  <div slot="viewport"></div>
</collection-viewer>
```