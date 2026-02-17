# input-transformer

A component which can be used to transform data from an `input-provider` component and pass the transformed data to other components (or HTML elements) via custom attributes.
This component can be configured to provide its data to multiple components (consumers) via custom attributes.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/input-transformer.js" type="module" defer></script>
```

**Example usage**

The following example shows how to bind an `input-provider` component to an `input-transformer`.

```html
<input-provider
  class="dollar-input"
  name="count"
  type="text"
  consumers=".cents-transformer"
  value=""
  placeholder="Dollars"
></input-provider>

<input-transformer
  class="cents-transformer"
  consumers=".cents-container"
  provider-template="{{Number(value) * 100}}"
></input-transformer>

<div class="cents-container"></div>
```

**Attributes**

- `consumers`: This allows you to connect this `input-transformer` to other elements on your page. It takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the value of the `input-provider` component in real-time. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers.
- `output-type`: The default output type is `string`. This property can be used to convert the output to either `boolean` or `number`.
- `debounce-delay`: The delay in milliseconds to wait before triggering a change event. This is useful to batch multiple updates together. Default is 0ms.
