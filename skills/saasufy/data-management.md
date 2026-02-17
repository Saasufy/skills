# Saasufy Data Management Skill

## When to Use This Skill

Use this skill when you need to:
- Create records in a user-defined Model collection
- Read records from a Model collection
- Update existing records
- Delete records
- Query records using predefined Views

**Important:** Before using this skill, you should understand the schema of the Model you're working with. Use the `saasufy-schema.md` skill to read the Model schema first to know which fields are available and what constraints exist.

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Service API Endpoint

The Service API endpoint depends on your deployed Saasufy service.
This URL is stored inside the `.saasufy-service-url` file at the root of the project. If it is not provided, the user should deploy their service and provide it to you; you should then add it to that file (replacing the default one which is a dummy URL containing the path `/sidxxxx/`).

**Note:** The examples in curl-commands.md use `https://saasufy.com/sid8006/api/` for local development. Adjust the base URL based on where your service is deployed.

## Understanding Models

A Model is like a database table. For example, if you have a "Product" Model with fields:
- `name` (string, required)
- `price` (number, required)
- `description` (string)
- `inStock` (boolean)

You would interact with it at the `/api/Product` endpoint.

## Reading the Schema First

**Always read the schema first** to understand what fields exist and their constraints:

```bash
# Get the Model definition
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Model?view=accountAlphabeticalView'

# Get the fields for a specific Model
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelField?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'

# Get the views for a specific Model
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelView?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'
```

## Data Operations

Replace `{ModelName}` with the actual name of your Model (e.g., "Product", "User", "Order").

### Get a Record by ID

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET '{SERVICE_URL}/api/{ModelName}/{RECORD_ID}'
```

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/sid8006/api/Product/9fed41a0-caae-4620-8ed1-931a3b388fa3'
```

### Get a List of Records Using a View

Views allow you to filter, sort, and query records. Use the view name defined in your ModelView.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET '{SERVICE_URL}/api/{ModelName}?view={viewName}'
```

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/sid8006/api/Product?view=defaultView'
```

With view parameters:
```bash
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/sid8006/api/Product?view=inStockView&viewParams[inStock]=true'
```

### Create a New Record

The fields you include in the JSON body must match the ModelFields defined in your schema. Pay attention to:
- Required fields
- Field types (string, number, boolean)
- Constraints (min, max, email, enum, etc.)

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST '{SERVICE_URL}/api/{ModelName}' \
  -d '{FIELD_DATA_JSON}'
```

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/sid8006/api/Product' \
  -d '{"name": "Laptop", "price": 999.99, "description": "High-performance laptop", "inStock": true}'
```

### Update a Record

Only include the fields you want to update. Other fields will remain unchanged.

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT '{SERVICE_URL}/api/{ModelName}/{RECORD_ID}' \
  -d '{UPDATED_FIELD_DATA_JSON}'
```

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPUT 'https://saasufy.com/sid8006/api/Product/9124ac66-a493-4092-8cbd-3f2c8dd8d65e' \
  -d '{"price": 899.99, "inStock": false}'
```

### Delete a Record

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE '{SERVICE_URL}/api/{ModelName}/{RECORD_ID}'
```

Example:
```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XDELETE 'https://saasufy.com/sid8006/api/Product/9124ac66-a493-4092-8cbd-3f2c8dd8d65e'
```

## Important Notes

1. **Always read the API key from `.saasufy-api-key` file first**
2. **Know your service URL** - adjust the base URL based on your deployment
3. **Read the schema first** - understand the Model structure before creating/updating records
4. **Validate data** - ensure your data matches the field types and constraints
5. **Use Views** - for efficient querying and filtering
6. **UUIDs** - Record IDs are UUIDs (version 4 format)
7. **Error handling** - Pay attention to validation errors returned by the API

## Common Workflows

### Creating Records in a New Model

1. Read the Model schema to understand fields and constraints
2. Prepare JSON data that matches the schema
3. POST the data to create a new record
4. Verify the record was created (optional: GET by ID)

### Querying Data

1. Check available Views for the Model
2. Use the appropriate View with any required viewParams
3. GET the records using the View

### Bulk Operations

For multiple records:
1. Read the schema once
2. Iterate through your data
3. Create/Update each record with individual API calls

## Example: Complete Workflow

```bash
# 1. Read API key
SAASUFY_API_KEY=$(cat .saasufy-api-key)

# 2. Check the schema to understand the "Message" model
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/Model?view=accountAlphabeticalView' | jq '.[] | select(.name=="Message")'

# 3. Get fields for the Message model (using the model ID from step 2)
curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/api/ModelField?view=accountModelAlphabeticalView&viewParams[modelId]=MODEL_ID_HERE'

# 4. Create a new message record
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/sid8006/api/Message' \
  -d '{"message": "Hello World", "from": "Alice"}'

# 5. Query messages using a view
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -XGET 'https://saasufy.com/sid8006/api/Message?view=defaultView'
```

## Tips for No-Code Experience

When helping users manage their data:
1. **Ask about their Model structure first** - "What fields does your {ModelName} have?"
2. **Offer to create records interactively** - "I can help you add a new {ModelName}. What values should I use?"
3. **Show them the results** - After creating/updating, fetch and display the record
4. **Suggest useful views** - Help them query data efficiently
5. **Validate before submitting** - Check that required fields are provided and types match
