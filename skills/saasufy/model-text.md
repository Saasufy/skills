# model-text

Used to displaying a field of a model instance in realtime.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/model-text.js" type="module" defer></script>
```

**Example usage**

```html
<model-text
  type="text"
  model-type="Product"
  model-id="{{productId}}"
  model-field="name"
></model-text>
```

**Attributes**

- `model-type` (required): Specifies the type of model to bind to. This should match a `Model` available in your Saasufy service.
- `model-id` (required): The id of the model instance/record to bind to.
- `model-field` (required): The field of the model instance/record to bind to for reading.
- `slice-to`: Optional attribute which can be set to a number to trim strings down to a maximum number of characters when reading. This is useful when dealing with potentially very long field values. Note that it may affect caching if the same field is being referenced in multiple parts of the application at the same time within the same `socket-provider`.
- `default-value`: A default value to show the user if the underlying model field's value is null or undefined.
- `hide-error-logs`: By default, this component will log errors to the console. If set, this attribute will suppress such errors from showing up on the console.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
