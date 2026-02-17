# collection-adder-form

A flexible form component for inserting data into collections.
This component is similar to the `collection-adder` component except that it requires input elements (and others) to be slotted into it.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-adder-form.js" type="module" defer></script>
```

**Example usage**

```html
<collection-adder-form
  collection-type="Product"
  model-values="categoryName=electronics,isNew:boolean=true"
  trim-spaces
>
  <div slot="message"></div>
  <input type="text" name="name" placeholder="Name">
  <input type="text" name="brand" placeholder="Brand">
  <input type="number" name="qty" placeholder="Quantity">
  <input type="submit" value="Add new product">
</collection-adder-form>
```

**Attributes of collection-adder-form**

- `collection-type` (required): Specifies the type of collection to add the resource to when the form is submitted. This should match a `Model` available in your Saasufy service.
- `model-values`: An optional list of key-value pairs in the format `field1=value1,field2=value2` to add to the newly created resource alongside the values collected from the user via the form. For non-string values, the type should be provided in the format `fieldName:type=value`; supported types are `string`, `number` and `boolean`. Unlike `collection-fields` which are rendered as input elements in the form, `model-values` are not shown to the user.
- `computable-model-values`: If this attribute exists on the element, then the `model-values` attribute will be re-computed before the form is submitted; this may be useful if the `model-values` attribute contains expressions which need to be updated before submitting.
- `success-message`: A message to show the user if the resource has been successfully added to the collection after submitting the form.
- `show-field-errors`: If this attribute exists on the element, then granular error messages will be shown above each corresponding input element instead of a single form-level error message.
- `trim-spaces`: If this attribute exists on the element, then leading and trailing spaces will be trimmed from each input element's value before submitting the form.
- `auto-reset-hidden-inputs`: If this attribute exists, then `hidden` input elements which are inside the `collection-adder-form` will be cleared along with other input fields. By default, `hidden` input elements are not cleared when the form is reset.
- `disable-submit-on-enter-key`: If this attribute exists, then the form will not submit when the enter key is pressed.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
- `consumers`: This allows you to connect this `collection-adder-form` to other elements on your page. It should be used in conjunction with the `provider-template` attribute. The `consumers` attribute takes a list of selectors with optional attributes to target in the format `element-selector:attribute-name`. For example `.my-input:value` will find all elements with a `my-input` class and update their `value` attributes with the output of the `collection-adder-form` component on submit. You can specify multiple selectors separated by commas such as `.my-input,my-div`; in this case, because attribute names are not specified, values will be injected into the `value` attribute (for input elements) or into the `innerHTML` property (for other kinds of elements). The default attribute/property depends on the element type.
- `success-consumers`: Same as the `consumers` attribute but the target elements are only updated if insertion was a success.
- `error-consumers`: Same as the `consumers` attribute but the target elements are only updated if insertion was a failure.
- `provider-template`: A template string within which the `{{value}}` can be injected before passing to consumers. In the case of the `collection-adder-form` component, the value is an object with either a `resource` property (on successful insert) or `error` (on failure). On success, the `resource` object will be the object which was added to the collection. On failure, the `error` object will be a JavaScript `Error` object with a `message` property.

**Attributes of slotted input, select, textarea... elements**

- `name` (required): This must correspond to a field name on the model specified via the `collection-type` attribute of the `collection-adder-form`.
- `output-type`: An optional type to cast the value of the input element into when the form is submitted.
