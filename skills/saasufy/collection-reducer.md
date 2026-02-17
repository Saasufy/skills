# collection-reducer

Similar to the `collection-viewer` component but designed to render collections in combined format. For example, to combine values from multiple records into a single item.
A common use case is to extract and join values to pass to other child components via their attributes.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-reducer.js" type="module" defer></script>
```

**Example usage**

```html
<!--
  Here, the collection-reducer fetches all available categories and joins their name fields
  to use as options for a model-input select element.
  This allows the user to choose among available category names to update the
  current product's categoryName field.
  You should assume that {{productId}} comes from a parent component (e.g. app-router).
-->
<collection-reducer
  collection-type="Category"
  collection-fields="name"
  collection-view="alphabeticalView"
  collection-page-size="100"
  collection-view-params=""
>
  <template slot="item">
    <div class="form">
      <label class="label-container">
        <div>Product category</div>
        <model-input
          type="select"
          options="{{joinFields(Category, 'name')}}"
          model-type="Product"
          default-value="accountId"
          model-id="{{productId}}"
          model-field="categoryName"
        ></model-input>
      </label>
    </div>
  </template>
  <div slot="viewport"></div>
</collection-reducer>
```

Unlike with `collection-viewer` or `model-viewer`, the template variable references not a single model instance but an array of model instances.
In the example above, the name of the first element can be accessed with `{{Category[0].name}}`.

**Attributes**

- `collection-type` (required): Specifies the type of collection to use. This should match a `Model` available in your Saasufy service.
- `collection-fields` (required): A comma-separated list of fields from the `Model` that you want to use. This lets you pick specific pieces of data from your collection to work with.
- `collection-view` (required): Determines the view of the collection. This should match one of the `Views` defined in your Saasufy service under that specific `Model`.
- `collection-view-params` (required): Parameters for the view, specified as comma-separated key-value pairs (e.g., key1=value1,key2=value2). These parameters can customize the behavior of the collection view. The keys must match `paramFields` specified in your Saasufy service under the relevant `View`.
- `collection-page-size`: Sets how many items from the collection are displayed at once. This is useful for pagination, allowing users to navigate through large sets of data in chunks.
- `collection-page-offset`: Indicates the current page offset in the collection's data. It’s like telling the viewer which page of data you want to display initially.
- `collection-get-count`: If this attribute is present, the component will get the record count of the target view. The count can be rendered into the template by prefixing the model name with a dollar sign and accessing the `count` property like this: `{{$MyModelName.count}}`.
- `type-alias`: Allows you to provide an alternative name for your `Model` to use when injecting values inside the template. This is useful for situations where you may have multiple `collection-viewer` elements and/or `model-viewer` elements of the same type nested inside each other and want to avoid `Model` name clashes in the nested template definitions. For example, if the `type-alias` in the snippet above was set to `SubChat`, then `{{Chat.message}}` would become `{{SubChat.message}}`.
- `hide-error-logs`: A flag which, when present, suppresses error logs from being printed to the console.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
