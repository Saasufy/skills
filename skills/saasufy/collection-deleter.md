# collection-deleter

A component which can be placed anywhere inside a `collection-viewer` component to delete a specific item from a collection as a result of a user action (e.g. on click).
It supports either immediate deletion or deletion upon confirmation; in the latter case, the parent `collection-viewer` must have a `confirm-modal` component slotted into its `modal` slot.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/collection-deleter.js" type="module" defer></script>
```

**Example usage**

```html
<!-- Must be placed somewhere inside a collection-viewer component, typically inside the slotted item template. -->
<collection-deleter model-id="{{Product.id}}" onclick="deleteItem()">&#x2715;</collection-deleter>
```

OR (with confirmation step)

```html
<!-- Must be placed somewhere inside a collection-viewer component, typically inside the slotted item template. -->
<collection-deleter model-id="{{Product.id}}" confirm-message="Are you sure you want to delete the {{Product.name}} product?" onclick="confirmDeleteItem()">&#x2715;</collection-deleter>
```

**Attributes**

- `model-id` (required): Specifies the ID of the resource to delete from the parent collection when this component is activated. This can be achieved by invoking either the `deleteItem()` or `confirmDeleteItem()` function from inside an event handler. The example above shows how to achieve deletion via the `onclick` event. You can either invoke `deleteItem()` to delete the resource immediately or you can invoke `confirmDeleteItem()` to require additional confirmation prior to deletion.
- `onclick` (required): The logic to execute to delete the item. Should be either `deleteItem()` or `confirmDeleteItem()`.
- `confirm-message`: The confirmation message to show the user when this component's `confirmDeleteItem()` function is invoked.
- `model-field`: If specified, this attribute can be used to delete a single field of a model instance, instead of the whole instance.

If `confirmDeleteItem()` is used, then the parent `collection-viewer` must have a `confirm-modal` element slotted into it as shown here:

```html
<collection-viewer
  collection-type="Product"
  collection-fields="name,qty"
  collection-view="alphabeticalView"
  collection-view-params=""
  collection-page-size="50"
>
  <template slot="item">
    <div>
      <div>{{Product.name}}</div>
      <collection-deleter model-id="{{Product.id}}" confirm-message="Are you sure you want to delete the {{Product.name}} product?" onclick="confirmDeleteItem()">&#x2715;</collection-deleter>
    </div>
  </template>

  <div slot="viewport" class="chat-viewport"></div>

  <!-- The confirm-modal element must be specified here with slot="modal" to prompt the user for confirmation -->
  <confirm-modal slot="modal" title="Delete confirmation" message="" confirm-button-label="Delete"></confirm-modal>
</collection-viewer>
```
