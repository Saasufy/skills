# log-out

A component which can be placed inside a `socket-provider` to deauthenticate the socket (e.g. on click).

If the user logged in via an OAuth provider which supports RP-initiated log out (such as Keycloak), deauthenticating the socket does not end the session on the provider itself; the user would be logged straight back in without being asked for their credentials the next time they log in. To also end the provider session, specify the `logout-url` attribute and the component will redirect the user to that endpoint after deauthenticating the socket.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/log-out.js" type="module" defer></script>
```

**Example usage**

```html
<log-out onclick="logOut()"><a href="javascript:void(0)">Log out</a></log-out>
```

Logging out of Keycloak as well as of the app:

```html
<log-out
  onclick="logOut()"
  provider="keycloak"
  logout-url="https://auth.example.com/realms/myrealm/protocol/openid-connect/logout"
  client-id="myclient"
  post-logout-redirect-uri="https://myapp.com/index.html"
><a href="javascript:void(0)">Log out</a></log-out>
```

**Attributes**

- `logout-url`: The log out endpoint of the OAuth provider; for OpenID Connect providers this is the `end_session_endpoint`. If it is not specified, the component only deauthenticates the socket. The redirect is skipped if the user did not log in via an OAuth provider.
- `provider`: The name of the OAuth provider, as specified on the `Authentication` page of your Saasufy control panel. If it is set, the redirect only happens when the user logged in via that specific provider; this is useful when your app offers several different ways to log in.
- `client-id`: The client ID to pass to the provider as the `client_id` query parameter. OpenID Connect providers require either this or an ID token in order to log the user out without asking them to confirm.
- `post-logout-redirect-uri`: The URL to send the user back to once the provider has logged them out, passed as the `post_logout_redirect_uri` query parameter. It usually needs to be registered with the provider in advance. If it is not specified, the user is left on the provider's own logged out page.

If the auth token contains an ID token (which requires the `idTokenField` of the provider to be set on the `Authentication` page of your Saasufy control panel), it is passed to the provider as the `id_token_hint` query parameter. Providers such as Keycloak show a log out confirmation page when it is missing. Any of these query parameters which are already present in `logout-url` are left untouched.

The component decides whether to redirect based on the `authSource` field of the user's auth token; see [account-table.md](account-table.md) for what that field holds. Sessions which were not created via an OAuth provider (e.g. via [log-in-form.md](log-in-form.md)) are only deauthenticated locally.

**Using the shared Saasufy Keycloak instance**

If you authenticate against the shared Keycloak instance described in [authentication.md](authentication.md), the `logout-url` is the `logout` endpoint of the tenant realm and the `client-id` is the automatically generated `saasufy-${accountId}` client ID of your `OAuthProvider` record:

```html
<log-out
  onclick="logOut()"
  provider="keycloak"
  logout-url="https://auth.saasufy.com/realms/tenant/protocol/openid-connect/logout"
  client-id="saasufy-123"
  post-logout-redirect-uri="https://myapp.com/index.html"
><a href="javascript:void(0)">Log out</a></log-out>
```

The exact endpoints for the shared instance can be found under the `keycloak` entry of https://saasufy.com/oauth-settings.js (use the tenant instance, not the master one). The default `keycloak` provider configuration already sets `idTokenField`, so the `id_token_hint` is passed automatically and the user is not shown a confirmation page.
