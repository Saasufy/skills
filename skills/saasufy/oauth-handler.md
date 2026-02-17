# oauth-handler

This component handles the final stage of OAuth and then redirects the user to the relevant page/URL if successful.
When using an OAuth provider, the callback URL which you register with the provider must lead the user back to a page which contains this `oauth-handler` component.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/oauth-handler.js" type="module" defer></script>
```

**Example usage**

```html
<oauth-handler provider="github" success-location-hash="/chat"></oauth-handler>
```

**Attributes**

- `provider` (required): The name of the provider. It must match the provider name specified on the `Authentication` page of your Saasufy control panel and also the value provided to the `oauth-link` component at the start of the OAuth flow.
- `success-location-hash`: If authentication is successful, the browser's `location.hash` will be set to this value. It can be used to redirect the user to their dashboard, for example.
- `success-url`: If authentication is successful, the browser's `location.href` will be set to this value. It can be used to redirect the user to their dashboard, for example.
- `extra-data`: This attribute can be used to pass additional data to the OAuth `getAccessTokenURL` endpoint. For Google OAuth, for example, an additional `redirect_uri` field is required, so it should be set using `extra-data="redirect_uri=http://localhost.com:8000/"` (substitute the URI with your own).
- `navigate-event-path`: If authentication is successful and a path is set via this attribute, the component will emit a `navigate` event which will bubble up the component hierarchy and can be used by a parent component to perform the success redirection. This approach can be used as an alternative to `success-location-hash` or `success-url` for doing the final redirect.
- `state-storage-key`: The state string is stored in the browser's `sessionStorage` under this key. Defaults to `oauth.state`. It must match the value provided to the `oauth-link` component at the start of the OAuth flow.
- `code-param-name`: The name of the query parameter which holds the `code` as provided by the OAuth provider within the OAuth callback URL. Defaults to `code`.
- `state-param-name`: The name of the query parameter which holds the `state` as provided by the OAuth provider within the OAuth callback URL. Defaults to `state`.
- `auth-timeout`: The number of milliseconds to wait for authentication to complete before timing out. Defaults to 10000 (10 seconds).
- `use-local-storage`: By default, the state is retrieved from sessionStorage. If this property is set, then the state will be retrieved from localStorage instead. This is useful for sharing the state across different tabs. If set, this attribute should also be set on the related `oauth-link` component.
