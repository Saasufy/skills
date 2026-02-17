# input-combiner

A component which can be used to combine data from multiple `input-provider` components and pass the combined data to other components (or HTML elements) via custom attributes.
A common use case for it is to pass user input to `collection-viewer` or `model-viewer` components to then fetch data from Saasufy.
This component can be configured to provide its data to multiple components (consumers) via custom attributes.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/input-combiner.js" type="module" defer></script>
```

**Example usage**

The following example shows how to bind multiple `input-provider` components to an `input-combiner`.
In this example, the `input-combiner` component forwards the combined, formatted values to the `collection-view-params` attribute of a `company-viewer` component.

Note that the `value` inside the `provider-template` attribute of the `input-combiner` is an object with keys that represent the `name` attribute of the different source `input-provider` components.
The `combineFilters` function produces a combined query string (joined with `%AND%`) by iterating over the `value` object which holds different parts of the query as provided by different `input-provider` components.

```html
<input-provider
  class="desc-input"
  name="desc"
  type="text"
  consumers=".search-combiner"
  provider-template="description contains (?i){{value}}"
  value=""
  placeholder="Description filter"
></input-provider>

<input-provider
  class="city-input"
  name="city"
  type="text"
  consumers=".search-combiner"
  provider-template="city contains (?i){{value}}"
  value=""
  placeholder="City filter"
></input-provider>

<input-combiner
  class="search-combiner"
  consumers=".company-viewer:collection-view-params"
  provider-template="query='{{combineFilters(value)}}'"
></input-combiner>
```

**Attributes**

- `consumers`: This allows you to connect this `input-combiner` to other elements on your page. It takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the value of the `input-provider` component in real-time. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers. In the case of the `input-combiner`, the value is an object wherein each key represents the unique label associated with different `input-provider` components.
- `debounce-delay`: The delay in milliseconds to wait before triggering a change event. This is useful to batch multiple updates together. Default is 500ms.
