# if-group

A component which exposes a `show-content` property which can be set to true or false to show or hide its content.
It's intended to be placed inside a `model-viewer`, `collection-viewer` or `collection-reducer` component such that the true/false value of the `show-content` attribute can be computed using a template `{{expression}}` placeholder. This component requires a template and a viewport to be slotted in. The content of the template will not be processed unless the `show-content` condition is met.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/if-group.js" type="module" defer></script>
```

**Example usage**

```html
<if-group show-content="{{!!Product.isVisible}}">
  <template slot="content">
    <div>Name: {{Product.name}}</div>
    <div>Description: {{Product.desc}}</div>
    <div>Quantity: {{Product.qty}}</div>
  </template>
  <div slot="viewport"></div>
</if-group>
```
