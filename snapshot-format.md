Fieldbook Snapshot Format
=========================

Overview
--------

The Fieldbook snapshot format captures almost everything about a book (see what is included below).

Audience
--------

This documentation is for programmers who would like to parse and work with a Fieldbook snapshot. It may be too technical for other users. If you don't have technical skills or a technical person to help you, it will be easier for you to work with your data in Excel format, which you can also download.

What is included
----------------

* Structure of sheets, columns, types, links and formulas
* Saved searches and forms
* All rows and links
* Links to attachments

Not captured:

* Webhooks
* Codelets
* Zapier integrations
* Users who the book was shared with
* The attachments themselves, which are separate files

JSON format
-----------

The snapshot is a JSON file and can be parsed with any JSON parser. The rest of this document describes the objects you will find, their keys and values.

Terminology
-----------

The JSON file uses its own set of terminology. Some terms you might not recognize from the app:

* “Field”: synonym for column
* “Record”: synonym for row
* “Join”: represents the fact of two sheets being linked
* “Symref”: a link chip/bubble in a cell, that links to a row in another sheet (short for “symmetric reference”)
* “Nav item”: a saved search or form, as shows up in the navigation section of the left sidebar
* “Query”: a saved search
* “Subsheet”: the table view of a linked sheet that appears on the detail view
* “Enum”: the list of values in a pick list (short for “enumerated values”)

IDs
---

Many objects have an `_id` key. This is usually of the form `$_ObjectId_1`. This is used as a reference by other objects. For instance, a join will contain the ids of the two sheets it links.

The ID key will not be explicitly mentioned below in every object where it appears.

Top-level keys
--------------

The top-level object keys are:

* `book`: All meta-data about the book: its title, the list of sheets, etc.
* `recordSets`: A list of lists of records, one per sheet
* `symrefSets`: A list of lists of symrefs, one per join. See more about joins and symrefs below.

Book
----

The `book` top-level key contains:

* `title`: book title
* `sheets`: a list of metadata about sheets, one per sheet, in the order that they appear in the book
* `joins`: a list of joins (sheet links), see below
* `localeSet`: info about the book's locale settings
  - `number`: 'us' for comma separator and decimal point (1,234.5); 'eu' for the reverse (1.234,5)
  - `date`: 'us' for mm/dd/yyyy format; 'eu' for dd/mm/yyyy format
  - `timezone`: time zone for interpreting the today() function

Sheets
------

Each object in the book's `sheets` list contains:

* `title`: sheet title
* `description`: sheet description
* `fields`: a list of field (column) objects, in they order that they appear in the sheet (see below)
* `navItems`: a list of navigation items (saved searches and forms), in the order they appear in the navigation (see below)
* `subsheets`: a list of subsheets, in the order they appear on the detail view, each with:
  - `sheet`: an object containing one key, the id of the sheet that is referenced by this subsheet
  - `field`: an object containing one key, the key of the field in that sheet that is the opposite side of this link

Fields
------

Each object in a sheet's `fields` list contains:

* `key`: a short id for this field that is unique within a sheet, usually of the form 'f1'
* `name`: display name
* `description`: column description
* `type`: this value can be:
  - 'general' or 'enum': a data column (possibly with an enum)
  - 'formula': a formula column
  - 'join': a link to another sheet
* `width`: field width in pixels
* `validation`: data input type ('generic', 'text', 'number', 'date', 'dayofyear', 'currency', 'percent', 'email', 'website', 'image', 'file', 'boolean' (checkbox), or 'picklist')
* `enum`: for a pick list, the list of values
* `expression`: for formula columns, the formula (as an AST)
* `expressionString`: for formula columns, the rendered display version of the formula
* `wrapText`: whether text should be wrapped in this column (true/false)
* `required`: whether this field requires a value (true/false)
* `hidden`: whether this field is hidden (true/false)

Note that in addition to the regular columns, each sheet contains a special field with the key `__name__`. This field can be ignored.

Records
-------

Each entry in the top-level `recordSets` key corresponds to a sheet, and contains a list of records.

Each record is an object that has a key for each field in its sheet's fields list (typically f0, f1, f2, etc.)

The value for each key is an object that contains:

* `value`: the value in this cell
* `type`: the value type ('string', 'numeric', 'currency', 'percent', 'boolean' (checkbox), 'date', 'dayofyear', 'image', 'file', or 'empty')
* `input`: the original input that was entered or imported, before being parsed and interpreted into the value

Nav items
---------

Each object in a sheet's `navItems` list contains:

* `key`: a short id for this nav item that is unique within a sheet, usually of the form 'n1'
* `name`: display name
* `description`: view description
* `type`: 'query' or 'form'
* `public`: for a form, whether it is publicly accessible (true/false)
* `uuid`: an id that is used to create a public form link, if any

If the type is 'query', this is a saved search/view and there will be a `query` key that contains an AST of the query.

If the type is 'form', this is a form and there will be a `form` key that contains the configuration for the form.

Joins
-----

A join represents the fact that two sheets are linked. Each object in a book's `joins` list contains a `left` and `right` side of the join (these are arbitrary and interchangeable). Each side contains a `sheetId` and `fieldKey` (within that sheet) identifying the link column that is one side of the join.

As an example, in a book linking Projects and Tasks, there would be a join where:

* the “right” side (for example) had the sheetId for Projects, and the fieldKey of the Tasks column within the Projects sheet
* the “left” side had the sheetId for Tasks, and the fieldKey of the Project column within the Tasks sheet

Symrefs
-------

Each entry in the top-level `symrefSets` key corresponds to a join, and contains a list of symrefs (“symmetric references”, aka links).

Each symref is an object with `left` and `right` sides. Each side has an `_id` key that refers to a row. To continue the example above, if the sheets Projects and Tasks are linked, and Project 3 has two tasks with IDs 11 and 93, you would see these two entries in the symref set:

* `{right: {_id: 3, ordinal: 0}, left: {_id: 11, ordinal: 0}}`
* `{right: {_id: 3, ordinal: 0}, left: {_id: 93, ordinal: 1}}`

Note that each side also has an `ordinal`. This indicates the order of the links within the link cell. (note: could be clearer)

Example code
------------

Here is sample code (written and tested in Node v6.9.5) to read a JSON file given on the command line and print the contents of the book. It demonstrates how to walk through sheets, records and fields; how to access values; and how to look up links:

```
const fs = require('fs');

let json = fs.readFileSync(process.argv[2], 'utf8');
let {book, recordSets, symrefSets} = JSON.parse(json);

// Build a map of records by id
let recordMap = {};
recordSets.forEach(recordSet => {
  recordSet.forEach(record => {
    recordMap[record._id] = record;
  })
})

let joinKey = (sheetId, fieldKey) => `${sheetId}:${fieldKey}`;

// Build a convenience map to look up the join info for a given sheet and field
let joinMap = {};
book.joins.forEach((join, i) => {
  joinMap[joinKey(join.left.sheetId, join.left.fieldKey)] = {index: i, side: 'left', oppositeSide: 'right'};
  joinMap[joinKey(join.right.sheetId, join.right.fieldKey)] = {index: i, side: 'right', oppositeSide: 'left'};
})

// Get all the linked records for a given record and field
let getLinkedRecords = (sheet, record, field) => {
  let {index, side, oppositeSide} = joinMap[joinKey(sheet._id, field.key)];

  return symrefSets[index]                    // look up the symref set
    .filter(s => s[side]._id === record._id)  // filter for symrefs that match the source record
    .map(s => s[oppositeSide])                // pick out the opposite side (i.e., the linked record)
    .sort((a, b) => a.ordinal - b.ordinal)    // sort by the symref ordinal to get links in correct order
    .map(r => recordMap[r._id]);              // look up the actual referenced record by id
}

console.log(`Book: ${book.title}`);

book.sheets.forEach((sheet, i) => {
  let fields = sheet.fields;
  let records = recordSets[i];

  console.log(`\n---\n\nSheet: ${sheet.title}`);
  records.forEach(record => {
    console.log(`\nRecord ${record.shortId}`);
    fields.forEach(field => {
      if (field.type === 'join') {
        let linkedRecords = getLinkedRecords(sheet, record, field);
        let names = linkedRecords.map(r => r.name);
        console.log(`- ${field.name}: ${names.join(', ')}`);
      } else if (field.type === 'formula') {
        // We aren't going to compute formulas here, but we can log the string
        console.log(`- ${field.name} = ${field.expressionString}`);
      } else {
        let value = record[field.key];
        if (value) console.log(`- ${field.name}: ${value.value}`);
      }
    })
  })
})
```
