# model-input

Used for displaying and editing a single field of a model instance in realtime.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/model-input.js" type="module" defer></script>
```

**Example usage**

```html
<model-input
  type="text"
  model-type="Product"
  model-id="{{productId}}"
  model-field="name"
  placeholder="Product name"
></model-input>
```

**Attributes**

- `model-type` (required): Specifies the type of model to bind to. This should match a `Model` available in your Saasufy service.
- `model-id` (required): The id of the model instance/record to bind to.
- `model-field` (required): The field of the model instance/record to bind to for reading and updating.
- `debounce-delay`: The delay in milliseconds to wait before sending an update to the server. This is useful to batch multiple updates together (as is common when user is typing). Default is 300ms.
- `disable-instant-flush`: If specified, changes will only be saved once they have settled (E.g. after the element has lost focus or when the enter key is pressed).
- `list`: If specified, this sets the list attribute of the inner `input` element to provide input suggestions (only works with standard input elements).
- `type`: The type of the input component; can be `text`, `number`, `select`, `textarea`, `checkbox` or `file`.
- `placeholder`: Can be used to set the placeholder text on the inner input element.
- `consumers`: This allows you to connect this `model-input` to other elements on your page. It takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the value of the `model-input` component in real-time. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type. Note that `model-input` elements of the `file` type are treated as write-only unless the `consumers` attribute is present.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers.
- `show-error-message`: If this attribute is present, an error message will be displayed above the input if it fails to update the value.
- `options`: A comma-separated list of options to provide - This only works if the input type is `select`.
- `height`: Allows you to set the height of the inner `input` element programmatically.
- `default-value`: A default value to show the user if the underlying model field's value is null or undefined. Note that setting a default value will not affect the underlying model's value.
- `value`: Used to set the model's value.
- `computable-value`: If this attribute is present, it will be possible to execute expressions using the `{{expression}}` syntax inside the `value` attribute.
- `slice-to`: Optional attribute which can be set to a number to trim strings down to a maximum number of characters. This is useful when dealing with potentially very long field values. Extra care should be taken when using this attribute on a `model-input` element as it will cause values to be overwritten with shortened values if the user edits the input box. Note also that it may affect caching if the same field is being referenced in multiple parts of the application at the same time within the same `socket-provider`.
- `hide-error-logs`: By default, this component will log errors to the console. If set, this attribute will suppress such errors from showing up on the console.
- `autocapitalize`: Can be `on` or `off` - It will set the auto-capitalize attribute on the inner input element. This is useful for mobile devices to enable or disable auto-capitalization of the first character which is typed into the input element.
- `autocorrect`: Can be `on` or `off` - It will set the auto-correct attribute on the inner input element. This is useful for mobile devices to enable or disable auto-correct.
- `enable-rebound`: By default, for efficiency reasons, real-time updates performed via `model-input` components are not sent back to the publishing client; it is usually unnecessary because state changes are shared locally between all components which are bound to the same `socket-provider`. You can add the `enable-rebound` attribute to the `model-input` component to force real-time updates to rebound back to the publisher to support more advanced scenarios. Note that enabling rebound comes with additional performance overheads.
- `accept`: This attribute should only be used if the `type` is set to `file`; it serves to constrain the types of files which can be selected by the user.
- `input-id`: Can be used to set an `id` attribute on the inner `input` element.
- `input-props`: Can be used to set additional attributes on the inner `input` element. Must be in the format `attr1=value1,attr2=value2`.
- `ignore-invalid-selection`: Can be used with a `model-input` component of type `select` to prevent the `error` class from being added if the selected value is not among the available options.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
