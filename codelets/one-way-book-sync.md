# Syncing sheets across books (one-way)

In these instructions, the books are named "primary" and "copy", and changes from "primary" are copied to "copy". Each book has a sheet named "Contacts", and these two sheets must have exactly the same column names. The name field of each Contacts sheet is "name".

To protect your webhook, it's a good idea to put some random string in the URL. In the code examples here, I've used "some-secret", but you should replace that with something harder to guess. It needs to be the same string in both codelets.

## Creating the webhook handler

First, go to the "copy" book, and create a new codelet. Click on the "API" button at the top-right, then click on the "Codelets" tab, and click "New Codelet". Name the codelet "sync-contacts".

Now you'll want to paste in the following code:

```js
var secret = 'some-secret';
var sheetSlug = 'contacts';

var Q = require('q');
var _ = require('underscore');
exports.endpoint = Q.async(function * (request, response) {
  // Check the secret
  if (request.query.secret !== secret) return {error: 'forbidden'};

  // Get changes to the contacts sheet
  var changes;
  try {
    changes = request.body.changes[sheetSlug];
  } catch (err) {
    console.log(err);
    return err.toString();
  }
  if (!changes) return 'Nothing to do';

  var i, record, id;

  // Handle creates
  var creates = changes.create || [];
  for (i = 0; i < creates.length; i++) {
    record = creates[i];
    yield client.create(sheetSlug, _.omit(record, 'id', 'record_url'));
  }

  // Handle updates
  var updates = changes.update || [];
  for (i = 0; i < updates.length; i++) {
    record = updates[i];
    id = yield findExistingRecordId(record.name);
    if (id == null) continue;
    yield client.update(sheetSlug, id, _.omit(record, 'id', 'record_url'));
  }

  // Handle deletes
  var deletes = changes.destroy || [];
  for (i = 0; i < deletes.length; i++) {
    record = deletes[i];
    id = yield findExistingRecordId(record.name);
    if (id == null) continue;
    yield client.destroy(sheetSlug, id);
  }

  return "Synced!"
})

var findExistingRecordId = function (name) {
  return client.list(sheetSlug, {name: name, limit: 1}).then(function (result) {
    if (result && result.items[0]) {
      return result.items[0].id;
    } else {
      console.log('No record found for name', name);
    }
  });
}
```

## Adding the webhook

Next, you need to set up a webhook on the "primary" book. The easy way to do this is to create a codelet in that book. The code for this is:

```js
var secret = 'some-secret';
var bookId = 'COPY_BOOK_ID';

exports.endpoint = function (request, response) {
  return client.createWebhook(`https://fieldbookcode.com/${bookId}/sync-contacts?secret=${bookId}`);
}
```

(Don't forget to replace "COPY_BOOK_ID" with the "copy" book's id, and "some-secret" with the same thing you used in the first codelet)

Save the codelet, then copy its URL and load it in a new tab. You should see something like:

```json
{"id":"57bdf3f11af08103003b1d4b","actions":["create","update","destroy"],"url":"https://fieldbookcode.com/COPY_BOOK_ID/sync-contacts?secret=some-secret"}
```

Once you've done that, you'll no longer need this second codelet, so you can delete it.
