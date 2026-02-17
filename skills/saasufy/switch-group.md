# switch-group

A component which exposes a `show-cases` property which can be set to key-value pairs in the format `key1=true,key2=false`. It can be used to conditionally display multiple slotted elements based on multiple conditions. It helps to keep HTML clean when the conditions are complex.
This element is intended to be placed inside a `model-viewer`, `collection-viewer` or `collection-reducer` component such that the true/false values of the `show-cases` attribute can be computed using template `{{expression}}` placeholders. This component requires one or more templates and a viewport to be slotted in. The content of the templates will not be processed unless the `show-cases` condition is met. All templates must have a `slot="content"` attribute and a name attribute in the format `name="key1"` where the value `key1` corresponds to the key specified inside the `show-cases` attribute of the `switch-group`.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/switch-group.js" type="module" defer></script>
```

**Example usage**

```html
<switch-group show-cases="image={{!!Section.imageURL}},text={{!!Section.text}}">
  <template slot="content" name="image">
    <img src="{{Section.imageURL}}" alt="{{Section.imageDesc}}" />
  </template>
  <template slot="content" name="text">
    <div>{{Section.text}}</div>
  </template>
  <div slot="viewport"></div>
</switch-group>
```
