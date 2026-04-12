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

### No permission to read

If you see an error like "You do not have permission to perform the ... operation on the ... field", it means that the current client doesn't have the right to access that resource. Saasufy enforces access control at the field level so you may get multiple such errors; one for each field you try to read. To make the error go away, check that your view is defined correctly and that it filters out records which belong to other accounts. You should be careful about setting the read access to "allow" as it will allow anyone to read the resource which may be undesirable.

### Views are showing empty data while authenticated

Check that there is data associated with your specific `accountId` within the specified collection/view. If some of the data was created via the Saasufy HTTP API, check that the account ID associated with your API credential is as expected. You can associate an HTTP API credential with an account ID of your choice. See the `Impersonating an account via HTTP API` section of the [data-management.md](data-management.md) guide for details.

