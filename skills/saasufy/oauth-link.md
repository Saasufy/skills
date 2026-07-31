# oauth-link

A link to initiate an OAuth flow to authenticate a user as part of log in. Can support a range of different OAuth providers.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/oauth-link.js" type="module" defer></script>
```

**Example usage**

```html
<oauth-link class="log-in-oauth-section" provider="github" client-id="dabb51f8af31a0bc53bf">
  <template slot="item">
    <a href="https://github.com/login/oauth/authorize?client_id={{oauth.clientId}}&state={{oauth.state}}">
      Log in with GitHub
    </a>
  </template>
  <div slot="viewport"></div>
</oauth-link>
```

**Attributes**

- `provider` (required): The name of the provider. It must match the provider name specified on the `Authentication` page of your Saasufy control panel and also the value provided to the `oauth-handler` component at the end of the OAuth flow.
- `client-id` (required): This is the client ID from the third-party OAuth provider.
- `redirect-uri`: A URI to redirect the user to once OAuth is successful. This attribute may be required depending on the OAuth provider/client.
- `grant-type`: The grant type for this OAuth flow. Typically set to `authorization_code`. This may be required depending on the OAuth provider/client.
- `state-size`: This allows you to control the size (in bytes) of the random state string which is passed to the OAuth provider as part of the OAuth flow. Defaults to 20.
- `state-storage-key`: The state string is stored in the browser's `sessionStorage` under this key. Defaults to `oauth.state`. It must match the value provided to the `oauth-handler` component at the end of the OAuth flow.
- `use-session-storage`: By default, the state is stored inside localStorage. If this property is set, then the state will be stored inside sessionStorage instead. If set, this attribute should also be set on the related `oauth-handler` component.
