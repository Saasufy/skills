# overlay-modal

A general-purpose overlay modal component.

**Styling**

You will need to use the CSS `::part()` pseudo-element to style child elements.

The parts which can be styled are: `overlay-background`, `dialog`, `title-bar`, `content` and `footer`.
You should target the relevant part by first referencing the internal `overlay-modal` element and using the `::part()` syntax, for example:

```css
overlay-modal::part(title-bar) {}
```

Here is what its internal HTML structure looks like (showing the shadow DOM inside the template tag):

```html
<overlay-modal>
  <template shadowrootmode="open">
    <style>
      .overlay-background {
        display: flex;
        justify-content: center;
        align-items: center;
        position: fixed;
        top: 0;
        right: 0;
        bottom: 0;
        left: 0;
        z-index: 10000;
        background-color: rgba(0, 0, 0, .3);
      }

      .modal-dialog {
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        width: var(--modal-width, 800px);
        height: var(--modal-height, auto);
        max-width: 100%;
        min-height: var(--modal-min-height, auto);
        background-color: #ffffff;
      }

      .modal-title-bar {
        display: var(--title-bar-display, flex);
        justify-content: space-between;
        align-items: center;
        width: 100%;
        padding: 10px;
        color: #ffffff;
        box-sizing: border-box;
        background: #e37042;
      }

      .modal-content {
        flex-grow: 1;
        padding: 10px;
      }

      .modal-footer {
        display: var(--footer-display, flex)
        padding: 10px;
      }

      .close-button {
        cursor: pointer;
      }
    </style>
    <div class="overlay-background" part="overlay-background">
      <div class="modal-dialog" part="dialog">
        <div class="modal-title-bar" part="title-bar">
          <div>
            <slot name="title"></slot>
          </div>
          <div class="close-button">✕</div>
        </div>
        <div class="modal-content" part="content">
          <slot name="content"></slot>
        </div>
        <div class="modal-footer" part="footer">
          <slot name="footer"></slot>
        </div>
      </div>
    </div>
  </template>
  <div slot="title">Delete Item</div>
  <div class="confirm-modal-content" slot="content">
    <div>Are you sure you want to delete HP Laptop?</div>
    <div class="confirm-modal-buttons-container">
      <input class="modal-confirm-button" type="button" value="Delete">
      <input class="modal-cancel-button" type="button" value="Cancel">
    </div>
  </div>
</overlay-modal>
```