# Template utility functions

The following utility functions can be used anywhere inside template `{{expression}}` placeholders:

- Generate a unique random ID in UUID format: `{{uuid.v4()}}`
- Generate a deterministic ID in UUID-compatible format derived from one or more values: `{{computeId('value1', 'value2')}}`
- Convert text to a format suitable for use inside a URL: `{{url('This is a test')}}`
- Convert text to lower case: `{{lowerCase('TEST')}}`
- Convert text to upper case: `{{upperCase('test')}}`
- Convert first letter of text to upper case: `{{capitalize('test')}}`
- Remove leading and trailing spaces from text: `{{trim('   test   ')}}`
- Specify one or more fallback values in case values are null or undefined: `{{fallback(Product.imageSrc, 'path/to/default-image.png')}}`
- Format UNIX timestamp as a human-readable date: `{{date(Product.updatedAt)}}`
- Make a string safe by escaping HTML characters: `{{safeString(Product.name)}}`
- Given an array of objects, extract the values from the specified field and join them together into a single comma-separated string: `{{joinFields(Product, 'name')}}`

### General utility functions

The following utility functions are available for advanced use cases and can be imported from the `saasufy-components` module:

**Import from CDN:**
```javascript
import {
  getRecordIds,
  getRecord,
  getRecordCount,
  generateRecords,
  getRecords,
  logAttributeChanges,
  wait
} from 'https://saasufy.com/node_modules/saasufy-components/utils.js';
```

**Import from npm:**
```javascript
import {
  getRecordIds,
  getRecord,
  getRecordCount,
  generateRecords,
  getRecords,
  logAttributeChanges,
  wait
} from 'saasufy-components/utils.js';
```

A reference to a socket can be obtained from a `socket-provider` element like this:

```javascript
let socket = document.querySelector('socket-provider').getSocket();
```

**Available functions:**

- `getRecordIds({ socket, type, viewName, viewParams, startPage, maxPages, pageSize })`: Retrieves record IDs from a collection view across multiple pages. Returns an array of IDs.
- `getRecord({ socket, type, id, fields })`: Fetches a single record by ID. If fields are specified, only those fields are retrieved; otherwise the entire record is returned.
- `getRecordCount({ socket, type, viewName, viewParams, offset })`: Gets the total count of records in a view. Returns a number representing the count.
- `generateRecords({ socket, type, viewName, viewParams, fields, startPage, pageSize, maxAttempts })`: An async generator function that yields records one by one from a collection view. Useful for processing large datasets efficiently.

**Example usage:**
```javascript
// Process all products one by one without loading them all into memory at once
let productGenerator = generateRecords({
  socket,
  type: 'Product',
  viewName: 'alphabeticalView',
  viewParams: { category: 'electronics' },
  fields: [ 'name', 'price', 'qty' ],
  pageSize: 50
});

for await (let product of productGenerator) {
  console.log(`Processing product: ${product.name}`);
  // Process each product individually
}
```
- `getRecords({ socket, type, viewName, viewParams, fields, startPage, maxPages, pageSize })`: Retrieves multiple records from a collection view. Returns an array of record objects.
- `logAttributeChanges(...elementSelectors)`: Logs attribute changes for DOM elements to the console. Useful for debugging. Returns a function to stop observing.
- `wait(duration)`: Creates a promise that resolves after the specified duration in milliseconds. Useful for adding delays in async operations.

