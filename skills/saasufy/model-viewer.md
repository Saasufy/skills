# model-viewer

Used for rendering a single model resource using a template. This is the single-model alternative to the `collection-viewer` which works with collections of models.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/model-viewer.js" type="module" defer></script>
```

**Example usage**

```html
<model-viewer
  model-type="Product"
  model-id="{{modelId}}"
  model-fields="name,desc,qty"
>
  <template slot="item">
    <div>
      <div>Name: {{Product.name}}</div>
      <div>Description: {{Product.desc}}</div>
      <div>Quantity: {{Product.qty}}</div>
    </div>
  </template>

  <div slot="viewport"></div>
</model-viewer>
```

**Attributes**

- `model-type` (required): Specifies the type of model to bind to. This should match a `Model` available in your Saasufy service.
- `model-id` (required): The id of the model instance/record to bind to.
- `model-fields` (required): A comma-separated list of fields to bind to. Note that unlike with `model-input` and `model-text`, the attribute name is plural and can bind to multiple fields.
- `type-alias`: Allows you to provide an alternative name for your `Model` to use when injecting values inside the template. This is useful for situations where you may have multiple `model-viewer` elements and/or `model-viewer` elements of the same type nested inside each other and want to avoid `Model` name clashes in the nested template definitions. For example, if the `type-alias` in the snippet above was set to `MyProduct`, then `{{Product.name}}` would become `{{MyProduct.name}}`.
- `fields-slice-to`: Optional attribute which can be used to trim strings down to a maximum number of characters when reading. The format is `fieldName1=number1,fieldName2=number2`. This is useful when dealing with potentially very long field values. Note that it may affect caching if the same field is being referenced in multiple parts of the application at the same time within the same `socket-provider`.
- `hide-error-logs`: A flag which, when present, suppresses error logs from being printed to the console.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
