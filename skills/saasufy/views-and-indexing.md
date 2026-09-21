# Views and Indexing

Read this guide before creating a `ModelView` or `ModelIndex`, especially when using a compound index (an index over more than one field) or the `between` index operation.

## How a View Produces Its Results

A `ModelView` resolves a query in two phases:

1. **Index phase.** Saasufy scans the index named by `transformIndex` using `transformIndexOperation` (`equals` or `between`) with the bounds given by `transformIndexOperationInputA` and `transformIndexOperationInputB`. This is the only phase which uses an index, and it determines the set of records the view can return.
2. **Filter phase.** Those records are then narrowed further by `transformFilterQuery` (or by `transformFilterField`/`transformFilterOperation`/`transformFilterOperationInput`). See [search-filtering-querying.md](search-filtering-querying.md) for the query syntax.

Results are then ordered by `transformOrderByField`.

## Index Names Are Assigned by Saasufy

Saasufy derives each index's name from its `fields`, in order, camelCased; a `name` supplied at creation is replaced on deployment:

```
fields: "status,priceAmount"    → name: "statusPriceAmount"
fields: "clinicianId,startAt"   → name: "clinicianIdStartAt"
```

Refer to an index by this canonical name in a view's `transformIndex`. If `transformIndex` names neither an existing field nor an existing index, Saasufy creates a placeholder index over a field of that name; where no such field exists, the index matches nothing and the view returns no results.

## Let Saasufy Create Single-Field Indexes

For a single-field `transformIndex`, name the **field**; the index is created on the next deployment:

```jsonc
// No ModelIndex call needed.
"transformIndex": "priceAmount",
"transformIndexOperation": "equals",
"transformIndexOperationInputA": "$paramFields.priceAmount"
```

Saasufy also creates indexes for `multi` fields automatically. `multi` fields are indexed element-wise, so an `equals` lookup against a single element works well.

Compound indexes are the exception — they are not inferred from a view, so create a `ModelIndex` for each one, specifying `fields` only:

```bash
curl -H "Authorization:Bearer $SAASUFY_API_KEY" \
  -H "Content-Type: application/json" \
  -XPOST 'https://saasufy.com/api/ModelIndex' \
  -d '{"modelId": "{MODEL_ID}", "fields": "clinicianId,startAt"}'
```

Then set `transformIndex` to the canonical name (`clinicianIdStartAt` here). If you are unsure of the name, deploy first and read it back from the API.

## Compound Index Operations Take Complete Keys

`transformIndexOperationInputA` and `transformIndexOperationInputB` each specify a complete index key, not a single value. A `between` scan therefore runs from one full key to another: for an index over `(clinicianId, startAt)`, from `(clinicianId, fromAt)` to `(clinicianId, toAt)`. The leading components are repeated in both bounds and only the final component differs:

```jsonc
"transformIndex": "clinicianIdStartAt",
"transformIndexOperation": "between",
"transformIndexOperationInputA": "$paramFields.clinicianId,$paramFields.fromAt",
"transformIndexOperationInputB": "$paramFields.clinicianId,$paramFields.toAt"
```

Each input must supply a value for every component of the index, comma-separated in index order (no spaces around the commas). Supplying fewer values than the index has components returns no results. The same applies to `equals` on a compound index.

## `between` Is Half-Open: `[from, to)`

A record whose key equals `from` is included; one whose key equals `to` is excluded. Pass `to = end + 1` where an inclusive upper bound is intended — a "midnight to midnight" day window otherwise drops anything landing exactly on the closing instant.

## Params Only Filter When Consumed

Declaring a name in `paramFields` has no filtering effect on its own, even when it matches a field on the model. A param filters only if it is consumed — by an index input, or by a reference inside `transformFilterQuery` or `transformFilterOperationInput`.

```jsonc
// clinicianId is declared but never consumed, so this returns every
// clinician's rows rather than just the requested one.
"paramFields": "clinicianId,fromAt,toAt",
"transformIndex": "startAt",
"transformIndexOperation": "between",
"transformIndexOperationInputA": "$paramFields.fromAt",
"transformIndexOperationInputB": "$paramFields.toAt"
```

Where a param scopes results to a tenant, owner or account, prefer binding it to an index component so that it is applied in the first phase.

## Recommended Workflow

1. Create the Model and its Fields.
2. Create the Views. For a single-field `transformIndex`, name the field.
3. Create a `ModelIndex` only for compound indexes, specifying `fields` only, and reference each from a view by its canonical name.
4. Deploy.
5. Check that every index's `fields` names real fields on the model, and that each view returns what you expect for a query which cannot exclude anything (for example an unfalsifiable range such as `fromAt=0`, `toAt=9999999999999`, with a known-good value for each leading component).
