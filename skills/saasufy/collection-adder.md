# collection-adder

A basic form component for inserting data into collections.
This component is a simplified version of the `collection-adder-form` component as it doesn't require input elements to be slotted into it.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-adder.js" type="module" defer></script>
```

**Example usage**

```html
<collection-adder
  collection-type="Product"
  collection-fields="name,brand,qty"
  model-values="categoryName=electronics,isNew:boolean=true"
  submit-button-label="Create"
  trim-spaces
></collection-adder>
```

**Attributes**

- `collection-type` (required): Specifies the type of collection to add the resource to when the form is submitted. This should match a `Model` available in your Saasufy service.
- `collection-fields`: A comma-separated list of fields from the `Model` to display as input elements inside the form for the user to fill in. Each field name in this list can optionally be followed by an input element type after a `:` character. For example `collection-fields="qty:number, size:select(small,medium,large)` will create one input element with `type="number"` and one with `type=select` with options `small`, `medium` or `large`. To make a select field optional, you can prefix the list of options with a comma; e.g. `size:select(,small,medium,large)`. Supported input types include: `text`, `select`, `number`, `checkbox`, `radio`, `textarea`, `file` and `text-select`.
- `model-values`: An optional list of key-value pairs in the format `field1=value1,field2=value2` to add to the newly created resource alongside the values collected from the user via the form. For non-string values, the type should be provided in the format `fieldName:type=value`; supported types are `string`, `number` and `boolean`. Unlike `collection-fields` which are rendered as input elements in the form, `model-values` are not shown to the user.
- `computable-model-values`: If this attribute exists on the element, then the `model-values` attribute will be re-computed before the form is submitted; this may be useful if the `model-values` attribute contains expressions which need to be updated before submitting.
- `field-labels`: Allows you to specify custom input labels by mapping field names to more user-friendly labels in the format `fieldName1='Field Name 1',fieldName2='Field Name 2'`.
- `submit-button-label`: Text to display on the submit button. If not specified, defaults to `Submit`.
- `hide-submit-button`: Adding this attribute will hide the submit button from the form.
- `success-message`: A message to show the user if the resource has been successfully added to the collection after submitting the form.
- `autocapitalize`: Can be set to `off` to disable auto-capitalization of the first input character on mobile devices.
- `autocorrect`: Can be set to `off` to disable auto-correction of input on mobile devices.
- `show-field-errors`: If this attribute exists on the element, then granular error messages will be shown above each corresponding input element instead of a single form-level error message.
- `trim-spaces`: If this attribute exists on the element, then leading and trailing spaces will be trimmed from each input element's value before submitting the form.
- `consumers`: This allows you to connect this `collection-adder` to other elements on your page. It should be used in conjunction with the `provider-template` attribute. The `consumers` attribute takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the output of the `collection-adder` component on submit. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type.
- `success-consumers`: Same as the `consumers` attribute but the target elements are only updated if insertion was a success.
- `error-consumers`: Same as the `consumers` attribute but the target elements are only updated if insertion was a failure.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers. In the case of the `collection-adder` component, the value is an object with either a `resource` property (on successful insert) or `error` (on failure). On success, the `resource` object will be the object which was added to the collection. On failure, the `error` object will be a JavaScript `Error` object with a `message` property.
