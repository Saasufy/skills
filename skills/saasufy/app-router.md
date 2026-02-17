# app-router

A component which allows you build apps which have multiple pages. Inside it, you should specify a template for each page in your app along with the route/path to bind each page to.
This component also supports simple redirects as well as redirects based on the user's current authentication state. Redirects can be soft or hard; a soft redirect does not change the current URL/path, a hard redirect does.

**Import**

```html
<script src="https://saasufy.com/node_modules/saasufy-components/app-router.js" type="module" defer></script>
```

**Example usage**

```html
<app-router>
  <template slot="page" route-path="/:productName">
    <div class="product-title">Product: {{capitalize(productName)}}</div>
  </template>
  <div slot="viewport"></div>
</app-router>
```

This example shows how an infinite number of pages can be represented with a single route template.
In this case, the `/:productName` inside the `route-path` attribute of the slotted `<template slot="page">` element serves as a placeholder variable which can match any text.
You can then use such route variables inside the page template itself as shown with `<div class="product-title">Product: {{capitalize(productName)}}</div>` - In this case, the `productName` variable which comes from the route path is first capitalized using the `capitalize` utility function before it is rendered into the template.
Note that the `app-router` can contain any number of pages/routes so the `slot="page"` attribute can be re-used multiple times, e.g:

```html
<app-router>
  <template slot="page" route-path="/home">
    <div>This is the home page</div>
  </template>
  <template slot="page" route-path="/about-us">
    <div>This is the about-us page</div>
  </template>
  <template slot="page" route-path="/products">
    <div>This is the products page</div>
  </template>
  <div slot="viewport"></div>
</app-router>
```

This example shows how to redirect based on the user's authentication status:

```html
<app-router default-page="/home">
  <template slot="page" route-path="/log-in" auth-redirect="/home" hard-redirect>
    <log-in-form
      hostname="sas.saasufy.com"
      port="443"
      network-symbol="sas"
      chain-module-name="sas_chain"
      secure="true"
    ></log-in-form>
  </template>
  <template slot="page" route-path="/home" no-auth-redirect="/log-in">
    <div>This is the home page</div>
  </template>
  <div slot="viewport"></div>
</app-router>
```

Note that the app router path is based on `location.hash` so the path section in your browser's address bar needs to start with the `#` character.
So for example, if your `index.html` file is served up from the URL `http://mywebsite.com`, then, to activate the `/products` route as in the example above, you would need to type the URL `http://mywebsite.com#/products`.

Also note that the app router will only render once its `viewport` has been added and slotted into it.
If you want to make sure that some data (e.g. a config file) has been fully loaded before rendering the app, you could set the `slot="viewport"` attribute on your viewport element at a later time using JavaScript.

**Attributes of app-router**

- `default-page`: The path/route of the default page to send the user to in case no routes are matched. Commonly used to point to a `/page-not-found` page. For the home page, it's typically recommended to create a page template with `route-path=""` instead.
- `debounce-delay`: The number of milliseconds to wait before changing the route. This can help to avoid multiple renders if the route changes rapidly (e.g. when doing hard redirects). Defaults to 100ms.
- `target-page`: A convenience attribute which can be used to programmatically change the `location.hash` in the address bar.
- `watch-origin-route`: If this attribute exists, state changes of an origin route (before redirects) will trigger a re-render.
- `force-render-paths`: An optional list of comma-separated paths to force a page render/re-render, regardless of whether or not the page has changed.
- `greedy-refresh`: If this attribute exists, the component will refresh itself whenever an attribute is set, even if that attribute's value did not change.

**Attributes of slotted page templates**

- `route-path`: The path of the page. Supports custom URL parameters in the format `/org/:orgId/user/:userId`.
- `partial-route`: An optional attribute which, if specified, will allow the `route-path` to be matched partially from the start. This can be used to ignore the ending of a path which is not relevant to the page in order to avoid unnecessary re-renders.
- `redirect`: An optional path/route which this page should redirect to. Note that the content of this page will not be shown so it should always be empty.
- `no-auth-redirect`: An optional alternative path/route to redirect to if the user is on this page but they are not authenticated. If the user is authenticated, then the content of the template will be displayed as normal.
- `auth-redirect`: An optional alternative path/route to redirect to if the user is on this page but they are already authenticated. If the user is not authenticated, then the content of the template will be displayed as normal.
- `hard-redirect`: An optional, additional attribute which can be specified alongside a `redirect`, `no-auth-redirect` or `auth-redirect` attribute to ensure that any redirect will change the URL in the address bar. Redirects are `soft` redirects by default; this means that they redirect the user to the target page but keep the current URL path unchanged.
- `auto-reset-window-scroll`: If this attribute is present, the page scroll will be reset to the top when each new page is rendered.
