# Debugging and Troubleshooting

## Common issues

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