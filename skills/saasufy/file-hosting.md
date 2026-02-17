# File Hosting

Saasufy provides basic HTTP/HTTPS file hosting functionality with support for client-side caching (with `ETag`).
This is especially useful for hosting images.

To upload files to Saasufy, you first need to create a field of type `string` on a model of your choice.
You will then need to check/enable the `blob` constraint to tell Saasufy to treat this field as a base64 file.

After you've deployed your schema from the Saasufy dashboard, you will be able to manually add new records into your model via its `data` section.
Saasufy will show you a file picker next to the relevant field name which you can use to upload a file/image to Saasufy.

After adding a record to Saasufy which holds a file in one of its fields, that file can be accessed over HTTP/HTTPS via your Saasufy service's `/files` HTTP endpoint.
The format of the URL to link directly to a specific file/image is:

```
https://saasufy.com/:serviceId/files/:modelName/:modelId/:fieldName
```
For example, if the URL for your Saasufy service (which you get after deploying your service) is `wss://saasufy.com/sid7999/socketcluster/`
and your file is stored inside an `Image` model with ID `58f2051c-14bc-4518-8816-ff387cfdd57e` inside a `data` field, you will be able to link to your image directly using this URL:

```
https://saasufy.com/sid7999/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data
```

You can use it to embed images into your application using the `<img>` tag like this:

```html
<img src="https://saasufy.com/sid7999/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data" alt="My image" />
```

Saasufy enforces access controls for HTTP in the same way as it does for its WebSocket protocol.
It's possible to block or restrict access to specific files stored on specific fields via the `Access` page under the relevant model.

If your model field's access is set to `restrict`, only authenticated users with matching permissions will be able to view/download the image/file.
In this case, you will need to add your user's JWT token to a cookie with the name `socketcluster.saasufyService.authToken` and it will be passed along with the request.
