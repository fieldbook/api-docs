Fieldbook API
=============

The Fieldbook API lets you read and write records from any book you have access to, as simple JSON records.

Quick start
-----------

If you just want to dive in and get started, see the [quick start guide](quick-start.md).

Version
-------

The version number in the URL is `v1`, but in semver terms consider this ~v0.3. We expect the API to evolve rapidly.

Limitations
-----------

To get a few limitations out of the way up front:

* The API doesn't yet support formulas. We'll serve up your data, but if you want to do calculations you'll have to do them yourself for now.

* In particular we don't support multi-field record names, or numeric ID names (see [our docs](http://docs.fieldbook.com/docs/the-name-column) for more info on these options). If you've configured one of these options, you'll have to compute the name yourself from the other info we give you.

* We don't handle conflicts among field or sheet slugs. If you have two fields named the same thing in a sheet, you'll get an arbitrary one of them in your records. If you have two sheets named the same thing in a book... well, you're gonna have a bad time. Make names unique.

Authentication
--------------

### The basics

* Authentication is required on all API calls, except for GET requests to a book with public API access enabled.
* HTTPS is enforced; non-HTTPS requests will get a redirect to an HTTPS URL.
* Requests use HTTP basic auth. The username is an API key, and the password is the secret associated with that key (see below).

### API keys

* Manage API keys for a book by opening up the API console and using the “Manage API access” button.
* You can revoke a key by deleting it from the management UI.
* Right now keys are named key-1, key-2, etc. In the future we'll allow naming of keys.

### Public (read-only) access

Optionally, any book can be enabled for public (read-only) access. In this case, anyone can read data from the book without authentication.

Endpoints
---------

* Each book has a base URL displayed in the API management panel, like `https://api.fieldbook.com/v1/5643be3316c813030039032e`.

* Each sheet has an URL based on its title, like `https://api.fieldbook.com/v1/5643be3316c813030039032e/people`. (See below for how sheet titles are converted into slugs for these URLs. Again, don't name two sheets the same thing, or there will be conflicts!)

* Each record has an URL based on its sheet title and its short numeric ID, like `https://api.fieldbook.com/v1/5643be3316c813030039032e/people/3`.

* You can optionally append `.json` to any URL.

Sheet titles & field names
--------------------------

Sheet titles and field names in the API are based on their display names, but converted to lowercase, underscored identifiers without puncutation. E.g.:

* “Tasks” --> `tasks`
* “First name” --> `first_name`
* “City, state & zip” --> `city_state_zip`
* “First-time visitor?” --> `first_time_visitor`

Content types
-------------

Basically, everything is JSON:

* All requests must accept `application/json` responses.
* All request bodies must have a `Content-Type` of `application/json`.

Record objects
--------------

Throughout the API, records (rows) are represented as simple JSON objects.

The keys of the object are based on the field names, as described above. (Again, don't name two fields the same thing, or there will be conflicts!)

### In response bodies

* Each record also has an `id` key with a short integer ID. This can be used to retrieve the record. (Again, don't name a field “ID”, or the actual ID will be shadowed.)

* The values are JSON values you would expect: strings, numbers, etc.

* All numeric types are numbers. A currency value like $7 is read from the API as just 7. A percent value like 30% is read as 0.3.

* Dates are strings in the form YYYY-MM-DD; a day-of-year is in the form MM-DD.

* Checkboxes are returned as boolean values (`true` or `false`).

* Linked cells are arrays of objects with the short ID and name of the record. Other fields of the linked record are not populated right now.

* Empty cells are returned as `null`.

* Again, formula values are not included right now.

### In request bodies

* In general, values are interpreted the same way they would be if you typed them into a Fieldbook sheet, whether they are strings or numbers.

* If a field has an input type set on it, that will guide the parsing and interpretation of the value. A text field will coerce all values to strings; a currency field will turn numbers into currency values. Again, this is what would happen if you typed the input into the sheet in the UI.

* For a linked field, an object or a list of objects can be provided that reference linked records by id, e.g.: `[{"id": 1}, {"id": 2}]`. If an `id` is given, any other attributes are ignored.

* If an object without an `id` is given for a linked field, a new record is created in the linked sheet with the given attributes.

* A `null` value clears a cell.

* Values for formula fields are ignored.

Input validation
----------------

Create and update requests perform validation on the request bodies. An HTTP 400 Bad Request response will be returned if:

* Any key in the request body doesn't match a field in the sheet.
* Any value fails the validation for the input type (if any) of its field (for instance, passing a string to a number field).
* Any value is missing from a required field.
* An object or array is given for a non-link field.
* A value other than an object or array is given for a link field.

Requests
--------

### Read a sheet (list of records)

```GET https://api.fieldbook.com/v1/:book_id/:sheet_title```

Retrieves a list of all records in the sheet. Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/5643be3316c813030039032e/people
```

Response (HTTP 200 OK):

```
[
  {
    "id":1,
    "name":"Alice",
    "age":23,
    "city":[
      {
        "id":2,
        "name":"Chicago"
      }
    ]
  },
  {
    "id":2,
    "name":"Bob",
    "age":38,
    "city":[
      {
        "id":1,
        "name":"New York"
      }
    ]
  },
  {
    "id":3,
    "name":"Carol",
    "age":41,
    "city":[
      {
        "id":3,
        "name":"Los Angeles"
      }
    ]
  }
]
```

#### Sheet queries

You can filter the list using simple `key=value` query parameters. E.g., append `?name=Alice` to the previous example:

```
$ curl -u $KEY:$SECRET \
  https://api.fieldbook.com/v1/5643be3316c813030039032e/people?name=Alice
```

Response (HTTP 200 OK):

```
[
  {
    "id":1,
    "name":"Alice",
    "age":23,
    "city":[
      {
        "id":2,
        "name":"Chicago"
      }
    ]
  }
]
```

Note:

* The query is a case-sensitive exact match.
* For link fields, the value is matched against the display string of the linked cells (which is a comma-separated list in the case of multi-links).
* Queries on formula fields are not yet supported.

A more full-fledged query mechanism is coming in a future update.

#### Include or exclude fields

By default all fields are included in response objects. You can customize this with the `include` and `exclude` parameters. Each takes a comma-separated list of field keys. If `include` is passed, then only the named fields will be returned (plus the `id` field). If `exclude` is passed, then any named fields will be excluded. If the same field key is passed in both the include and exclude list, the exclude list will take priority.

For instance:

```
$ curl -u $KEY:$SECRET \
  https://api.fieldbook.com/v1/5643be3316c813030039032e/people?include=name,age
```

Response (HTTP 200 OK):

```
[
  {
    "id":1,
    "name":"Alice",
    "age":23
  },
  {
    "id":2,
    "name":"Bob",
    "age":38
  },
  {
    "id":3,
    "name":"Carol",
    "age":41
  }
]
```

Note that the `id` field is always included unless it is specifically excluded:

```
$ curl -u $KEY:$SECRET \
  https://api.fieldbook.com/v1/5643be3316c813030039032e/people?include=name&exclude=id
```

Response (HTTP 200 OK):

```
[
  {
    "name":"Alice"
  },
  {
    "name":"Bob"
  },
  {
    "name":"Carol"
  }
]
```

#### Pagination

You can paginate queries using the `limit` and `offset` query parameters:

* `limit=N` will limit the result object to at most N records
* `offset=K` will skip the first K records before returning anything

E.g., append `?limit=10` to a query to just get the first 10 records; append `?limit=10&offset=10` to get the second page of records (11–20), update the offset to 20 to get the third page, etc.

When either `limit` or `offset` are supplied, the response will not be an array but an object, with keys:

* `count`: The total number of records in the query result set.
* `offset`: The offset of the sub-list that is being returned (echoed from the supplied offset parameter, or 0 if none was supplied).
* `items`: An array of records, starting at the offset. If a limit is given, the array will be no longer than the limit.

If `offset + items.length < count`, then there are more records to load beyond what was returned; update the offset to fetch the next page.

Example:

```
$ curl -u $KEY:$SECRET \
  https://api.fieldbook.com/v1/5643be3316c813030039032e/people?limit=2
```

Response (HTTP 200 OK):

```
{
  "count":3,
  "offset":0,
  "items":[
    {
      "id":1,
      "name":"Alice",
      "age":23,
      "city":[
        {
          "id":2,
          "name":"Chicago"
        }
      ]
    },
    {
      "id":2,
      "name":"Bob",
      "age":38,
      "city":[
        {
          "id":1,
          "name":"New York"
        }
      ]
    }
  ]
}
```

### Read a record

```GET https://api.fieldbook.com/v1/:book_id/:sheet_title/:record_id```

Retrieves a single record. Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/5643be3316c813030039032e/people/1
```

Response (HTTP 200 OK):

```
{
  "id":1,
  "name":"Alice",
  "age":23,
  "city":[
    {
      "id":2,
      "name":"Chicago"
    }
  ]
}
```

#### Include or exclude fields

A GET request for a single record can take the same `include` and `exclude` parameters as a GET request for a sheet, see above.

### Create a record

```POST https://api.fieldbook.com/v1/:book_id/:sheet_title <JSON body>```

With a JSON body representing a single record, creates a record in the sheet. Returns the new record. Example:

```
$ curl -u $KEY:$SECRET -H "Content-Type: application/json" \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/people \
    -d '{"name":"Dave","age":19,"city":[{"id":1}]}'
```

Response (HTTP 201 Created):

```
{
  "id":4,
  "name":"Dave",
  "age":19,
  "city":[
    {
      "id":1,
      "name":"New York"
    }
  ]
}
```

The `Location` header of the response will have the new full URL of the record, which in this example is `https://api.fieldbook.com/v1/5643be3316c813030039032e/people/4`.

This appends the record to the sheet; inserting at an arbitrary location is not yet supported.

### Update a record

```PATCH https://api.fieldbook.com/v1/:book_id/:sheet_title/:record_id <JSON body>```

With a JSON body containing any attributes for a record, updates the record. Only the keys included in the body are updated; other keys are left alone. Returns the full updated record. Example:

```
$ curl -u $KEY:$SECRET -H "Content-Type: application/json" -X PATCH \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/people/1 \
    -d '{"age":24,"city":[{"name":"Boston"}]}'
```

Response (HTTP 200 OK):

```
{
  "id":1,
  "name":"Alice",
  "age":24,
  "city":[
    {
      "id":4,
      "name":"Boston"
    }
  ]
}
```

Update via `PUT` is not currently supported.

### Delete a record

```DELETE https://api.fieldbook.com/v1/:book_id/:sheet_title/:record_id```

Deletes a record. Example:

```
$ curl -u $KEY:$SECRET -X DELETE \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/people/1
```

Response: HTTP 204 No Content (empty body).

Webhooks
--------

Fieldbook supports webhook callbacks on record create, update and delete events.

### Key things to know

* Webhooks are on a per-book basis. When you register a webhook callback, it will listen for changes in all sheets.

* Webhooks are sent about record data changes. There are no meta-data webhooks yet.

* Each registered webhook can listen for create, update, and/or delete events. However, each callback may include multiple types of changes, across multiple records. In general, we try to send one callback for each action a user takes in the system, so bulk actions can cause multiple types of updates at once, and to multiple records. For instance, pasting a block of values into a sheet can update multiple records and also create multiple new records, in a single event.

### Registering a webhook

To register a webhook, POST a webhook body to the webhooks collection for a book:

```POST https://api.fieldbook.com/v1/:book_id/meta/webhooks <JSON body>```

The request body:

* Must have an `url` key with a callback URL.
* May optionally have an `actions` key containing an array of actions to listen for. The actions are `create`, `update`, and `destroy`. If omitted, the callback will fire on all actions.

Example:

```
$ curl -u $KEY:$SECRET -H "Content-Type: application/json" -X POST \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/meta/webhooks \
    -d '{"url":"https://you.com/your/callback","actions":["create", "update"]}'
```

Response (HTTP 200 OK):

```
{
  "id": "56b01529cf979181cfe28941",
  "url": "https://you.com/your/callback",
  "actions": [
    "create",
    "update"
  ],
}
```

### Callback format

On each relevant event, a callback will be POSTed to the callback URL for each webhook. The request body contains:

* The `webhookId` that the callback is for
* A `user` hash with basic details about the user who made the change
* A `changes` hash with one key per sheet that was affected, each of which has:
    - Optionally, a `create` array of created records
    - Optionally, an `update` array of updated records
    - Optionally, a `destroy` array of deleted records.

Example:

```
{
  "webhookId": "56b01529cf979181cfe28941",
  "user": {
    "email": "jason@fieldbook.com",
    "name": "Jason Crawford",
    "id": "56a97a97242dce2ee012a9ad"
  },
  "changes": {
    "people": {
      "create": [
        {
          "id": 4,
          "name": "Dave",
          "age": 54,
          "city": [
            {
              "id": 1,
              "name": "New York"
            }
          ]
        }
      ],
      "update": [
        {
          "id":3,
          "name":"Carol",
          "age":42,
          "city":[
            {
              "id":3,
              "name":"Los Angeles"
            }
          ]
        }
      ]
    }
  }
}
```

### Delivery caveats

Some caveats on the delivery of webhook callbacks:

* Delivery order is not guaranteed.

* The record data in each callback is the latest data for that record at the time the callback was generated. This will usually correspond to the action taken by the user, but in some conditions may not. For instance, if a record is created by one user and then quickly updated by another user, it is possible that the create callback will contain the data as edited by the second user.

* If the callback request receives an error response, we will retry at least once. However, we don't currently guarantee any particular number of retries or timing of the retries.

* Callbacks are designed to notify of changes to record data, not metadata. They should have reasonable behavior in the face of metadata changes, but we do not advise relying too closely on any particular webhook behavior around metadata.

### Deleting (de-registering) a webhook

To de-register a webhook, just DELETE it. Example:

```
$ curl -u $KEY:$SECRET -H "Content-Type: application/json" -X DELETE \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/meta/webhooks/56b01529cf979181cfe28941
```

The response will be HTTP 204 No Content.

### Listing webhooks

To list all webhooks on a book, just GET the collection. Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/5643be3316c813030039032e/meta/webhooks
```

Response (HTTP 200 OK):

```
[
  {
    "id": "56b01529cf979181cfe28941",
    "url": "https://you.com/your/callback",
    "actions": [
      "create",
      "update"
    ],
  },
  {
    "id": "564125b564b72f4fd3ed9dba",
    "url": "https://you.com/other/callback",
    "actions": [
      "destroy"
    ],
  }
]
```

### Webhook security

If you want to protect your webhook receiving endpoints, you can add basic auth parameters to the callback URL, like this:

```
https://user:password@example.com/your/callback
```

Callback URLs are encrypted when stored in our database, in order to protect these credentials.

HTTPS is required when using this format. (However, even if you're not using a username/password, we recomend using HTTPS in callback URLs.)

Method override
---------------

If you're using an HTTP client that for some reason doesn't support all HTTP methods (such as PATCH), you can do a POST instead and specify the X-HTTP-Method-Override header. Example:

```
$ curl -u $KEY:$SECRET -H "Content-Type: application/json" \
    -H "X-HTTP-Method-Override: PATCH" -X POST \
    https://api.fieldbook.com/v1/5643be3316c813030039032e/people/1 \
    -d '{"age":24,"city":[{"name":"Boston"}]}'
```

This will be interpreted as a PATCH.

The method override header only works with POST requests.

Rate limits
-----------

To prevent runaway API clients from causing sudden shocks to the system and degrading performance for everyone, we limit the API to 5 simultaneous outstanding requests from any given API key. Beyond that limit, requests may receive an HTTP 429 Too Many Requests error response.

* If you are writing a script that uses the API to process many records in batch, we recommend that you serialize your API calls: wait for one call to finish before firing off the next call. This is the simplest way to avoid any rate limit errors.

* If you have a server or codelet that uses our API and needs a higher rate limit, contact us: support@fieldbook.com

Note that sheet names are lowercase only and spaces need to be replaced with underscores `_`, so if you have a sheet named `My Sheet` the list of records would be available at `GET https://api.fieldbook.com/v1/:book_id/my_sheet`


Future work
-----------

There are a lot of things we're thinking about supporting in the future; shoot us a note at support@fieldbook.com to let us know which of these are most important to your needs:

* Support for full [queries](http://docs.fieldbook.com/docs/queries)
* Including [formulas](http://docs.fieldbook.com/docs/formulas) (calculated/derived values) in responses
* Read-only API keys
