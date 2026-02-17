# socket-provider

A top level component which connects to your Saasufy service and inside which you can place other components which depends on Saasufy data.
A Saasufy component which integrates with data from Saasufy is known as a `socket-consumer` and must always be placed inside a `socket-provider` element (although it does not have to be a direct child).

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/socket.js" type="module" defer></script>
```

**Example usage**

```html
<socket-provider url="wss://saasufy.com/sid7890/socketcluster/">
  <!-- Elements which rely on Saasufy data go here. -->
</socket-provider>
```

**Attributes**

- `url` (required): Specifies the URL for the Saasufy service to use as the data store for all components which are placed inside it.
- `auth-token-name`: Allows you to specify a custom key for your token - This will be used as the key in localStorage. You generally do not need to set this attribute. It is intended for situations were an app has multiple `socket-provider` elements connecting different services with different/separate authentication flows.
- `disable-tab-sync`: By default, the socket provider synchronizes socket auth state across multiple tabs via localStorage. If this attribute is set, then the socket auth state will not sync automatically and a manual page refresh may be necessary to update to the latest auth state.
- `socket-options`: Can be used to set options on the inner `socket`. Must be in the format `option1:type1=value1,option2:type2=value2`; the type of each option can be string, boolean or number. If not specified, the default type is string.
- `disconnect-on-deauth`: If this attribute is set, the underlying socket will be disconnected if the socket becomes unauthenticated. This means that components will not receive real-time updates until the user's next interaction (which will cause the socket to reconnect). If this attribute is not set, the socket will attempt to reconnect immediately after losing authentication.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
