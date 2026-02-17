---
name: saasufy
description: This skill helps you to setup, build, deploy and test your Saasufy application. It is an essential skill to use when building Saasufy applications or when using Saasufy components. You should read relevant linked documentation before creating any Saasufy component or integrating any feature of Saasufy.
---

# Saasufy

## Setup

[setup.md](setup.md): This is an essential guide to read before starting to build an application.

## Schema Management

[schema-management.md](schema-management.md): An important guide which explains how to create, read, update and delete records to define the data schema of a Saasufy service via the Admin HTTP API. It provides exact curl commands for managing the schema. The schema is also where authorization/access-control rules are defined so it's an integral part of enforcing security constraints.

## Data Management

[data-management.md](data-management.md): A guide which explains how to create, read, update and delete records within a model collection in the user's service (once deployed). This can be useful for adding some dummy data into the collections to aid with testing. It provides curl commands for managing data within any collection.

## Components

This important section contains documentation for how to use frontend Saasufy components/elements. It's essential to read the documentation of the relevant component before using it. These components are designed to work with each other declaratively and should not need code aside for advanced use cases.

### Application State Management

- [app-router.md](app-router.md): Shows how to handle frontend routing logic to render different pages and sections within the app based onthe URL.
- [socket-provider.md](socket-provider.md): Shows how to setup the app to allow components to connect with the Saasufy backend for synching.

### Handling a Collection of Records

[collection-viewer.md](collection-viewer.md): For displaying and managing collections of records. Provides filtering capabilities by binding to Saasufy views (`ModelView` abstraction).
[collection-adder.md](collection-adder.md): A simple form component for adding records to a collection.
[collection-adder-form.md](collection-adder-form.md): An advanced form component for adding records to a collection which provides full control over the form's layout and child input elements.
[collection-deleter.md](collection-deleter.md): A component designed to be slotted inside a `collection-viewer` to delete records from the collection; can optionally support a confirmation step by slotting a `confirm-modal` into the same `collection-viewer`.
[collection-reducer.md](collection-reducer.md): For flexibly reducing records within a collection down to a single aggregated object. Provides filtering capabilities by binding to Saasufy views (`ModelView` abstraction).

### Handling a Single Record

[model-input.md](model-input.md]): A component which binds to a single field/property of a model record and can be used to update a model field's value in real time. It also updates itself in realtime.
[model-text.md](model-text.md): A read-only component which binds to a single field/property of a model record and updates in realtime.
[model-viewer.md](model-viewer.md): A read-only component which binds to a model record and lets you render multiple fields which update in realtime whenever any of those fields change on the backend.

### User Input

[input-provider.md](input-provider.md): A declarative input component which can be bound to other components to update their attributes whenever its value changes.
[input-combiner.md](input-combiner.md): A declarative hidden component which can be bound to other components. It can receive multiple values from different components, aggregate them and forward the combined value to a different component.
[input-transformer.md](input-transformer.md): A declarative hidden component which can be bound to other components to transform input values before they are fed to those components.

### Authentication

[log-in-form.md](log-in-form.md): A login form component.
[log-out.md](log-out.md): A logout link component.
[oauth-link.md](oauth-link.md): A flexible OAuth link component which forwards users to a specific OAuth provider and can be configured to work with any supported OAuth provider.
[oauth-handler.md](oauth-handler.md): A flexible OAuth handler component which can handle the response/redirect from any supported OAuth provider and, upon successful authentication via the Saasufy backend, will redirect the user to the appropriate URL.

### Display Groups

[render-group.md](render-group.md): Can be used to group multiple data-driven components together in such a way that they will only render when all of them have finished loading. This can be used to better control the initial rendering of a page.
[if-group.md](if-group.md): Used to conditionally display child elements based on the result of a JavaScript expression.
[switch-group.md](switch-group.md): Used to conditionally display child elements with multiple possible cases based on the result of a JavaScript expression.
[collection-adder-group.md](collection-adder-group.md): A wrapper which can be used with `collection-adder` components to make them behave as one; for example, to allow creating multiple collection resources with a single button click.

### Modal Windows

Note that modal elements use the shadow DOM so you should read the relevant docs below in order to style them correctly with CSS.

[confirm-modal.md](confirm-modal.md): A modal component which is designed to be slotted inside a `collection-viewer` component and intended to work alongside the `collection-deleter` component; the modal is triggered automatically by the `collection-viewer` when the `collection-deleter` is clicked. It asks the user for confirmation with a custom message and, if confirmed, it will trigger an event on the `collection-viewer` which will perform the deletion.
[overlay-modal.md](overlay-modal.md): A general-purpose overlay modal component.

## Authorization/Access Control

[access-control.md](access-control.md): Provides detailed information about how to create access control rules in Saasufy via the Admin HTTP API.

## Debugging and Troubleshooting

[debugging-and-troubleshooting.md](debugging-and-troubleshooting.md): A guide with a comprehensive list of common issues. This is particularly useful for resolving issues such as missed realtime updates or loss of focus of input elements.

## Deployment and Testing

[deployment-and-testing.md](deployment-and-testing.md): A guide which describes Saasufy's deployment process and explains how to test.

## File Hosting

[file-hosting.md](file-hosting.md): Describes how to upload files to Saasufy and how to view them as static files over HTTP.

## Search Filtering and Querying

[search-filtering-querying.md](search-filtering-querying.md): A guide for leveraging the advanced search and filtering capabilities of Saasufy.

## Utility Functions

[utility-functions.md](utility-functions.md): Describes utility functions which are provided by Saasufy to help manage data with JavaScript. It provides utility functions for template rendering as well as utility functions for fetching filtered collections in various ways; for example with pagination, filtering etc... The JavaScript utility functions are intended to be used as a last-resort for fetching data for frontend processing. In the vast majority of cases, for displaying filtered views, the `collection-viewer` component should be used instead since those utility functions do not provide realtime updates.

## Saasufy WebSocket API

[websocket-api.md](websocket-api.md): This guide describes how to interact with Saasufy via WebSockets directly using the SocketCluster client. It also explains how Saasufy's pub/sub mechanism works and how to subscribe to realtime changes using code. This API is intended to be used as a last-resort where declarative components are unsuitable. It's recommended to also check out the [utility-functions.md](utility-functions.md) guide as it provides more user-friendly functions which wrap the client socket instance.
