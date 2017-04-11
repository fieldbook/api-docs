Metadata API
============

The Fieldbook metadata API lets you read the structure of a book (sheets and fields) so you can adapt to dynamic or unknown schemas. Example use cases:

* Creating a custom input form for a sheet that automatically includes all its fields
* Creating custom notifications that automatically include all fields
* Building a third-party integration that works with arbitrary customer data

Authentication
--------------

See the main API reference: [Authentication](reference.md#authentication)

Content types
-------------

As in the main API, everything is JSON; be sure to include header `Accept: application/json`.

Getting book info
-----------------

```GET https://api.fieldbook.com/v1/books/:book_id```

Retrieves basic info about a book, mainly its title. Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/books/58e1a67f5662a603001916eb
```

Response (HTTP 200 OK):

```
{
  "id": "58e1a67f5662a603001916eb",
  "title": "Order Tracking",
  "url": "https://fieldbook.com/books/58e1a67f5662a603001916eb"
}
```

Getting sheet info
------------------

```GET https://api.fieldbook.com/v1/books/:book_id/sheets```

Lists sheets on a book. The response is an array of objects, each with:

* `id`: permanent globally unique sheet id
* `title`: sheet display name
* `slug`: what you use to read and write records through the main API
* `url`: Fieldbook web link for the sheet

Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/books/58e1a67f5662a603001916eb/sheets
```

Response (HTTP 200 OK):

```
[
  {
    "id": "58e1a67f5662a603001916ed",
    "title": "Products",
    "slug": "products",
    "url": "https://fieldbook.com/sheets/58e1a67f5662a603001916ed"
  },
  {
    "id": "58e1aa1ec053ce0300bd2cf1",
    "title": "Orders",
    "slug": "orders",
    "url": "https://fieldbook.com/sheets/58e1aa1ec053ce0300bd2cf1"
  },
  {
    "id": "58e1ac185662a6030019170e",
    "title": "Line items",
    "slug": "line_items",
    "url": "https://fieldbook.com/sheets/58e1ac185662a6030019170e"
  }
]
```

Getting field info
------------------

```GET https://api.fieldbook.com/v1/sheets/:sheet_id/fields```

Lists fields in a sheet. The response is an array of objects, each with:

* `key`: permanent field id, unique only within a sheet
* `name`: display name for the field
* `slug`: field slug used in reading/writing records via the main API
* `fieldType`: data, link or formula
* `inputType`: data input type, if fieldType=data
* `required`: boolean, may be omitted if false
* `enum`: for pick list fields, this is the choice list in order

The inputType corresponds to our [data input types](http://docs.fieldbook.com/docs/data-types), valid values:

* generic
* text
* number
* currency
* percent
* date
* boolean
* picklist
* image
* file
* email
* website
* dayofyear

Example:

```
$ curl -u $KEY:$SECRET https://api.fieldbook.com/v1/sheets/58e1a67f5662a603001916ed/fields
```

Response (HTTP 200 OK):

```
[
  {
    "key": "f0",
    "name": "Name",
    "required": true,
    "slug": "name",
    "fieldType": "data",
    "inputType": "generic"
  },
  {
    "key": "f1",
    "name": "Description",
    "slug": "description",
    "fieldType": "data",
    "inputType": "text"
  },
  {
    "key": "f2",
    "name": "Price",
    "slug": "price",
    "fieldType": "data",
    "inputType": "currency"
  },
  {
    "key": "f3",
    "name": "Status",
    "slug": "status",
    "fieldType": "data",
    "inputType": "picklist",
    "enum": [
      "Available",
      "Out of stock",
      "Discontinued"
    ]
  },
  {
    "key": "f4",
    "name": "Line items",
    "slug": "price",
    "fieldType": "link"
  }
]
```

Read-only
---------

The metadata API is read-only right now; you can explore the structure of a book but you can't create new books or sheets or alter structure yet. We plan to add a read/write metadata API in the future.
