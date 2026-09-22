# Debugging and Troubleshooting

## Common issues

### Most likely issues

If you encounter any issues with Saasufy, the first thing to do is to double-check the current state of the Saasufy control panel via HTTP API. If there is a problem, always check your schema first before doing any debugging: 

- Check that your views and parameters are defined correctly.
- Check that your models are defined correctly and that the fields exists with the correct types and constraints.
- Check that the access rules are suitable, etc...

Complex technical issues should be rare in Saasufy. Assume the issue is with the way your schema is defined or with your data.

### Race conditions related to mixing imperative code and declarative markup

Try using or creating declarative custom HTML elements (Web Components) as much as possible to avoid race conditions which can happen when calling functions imperatively. Note that some components such as the `app-router` implement debouncing when the URL is changed for efficiency reasons.
These issues can be avoided by using or creating Web Components and letting the parent component decide when to render (and initialize) the child element as opposed to using setTimeout to solve these issues which is a hacky approach and not recommended.

### Input field losing focus on update

There could be a conflict between the `collection-viewer` element and the child `model-input` element.
The `collection-fields` attribute of the `collection-viewer` element represents a list of fields which the collection viewer watches for relevant updates. Whenever any of these fields are updated in a way which affects the `collection-viewer`, it will re-render its viewport and its children.
When such a field is modified via a nested `model-input` element, the element will lose focus upon re-render. To fix this, you should remove the affected field from the `collection-fields` attribute of the parent `collection-viewer`; this ensures that the re-rendering is limited to the `model-input` element and not the entire `collection-viewer`.

For example, the following code will cause a loss of focus when editing the `title` field via the child `model-input` element because the `title` field is referenced both in the `collection-viewer` and also in the child `model-input` element:

```html
<collection-viewer
  collection-type="Todo"
  collection-fields="title,status,assignedTo"
  collection-view="allTodosView"
  collection-view-params=""
  collection-view-primary-fields=""
  collection-page-size="50"
>
  <template slot="item">
    <div class="todo-item">
      <div>{{Todo.status}}></div>
      <model-input
        type="text"
        model-type="Todo"
        model-id="{{Todo.id}}"
        model-field="title"
        class="todo-title {{Todo.status ? 'completed' : ''}}"
      ></model-input>
    </div>
    </template>

    <template slot="no-item">
    <div class="empty-state">
      No TODOs yet. Add one above to get started!
    </div>
  </template>

  <div slot="viewport" class="todo-list"></div>
</collection-viewer>
```

To correct this, the `title` field should be removed from the `collection-fields` attribute of the parent `collection-viewer`.
Note that the `assignedTo` field is also not needed in this case because it is not being referenced anywhere within that component and can lead to unnecessary re-rendering of the `collection-viewer`.

### Realtime update not happening

**Possibility 1**
If a `collection-viewer` element on the frontend has more than one `collection-view-primary-fields`, it can cause it to miss all realtime updates depending on your setup. This happens automatically and is part of the pub/sub delivery efficiency mechanism. In such situations, it is recommended to have at most one primary field explicitly defined inside the `collection-view-primary-fields` attribute. Note that if this attribute is not specified, then all the fields provided inside `collection-view-params` will be assumed to be primary fields. Note that `collection-view-primary-fields` can only reference fields which are present inside the `collection-view-params` attribute (should be a subset or empty string).

**Possibility 2**
You should consider removing some `primaryFields` from the affected `ModelView` in your Saasufy service schema via the `Admin HTTP API`. In most situations, it is recommended to have at most one primary field on a view (though it must match one of the `paramFields` of that view). Keep in mind that removing all primary fields can have negative performance implications depending on the view.

**Possibility 3**
If the update operation is being performed programmatically, it is strongly recommended that you add a `publisherId` property to the JSON payload.
For example:

```js
  // The publisherId does not necessarily have to be unique.
  // Just passing any string will turn off the anti-self-delivery
  // feature for that specific case.
  await socket.invoke('crud', {
    action: 'update',
    type: 'Candidate',
    id: candidateId,
    value: {
      status: "reviewed"
    },
    publisherId: "myapp"
  });
```
By default, for efficiency reasons, change notifications originating from a specific CRUD action are not sent to the socket which initiated the action (anti-self-delivery feature).

### Nested template expression inside curly brackets not updating as expected

In Saasufy, template expressions are evaluated as soon as possible (as soon as the expression can execute without throwing an error).
If an `{{expression}}` is nested inside a child component, it may be evaluated and substituted by the parent component before the child component itself was rendered; this means that when the child component updates itself later, the inner expression has already been substituted and is therefore not re-evaluated.

This issue can be resolved by using a variable from the child component as part of the expression; this ensures that the expression would fail to be evaluated during the parent's rendering phase and will be picked up later by the child component which has the necessary variable.

For example, the nested template expression `{{Date.now()}}` below would be evaluated by the parent app-router and not by the child app-router; this means that the timestamp would not update as expected when the `/product/:productName` sub-route changes:

```js
<app-router>
  <template slot="page" partial-route route-path="/category/:categoryName">
    <app-router>
      <template slot="page" partial-route route-path="/product/:productName">
        <div>Product was loaded at: {{Date.now()}}</div>
      </template>
      <div slot="viewport"></div>
    </app-router>
  </template>
  <div slot="viewport"></div>
</app-router>
```

You can force the expression to be re-evaluated by the child app-router by referencing one of its variables.
For example, the above markup could be written as:

```js
<app-router>
  <template slot="page" partial-route route-path="/category/:categoryName">
    <app-router>
      <template slot="page" partial-route route-path="/product/:productName">
        <div>Product was loaded at: {{(() => Date.now())(productName)}}</div>
      </template>
      <div slot="viewport"></div>
    </app-router>
  </template>
  <div slot="viewport"></div>
</app-router>
```

Even though the `productName` variable from the child app-router is not being used inside the fat-arrow function, merely referencing it inside the expression creates a dependency on that variable which ensures that the expression will not be accidentally rendered by the parent app-router.

### Template placeholder is empty because the Model name clashes with a global variable

Template expressions are evaluated as JavaScript, so a name which is not provided by an enclosing component falls through to the global `window` object. If a `Model` shares its name with a browser global, the placeholder silently resolves against that global instead of your data.

For example, `{{Credential.id}}` renders as an empty string because the browser defines `window.Credential`, so the expression evaluates as `window.Credential.id` which is `undefined`. Nothing throws, so there is no error in the console. Other names to watch out for include `Notification`, `Event`, `Request`, `Response`, `Location`, `File`, `Comment`, `Text`, `Option` and `Screen`; check `window.YourModelName` in the browser console to confirm.

The fix is to set a `type-alias` on the `collection-viewer`, `model-viewer` or `collection-reducer` and update every placeholder in its template to match:

```html
<collection-viewer collection-type="Credential" type-alias="AppCredential" ...>
  <template slot="item">
    <div class="credential">{{AppCredential.label}} ({{AppCredential.id}})</div>
  </template>

  <div slot="viewport"></div>
</collection-viewer>
```

The alias also applies to the error variable, so it becomes `{{$AppCredential.error.message}}`. Alternatively, rename the `Model` itself in your schema; when naming new models, prefer names which cannot clash, such as `UserCredential` instead of `Credential`.

### Issues related to passing reserved characters in HTML attributes such as commas and equal signs

Some component attributes take comma-separated values. In certain advanced scenarios, you may want the value for one of the properties to itself be a comma-separate value. In this case you would need to add single quotation marks around the nested value. See how the comma-separated value of the `fields` property is specified below.

```html
<collection-adder
  slot="collection-adder"
  collection-type="ModelIndex"
  model-values="name=groupIdMemberAccountId,fields='groupId,memberAccountId',maxCardinality:number=1,modelId=${this.modelId}"
  hide-submit-button
></collection-adder>
```

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

### View is empty

There may not be any data for that view which matches the specified filters.
Otherwise, check that the view is defined correctly. See the `Create a New View` section of the [schema-management.md](schema-management.md) guide for details and ensure that it's matching the `paramFields` which are passed to the view against matching model fields. If using advanced queries, check the [search-filtering-querying.md](search-filtering-querying.md) guide.

If the view uses `transformIndex`, check the index phase before the filter query — the index determines the set of records the view can return. See [views-and-indexing.md](views-and-indexing.md) for details on each of the following:

1. **A compound `between` which doesn't bind every index component.** Each of `transformIndexOperationInputA` and `transformIndexOperationInputB` specifies a complete index key, so for an index over `(clinicianId, startAt)` both must supply a value for every component, comma-separated in index order:

   ```jsonc
   "transformIndexOperationInputA": "$paramFields.clinicianId,$paramFields.fromAt",
   "transformIndexOperationInputB": "$paramFields.clinicianId,$paramFields.toAt"
   ```

   To confirm this is the cause, query the view with a range which cannot exclude anything (e.g. `fromAt=0`, `toAt=9999999999999`) and a known-good value for each leading component.

2. **A `transformIndex` pointing at a placeholder index.** Index names are assigned by Saasufy from the indexed fields, so a custom name supplied at creation is replaced on deployment and any view referencing it is left dangling; Saasufy then creates an index over a field of that name, which matches nothing if no such field exists. List the indexes and check that each one's `fields` names real fields on the model:

   ```bash
   SAASUFY_API_KEY=$(cat .saasufy-api-key)
   curl -g -H "Authorization:Bearer $SAASUFY_API_KEY" \
     -XGET 'https://saasufy.com/api/ModelIndex?view=accountModelAlphabeticalView&viewParams[modelId]={MODEL_ID}'
   ```

   An entry such as `{name: "clinicianStartIndex", fields: "clinicianStartIndex"}` is a placeholder. Delete it, create the intended compound index with `fields` only, and point `transformIndex` at the canonical name (`clinicianIdStartAt`).

3. **A `between` boundary landing on the upper bound.** `between` is half-open — `[from, to)` — so a record whose key equals `to` is excluded. Pass `to = end + 1` for an inclusive upper bound.

### View returns too many records (including other accounts' records)

A param declared in `paramFields` only filters if it is consumed — by an index input (`transformIndexOperationInputA`/`InputB`) or by a reference inside `transformFilterQuery` or `transformFilterOperationInput`. Matching the name of a model field is not sufficient.

Check that each name in `paramFields` appears in the view's index inputs or filter query. Where a param scopes results to a tenant, owner or account, prefer binding it to an index component. See [views-and-indexing.md](views-and-indexing.md).

### Schema tooling operates on only part of a model

Admin listing endpoints return a limited page of records (10 by default), so tooling which enumerates a model's fields, indexes or views needs to page through the results. A symptom is a bulk operation which appears to succeed but only affects the first handful of records alphabetically — for example access rules landing on only the first 10 fields of a 26-field model. See the `Paginating Listing Endpoints` section of the [schema-management.md](schema-management.md) guide.
