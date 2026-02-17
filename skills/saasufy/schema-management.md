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

## Model Management

Models represent collections (like database tables) in your Saasufy service.

### List All Models

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
- `paramFields` (string, comma-separated field names used as parameters)
- `primaryFields` (string, comma-separated primary index fields)
- `affectingFields` (string, fields that affect the view results)
- `transformIndex` (string, index to use for the transform)
- `transformIndexOperation` (string, can be "equals" or "between")
- `transformIndexOperationInputA`: (string, typically referencing a viewParam passed into the view from the frontend; for example "$paramFields.companyEmployeeCountLow")
- `transformIndexOperationInputB`: (string, represents the second operand if the "between" index operation is used)
- `transformOrderByField` (string, field to sort by)
- `transformOrderByDesc` (boolean, descending order)
- `transformFilterType` (string, filter type)
- `transformFilterField` (string, field to filter on)
- `transformFilterQuery` (string, advanced query to filter records for this view; can be set to "$paramFields.query" to provide maximum flexibility to allow the frontend to construct the query)
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

Indexes improve query performance on specific fields.

### List All Indexes for a Model

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
- `name` (string, index name)
- `fields` (string, comma-separated field names to index)

Optional fields:
- `format` (string, index format)
- `maxCardinality` (integer, max cardinality for the index)

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelIndex' \
  -d '{"modelId": "{MODEL_ID}", "name": "emailIndex", "fields": "email"}'
```

Compound index:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelIndex' \
  -d '{"modelId": "{MODEL_ID}", "name": "statusDateIndex", "fields": "status,createdAt"}'
```

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

## Common Workflows

### Create a Complete Model Schema

1. Create the Model
2. Create Fields for the Model
3. Create Views for querying the Model
4. Create Indexes for performance
5. After making schema changes, you must deploy/start the service for the changes to take effect.

### Modify an Existing Schema

1. List Models to find the one to modify
2. List Fields/Views/Indexes for that Model
3. Update or create new Fields/Views/Indexes as needed
4. After making schema changes, you must deploy/start the service for the changes to take effect.
