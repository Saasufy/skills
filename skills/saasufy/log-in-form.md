# log-in-form

A form component which allows end-users to authenticate themselves into your app using the Saasufy blockchain or any Capitalisk-based blockchain.
On success, the socket of the form's `socket-provider` will be set to the authenticated state; this will then cause appropriate Saasufy components which share this `socket-provider` to reload themselves.
Once a user is authenticated, they will be able to access restricted data (as specified on the Saasufy `Model -> Access` page).
It's also possible to specify a `success-location-hash` attribute to trigger a client-side redirect upon successful authentication.
The change in the location hash can then be detected by an `app-router` to switch to a different page upon successful login.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/log-in-form.js" type="module" defer></script>
```

**Example usage**

```html
<log-in-form
  hostname="capitalisk.com"
  port="443"
  network-symbol="clsk"
  chain-module-name="capitalisk_chain"
  secure="true"
></log-in-form>
```

**Attributes**

- `hostname` (required): The hostname of the blockchain node to match authentication details against.
- `port` (required): The port of the blockchain node to match authentication details against. Should be 80 for HTTP/WS or 443 for HTTPS/WSS.
- `network-symbol` (required): The symbol of the target blockchain.
- `chain-module-name` (required): The module name used by the target blockchain.
- `secure` (required): Should be `true` or `false` depending on whether or not the node is exposed over HTTP/WS or HTTPS/WSS.
- `auth-timeout`: The maximum number of milliseconds to wait for authentication to complete. Defaults to 10000 (10 seconds).
- `success-location-hash`: If authentication is successful, the browser's `location.hash` will be set to this value. It can be used to redirect the user to their dashboard, for example.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.
