# collection-viewer

Used for rendering collections as lists, tables or other sequences based on a specific view using a template.
Supports pagination by allowing you to specify custom buttons or links to navigate between pages.
Can also perform basic CRUD operations such as deleting or creating records by listening for events from child components.

For querying/filtering (using the `collection-view-params` attribute), check the `saasufy-search-filtering-querying` skill.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-viewer.js" type="module" defer></script>
```

**Example usage**

```html
<collection-viewer
  collection-type="Chat"
  collection-fields="username,message,createdAt"
  collection-view="recentView"
  collection-view-params=""
  collection-view-primary-fields=""
  collection-page-size="50"
>
  <template slot="item">
    <div>
      <div>{{Chat.username}}</div>
      <div>{{Chat.message}}</div>
      <div>{{date(Chat.createdAt)}}</div>
    </div>
  </template>

  <template slot="no-item">
    <div>There are no messages.</div>
  </template>

  <template slot="loader">
    <div class="loading-spinner-container">
      <div class="spinning">&#8635;</div>
    </div>
  </template>

  <div slot="viewport" class="chat-viewport"></div>
</collection-viewer>
```

In addition to the slots shown above, the `collection-viewer` also supports the following optional slots:
- `previous-page`: This is typically an `input` element of type `button` for pagination.
- `next-page`: This is typically an `input` element of type `button` for pagination.
- `page-number`: This is typically a `div` element to display the current page number. Alternatively, it can be an `input-provider` element to allow the user to manually type in the page number.
- `first-item`: This should be a `template` element. It can be used to inject an element just above the first record. This can be useful to render the `collection-viewer` as a table with a heading above table rows.
- `last-item`: This should be a `template` element. It can be used to inject an element just after the last record.
- `error`: This can be used to display an error message if the required data cannot be loaded. The `Error` object can be accessed inside the template via the syntax: `$CollectionName.error` (substitute `CollectionName` with your `collection-type` or, if set, your `type-alias` preceded by a dollar sign). For example, the error message for a `Chat` collection will be `$Chat.error.message`.

**Attributes**

- `collection-type` (required): Specifies the type of collection to display. This should match a `Model` available in your Saasufy service.
- `collection-fields` (required): A comma-separated list of fields from the `Model` that you want to display or use. This lets you pick specific pieces of data from your collection to work with. Note that updates made to these fields can cause the `collection-viewer` to re-render itself so you should consider whether or not a field needs to be specified at this level in the component hierarchy or should be delegated to a child component (otherwise it could cause nested `model-input` components to lose focus).
- `collection-view` (required): Determines the view of the collection. This should match one of the `Views` defined in your Saasufy service under that specific `Model`.
- `collection-view-params` (required): Parameters for the view, specified as comma-separated key-value pairs (e.g., key1=value1,key2=value2). These parameters can customize the behavior of the collection view. The keys must match `paramFields` specified in your Saasufy service under the relevant `View`.
- `collection-view-primary-fields` (required): A list of field names to specify which `collection-view-params` to watch for realtime updates (to constrain the channel). Fewer primary fields means that the view will be exposed to a broader range of realtime updates but at the cost of performance. It is generally recommended to have at most one primary field. Can be an empty string.
- `collection-page-size`: Sets how many items from the collection are displayed at once. This is useful for pagination, allowing users to navigate through large sets of data in chunks.
- `collection-page-offset`: Indicates the current page offset in the collection's data. It’s like telling the viewer which page of data you want to display initially.
- `collection-get-count`: If this attribute is present, the component will get the record count of the target view. The count can be rendered into the template by prefixing the model name with a dollar sign and accessing the `count` property like this: `{{$MyModelName.count}}`.
- `collection-disable-realtime`: If this attribute is present, the underlying collection will not listen for changes nor update in realtime. Fields of records that are in view may still update in realtime.
- `auto-reset-page-offset`: If this attribute is present, the `collection-page-offset` will be reset to zero whenever the view params change.
- `type-alias`: Allows you to provide an alternative name for your `Model` to use when injecting values inside the template. This is useful for situations where you may have multiple `collection-viewer` elements and/or `model-viewer` elements of the same type nested inside each other and want to avoid `Model` name clashes in the nested template definitions. For example, if the `type-alias` in the snippet above was set to `SubChat`, then `{{Chat.message}}` would become `{{SubChat.message}}`.
- `hide-error-logs`: A flag which, when present, suppresses error logs from being printed to the console.
- `max-show-loader`: If this attribute is present, your slotted `loader` element will be shown as often as possible; this includes situations where the collection is merely refreshing itself. It is disabled by default.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.




