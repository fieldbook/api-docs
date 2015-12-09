Fieldbook API – BETA
====================

Thanks for trying out the first version of the Fieldbook API. It's very early and the API is very basic, but we're putting it out there for your feedback and in the hopes that even this early version will be useful to you.

Quick start
-----------

If you just want to dive in and get started, see the [quick start guide](quick-start.md).

Version
-------

The version number in the URL is `v1`, but in semver terms consider this ~v0.2. We expect the API to evolve rapidly.

Limitations
-----------

Just to get a few limitations out of the way up front:

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
  },
]
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

Future work
-----------

There are a lot of things we're thinking about supporting in the future; shoot us a note at support@fieldbook.com to let us know which of these are most important to your needs:

* Pagination for large sheets
* Support for [queries](http://docs.fieldbook.com/docs/queries)
* Including [formulas](http://docs.fieldbook.com/docs/formulas) (calculated/derived values) in responses
* Webhooks (callbacks on edit events)
* Read-only API keys
