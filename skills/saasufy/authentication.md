# Saasufy Authentication

## When to Use This Skill

Use this skill when you need to:
- Let users log in or sign up for your application
- Add or configure an OAuth provider (e.g. GitHub, Google, Keycloak)
- Support email signups using the shared Saasufy Keycloak instance
- Understand which fields end up on a user's JWT for use by access control rules

For rules about what an authenticated user is then allowed to do, see [access-control.md](access-control.md).

## Authentication

The API key must be read from the `.saasufy-api-key` file in the repository root. Always read this file first before making API calls.

```bash
SAASUFY_API_KEY=$(cat .saasufy-api-key)
```

## Base URL

All Admin HTTP API endpoints use: `https://saasufy.com/api/`

## OAuth

Saasufy exposes OAuth providers on the control panel and via the Admin HTTP API through the `OAuthProvider` model. This is how you can add new OAuth providers and modify existing ones (e.g. to set the client ID, secret and redirectURI). See [schema-management.md](schema-management.md) and [data-management.md](data-management.md) guides for info about how to do this using the Admin HTTP API.

Saasufy can integrate with essentially any OAuth provider; most aspects of an OAuth flow can be customized declaratively by specifying what field names to use in HTTP requests and which ones to extract from responses.

Saasufy provides default configurations for a few OAuth providers including 'github', 'google' and 'keycloak'. For Keycloak, it's possible to set up your own self-hosted instance or you can use the existing shared instance hosted by Saasufy at https://auth.saasufy.com.

### Email Signups via the Shared Keycloak Instance

If you want to support email signups in your application, the simplest approach is to use the existing shared Saasufy instance.
To integrate, you should create a new `OAuthProvider` with `providerName` set to `keycloak` and you should provide a URL for your application as the `redirectURI`; this should point to the page where you placed your `oauth-handler` component (which should itself provide the same or matching URI via its `redirect-uri` attribute).

Creating the `keycloak` provider will automatically create a corresponding OAuth client inside the Saasufy Keycloak instance with the correct default values; the only value which should be set explicitly for this use case is the `redirectURI`. Changing the `redirectURI` will automatically update the corresponding redirect URI for that OAuth client inside the Keycloak instance. When using `keycloak` as the provider, the `providerClientId` and `providerClientSecret` will be created automatically and does not need to be specified when creating the `OAuthProvider` record. By default, a client ID in the format `saasufy-${accountId}` will be generated and all changes done to this client (with that ID) will be reflected inside the Keycloak instance; if any other `providerClientId` is used, the user will need to create the OAuth client manually inside the Keycloak instance; however, this is not possible on the shared Saasufy instance. You can change the `providerClientSecret` and `redirectURI` at any time; doing so will cause them to be updated inside the Keycloak instance (again, provided that `saasufy-${accountId}` is used as the `providerClientId`).

Note that deleting and re-creating an `OAuthProvider` record with the `providerName` set to `keycloak` will automatically reset the client in the Keycloak instance with the original (automatic) `providerClientId` and a newly generated secret; then you just need to provide the `redirectURI` again.

After making changes to `OAuthProvider` records, remember to deploy the changes on Saasufy.

### Frontend Components

The OAuth flow is driven from the frontend by two components which must agree on the `provider` name and on the `state-storage-key`:

- [oauth-link.md](oauth-link.md): Starts the flow by forwarding the user to the OAuth provider.
- [oauth-handler.md](oauth-handler.md): Handles the response/redirect coming back from the provider and, on success, redirects the user to the appropriate URL.

For username/password login against Saasufy itself, see [log-in-form.md](log-in-form.md) and [log-out.md](log-out.md).
