# collection-adder-group

A component which can be used to group together multiple `collection-adder` components. It can be used to insert multiple records into a collection via a single button click.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-adder-group.js" type="module" defer></script>
```

**Example usage**

```html
<!--
  This collection-adder-group adds multiple pre-defined records into a Product
  collection using a single button click. This example assumes that the name
  field is the only required field in the Product model.
-->
<collection-adder-group>
  <collection-adder
    slot="collection-adder"
    collection-type="Product"
    model-values="name=Flour"
    hide-submit-button
  ></collection-adder>
  <collection-adder
    slot="collection-adder"
    collection-type="Product"
    model-values="name=Egg"
    hide-submit-button
  ></collection-adder>
  <collection-adder
    slot="collection-adder"
    collection-type="Product"
    model-values="name=Milk"
    hide-submit-button
  ></collection-adder>
  <div slot="error-container"></div>
  <input slot="submit-button" type="button" value="Add pancake ingredients">
</collection-adder-group>
```
