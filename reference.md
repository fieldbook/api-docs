Fieldbook API – BETA
====================

Thanks for trying out the first version of the Fieldbook API. It's very early and the API is very basic, but we're putting it out there for your feedback and in the hopes that even this early version will be useful to you.

Quick start
-----------

If you just want to dive in and get started, see the [quick start guide](quick-start.md).

Version
-------

The version number in the URL is `v1`, but in semver terms consider this v0.1. We expect the API to evolve rapidly.

Limitations
-----------

Just to get a few limitations out of the way up front:

* The API is read-only. Write access coming later.

* The API doesn't yet support formulas. We'll serve up your data, but if you want to do calculations you'll have to do them yourself for now.

* In particular we don't support multi-field record names, or numeric ID names (see [our docs](http://docs.fieldbook.com/docs/the-name-column) for more info on these options). If you've configured one of these options, you'll have to compute the name yourself from the other info we give you.

* We don't handle conflicts among field or sheet slugs. If you have two fields named the same thing in a sheet, you'll get an arbitrary one of them in your records. If you have two sheets named the same thing in a book... well, you're gonna have a bad time. Make names unique.

Authentication
--------------

### The basics

* Authentication is required on all API calls.
* HTTPS is enforced; non-HTTPS requests will get a redirect to an HTTPS URL.
* Requests use HTTP basic auth. The username is an API key, and the password is the secret associated with that key (see below).

### API keys

* See the [quick start guide](quick-start.md) for how to manage API keys.
* You can revoke a key by deleting it from the management UI.
* Right now keys are named key-1, key-2, etc. In the future we'll allow naming of keys.

Endpoints
---------

* Each book has a base URL displayed in the API management panel, like `https://api.fieldbook.com/v1/5643be3316c813030039032e`.

* Each sheet has an URL based on its name, like `https://api.fieldbook.com/v1/5643be3316c813030039032e/people`. (Again, don't name two sheets the same thing, or there will be conflicts!)

* Each record has an URL based on its sheet name and its short numeric ID, like `https://api.fieldbook.com/v1/5643be3316c813030039032e/people/3`.

* You can optionally append `.json` to any URL.

Content types
-------------

Basically, everything is JSON. All requests must accept `application/json` responses.

Record objects
--------------

* Throughout the API, records (rows) are represented as simple JSON objects.

* The keys of the object are based on the field names. (Again, don't name two fields the same thing, or there will be conflicts!)

* Each record also has an `id` key with a short integer ID. This can be used to retrieve the record. (Again, don't name a field “ID”, or the actual ID will be shadowed.)

* The values are JSON values you would expect: strings, numbers, etc.

* All numeric types are numbers. A currency value like $7 is read from the API as just 7. A percent value like 30% is read as 0.3.

* Dates are strings in the form YYYY-MM-DD; a day-of-year is in the form MM-DD.

* Linked cells are arrays of objects with the short ID and name of the record. Other fields of the linked record are not populated right now.

* Again, formula values are not included right now.

Requests
--------

This version of the API is read-only, so there are really only two calls:

* `GET https://api.fieldbook.com/v1/:book_id/:sheet_name` retrieves a list of all records in the sheet.
* `GET https://api.fieldbook.com/v1/:book_id/:sheet_name/:record_id` retrieves a single record.

Future work
-----------

There are a lot of things we're thinking about supporting in the future; shoot us a note at support@fieldbook.com to let us know which of these are most important to your needs:

* Write access
* Pagination for large sheets
* Support for [queries](http://docs.fieldbook.com/docs/queries)
* Including [formulas](http://docs.fieldbook.com/docs/formulas) (calculated/derived values) in responses
* Webhooks (callbacks on edit events)
* Read-only API keys
* Public (unauthenticated) API access to public books
