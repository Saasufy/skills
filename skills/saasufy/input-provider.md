# input-provider

An input component which can pass data to other components (or HTML elements) via custom attributes.
A common use case for it is to pass user input to `collection-viewer` or `model-viewer` components to then fetch data from Saasufy.
This component can be configured to provide its data to multiple components (consumers) via custom attributes.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/input-provider.js" type="module" defer></script>
```

**Example usage**

```html
<input-provider
  type="text"
  consumers=".my-input"
  value="Hello world!"
  placeholder="Message"
></input-provider>
```

**Attributes**

- `input-id`: Can be used to set an `id` attribute on the inner `input` element.
- `list`: If specified, this sets the list attribute of the inner `input` element to provide input suggestions (only works with standard input elements).
- `type`: The type of the input component; can be `text`, `number`, `select`, `textarea` or `checkbox`.
- `disable-instant-flush`: If specified, changes will only be saved once they have settled (E.g. after the element has lost focus or when the enter key is pressed).
- `placeholder`: Can be used to set the placeholder text on the inner input element.
- `consumers`: This allows you to connect this `input-provider` to other elements on your page. It takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the value of the `input-provider` component in real-time. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers.
- `debounce-delay`: The delay in milliseconds to wait before triggering a change event. This is useful to batch multiple updates together (as is common when user is typing). Default is 800ms.
- `options`: A comma-separated list of options to provide - This only works if the input type is `select`.
- `height`: Allows you to set the height of the inner `input` element programmatically.
- `value`: Used to set the input's value.
- `computable-value`: If this attribute is present, it will be possible to execute expressions using the `{{expression}}` syntax inside the `value` attribute.
- `autocapitalize`: Can be `on` or `off` - It will set the auto-capitalize attribute on the inner input element. This is useful for mobile devices to enable or disable auto-capitalization of the first character which is typed into the input element.
- `autocorrect`: Can be `on` or `off` - It will set the auto-correct attribute on the inner input element. This is useful for mobile devices to enable or disable auto-correct.
- `input-props`: Can be used to set additional attributes on the inner `input` element. Must be in the format `attr1=value1,attr2=value2`.
- `skip-init-change`: This attribute can be used to prevent an `input-provider` from triggering a change when initialized with a default value. By default, the `input-provider` will trigger a change when initialized with a default value.
- `force-init-change`: This attribute can be used to force an `input-provider` to always trigger a change when initialized, even if the value is empty.
