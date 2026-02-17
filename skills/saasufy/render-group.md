# render-group

A component which can be used to guarantee that certain children components are always rendered at the same time (once they are loaded) to provide a smooth user experience.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/render-group.js" type="module" defer></script>
```

**Example usage**

```html
<render-group wait-for="1,2,3">
  <model-input
    type="text"
    load-id="1"
    model-type="Product"
    model-id="{{productId}}"
    model-field="name"
  ></model-input>
  <model-input
    type="text"
    load-id="2"
    model-type="Product"
    model-id="{{productId}}"
    model-field="desc"
  ></model-input>
  <model-input
    type="number"
    load-id="3"
    model-type="Product"
    model-id="{{productId}}"
    model-field="qty"
  ></model-input>
</render-group>
```

**Attributes**

- `wait-for` (required): A list of components to wait for before rendering based on their `load-id` attributes.
The `render-group` will only render its content once all the components specified via this attribute have finished loading their data.

