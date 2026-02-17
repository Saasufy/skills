# Overview

Queries are passed to Saasufy views via `viewParams`.
For example, the `collection-viewer` element has a `collection-view-params` attribute which represents a comma-separated list of parameters with their values. A query should be passed by itself or alongside other parameters like this: `companyEmployeeCountLow=10,query='companyName contains Micro ~AND~ yearFounded < 2000'`. Any parameter can potentially hold a query; the correct parameter to use depends how the view is defined within Saasufy.

When creating a `ModelView` in Saasufy, a `transformFilterQuery` field can optionally be specified to transform it.
To access the `query` property of the `viewParam` as shown in the earlier example, the `transformFilterQuery` would have to be set to the string `$paramFields.query`.

The query is applied during the second phase of filtering and can be passed in along with other `viewParams`.
Sometimes, in order to get optimal performance with indexed filtering, it may be useful to also set the `transformIndex`, `transformIndexOperation` and `transformIndexOperationInputA` using other `$paramFields` passed into the view through `viewParams`.

For example, consider this sample `ModelView` which combines indexed filtering with a flexible query:

```json
{
  "affectingFields": "companyDescription, companyName",
  "id": "dd30dd37-c73b-4d56-9753-c08828dba606",
  "modelId": "9666386a-55a1-46ba-bc5a-d79775e44347",
  "name": "searchView",
  "paramFields": "companyEmployeeCountLow, query",
  "primaryFields": "companyEmployeeCountLow",
  "transformFilterQuery": "$paramFields.query",
  "transformFilterType": "advanced",
  "transformIndex": "companyEmployeeCountLow",
  "transformIndexOperation": "equals",
  "transformIndexOperationInputA": "$paramFields.companyEmployeeCountLow",
  "transformOrderByField": "companyName"
}
```

# Query Format

## Basic Structure

The query follows the format:

```
fieldName1 comparisonOperator1 value1 ~booleanOperator~ comparisonOperator2 operator2 value2
```

Quotation marks are not required around strings, however it Saasufy queries are very strict in terms of spacing. There must be exactly one space between each term in the query. String values can have spaces inside them and they are treated as part of the string; everything is determined based on the position of the operators.

Supported comparison operators are: `=`, `>`, `<`, `>=`, `<=`, `contains` and `excludes`.
For `contains` and `excludes`, the value will be treated as a regular expression in the JavaScript format; it can optionally start with `(?i)` to indicate that it should be case-insensitive, for example:

```
companyName contains (?i)Micro ~AND~ yearFounded < 2000
```

Supported boolean operators are `~AND~` and `~OR~` as well as variants which can be used to adjust priority (which act like brackets).

## Complex Boolean Expressions

It is possible to construct complex boolean queries by adding a number within the comparison operator in the form `~ANDx~` or `~ORx~`; the greater the `x`, the higher the priority. If no `x` is specified, then the priority is treated as `0`.

For example, in the following case, the `OR` boolean operator has higher priority than the `AND`:

```
yearFounded <= 2000 ~AND~ companyName contains (?i)Micro ~OR1~ companyName contains (?i)General Moto
```

This query would only include companies founded on or before the year 2000 and whose companyName starts with either 'Micro' or 'General Moto' (case insensitive).
