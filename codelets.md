Fieldbook Codelets
==================

What are Codelets?
------------------

Fieldbook lets you extend the API of any book with little snippets of code –
*codelets* – that define custom endpoints. With Codelets, you can do in a few
minutes and a couple dozen lines of code what would previously have required an
entire API server hosted separately.

### Examples

See these tutorial examples for how to use Codelets to:

* [Create a custom Slackbot for your Fieldbook that responds to slash commands](codelets/slackbot.md)
* [Keep a book in sync with GitHub pull requests by handling webhook callbacks](codelets/github.md)
* [Add a custom form-submission endpoint to your Fieldbook](codelets/form.md)

Getting started
---------------

To access codelets on your book, open the API modal:

![api-button](images/api-button.png)

Then open the Codelets tab:

![codelets-tab](images/codelets-tab.png)

From this pane, you can edit the code for your custom endpoint. Using curl or a
browser, you can fire a request against the given URL to run the codelet.

The code should define an `exports.endpoint` function that takes a request, a
response, and optionally a done callback:

```
exports.endpoint = function (request, response, done) {
  // codelet body here
}
```

Codelet requests
----------------

Your codelet URL will respond to any of these HTTP methods: GET, POST, PUT,
PATCH or DELETE. The request object passed to the function will have:

* `request.method`
* `request.headers`
* `request.body`
* `request.query`
* `request.params` (a combination of body and query, with query taking precedence)

Requests will accept JSON body parameters if the Content-Type of the request is
`application/json`, and similarly will accept form parameters of the
Content-Type is `application/x-www-form-urlencoded`.

Its easy to inpect what is available on the request object, just return it from
a codelet like so:

```javascript
exports.endpoint = function (request) {
  return request;
}
```

And then request it with some parameters:

```
$ curl "$CODELET_URL?foo=bar" -d '{"zip":"hello"}' -H 'Content-Type: application/json'
```

Example response:

```
{
  "headers": {
    "host": "fieldbook.com",
    "user-agent": "curl/7.43.0",
    "accept": "*/*",
    "content-type": "application/json",
    "x-request-id": "92cf3346-f407-4d2e-a1be-35f83af9d532",
    "x-forwarded-for": "XXX.XXX.XXX.XXX",
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "x-request-start": "1455144616294",
    "content-length": "16"
  },
  "body": {
    "zip": "hello"
  },
  "query": {
    "foo": "bar"
  },
  "params": {
    "zip": "hello",
    "foo": "bar"
  },
  "method": "POST"
}
```

Codelet responses
-----------------

For convenience, there are multiple ways to generate a response:

### Return an object

A JSON object or string will be directly translated into a response body:

```javascript
exports.endpoint = function (req) {
  return {hello: 1};
}
```

### Return a promise

You can also return a promise for a response object:

```javascript
var Q = require('q');
exports.endpoint = function (req) {
  return Q.delay(10).then(function () {
    return {hello: 1};
  })
}
```

### Invoke the done callback

Invoke the callback as `done(error, result)`. Pass null for error if there is
none:

```javascript
exports.endpoint = function (req, res, done) {
  setTimeout(function () {
    done(null, {hello: 1})
  }, 500);
}
```

### Use the response object

Directly invoke the response object for more control over response headers and
such:

```javascript
exports.endpoint = function (req, res) {
  res.type('text/plain');
  res.send('Hello World');
}
```

Methods on the response object:

#### res.send(STRING or OBJECT)

Sends a response. If no Content-Type header is set, the type will be determined
by the argument. A string will be sent as `text/plain`; an object will be
stringified to JSON and sent as `application/json`.

If a Content-Type has already been set, the argument will be handled
accordingly. For instance, if `text/plain` has been set, and an object is passed
to send(), then `object.toString()` will be invoked to create a text response.

When called with no arguments, will send the prepared body (by the write()
method).

Calling this method ends the request and further calls to this or any other
response method will result in an error.

#### res.write(STRING)

Call to incrementally append data to the body of the response.

If you ever call write or send on the response object, you may still return a
promise from your endpoint, but the result of the promise will be ignored. If
you call write and also use the done callback to return a result, an error will
be thrown.

#### res.status(CODE)

Sets the HTTP status code of the response.

#### res.setHeader(NAME, VALUE)

Set a header value (however, you may not set cookies; see below).

#### res.type(CONTENT_TYPE)

Convenience method for setting the Content-Type header.

#### res.location(URL)

Convenience method for setting the Location header.

#### res.redirect(CODE, URL) or redirect(URL)

Shorthand for setting the status code and URL for a redirect. CODE defaults to
302 if not passed.

Accessing your Fieldbook data
-----------------------------

The pre-initialized `client` object provides access to the book the codelet is
on. It's an instance of the
[fieldbook-client](https://github.com/fieldbook/fieldbook-client) Node module.
Here is an example using this client to return all names from the “People” sheet
of a book:

```javascript
exports.endpoint = function (request) {
  return client.list('people').then(function (records) {
    return records.map(function (record) {
      return record.name;
    }
  };
}
```

ES6
---

Codelets are run on Node 5.5.0 with the `--harmony` flag. This means you can use
a number of great ES6 features, like fat arrow syntax and generators. Here the
same example rewritten to use `yield` and fat arrows:

```javascript
var Q = require('q');
exports.endpoint = Q.async(function * (request) {
  var people = yield client.list('people');
  return people.map(p => p.name);
})
```

Restrictions
------------

* A codelet should take no more than a few seconds to run. Longer codelets may
be terminated in the middle of their execution. Timeout happens around 50
seconds.

* You may not set cookies, and trying to will result in an error when your code
is run.

Available modules
-----------------

You can `require()` Node modules in your codelets. The following modules are
currently supported:

* amazon-product-api (0.3.8)
* async (1.5.2)
* aws-sdk (2.2.33)
* bcrypt (0.8.5)
* bitly (4.1.1)
* bluebird (3.2.1)
* body-parser (1.15.0)
* bunyan (1.6.0)
* chalk (1.1.1)
* cheerio (0.20.0)
* clone (1.0.2)
* co (4.6.0)
* colors (1.1.2)
* connect (3.4.1)
* cors (2.7.1)
* cradle (0.7.1)
* dropbox (0.10.3)
* ebay-api (1.12.0)
* elasticsearch (10.1.3)
* fb (1.0.2)
* fieldbook-client (1.0.4)
* firebase (2.4.0)
* flickrapi (0.3.36)
* formidable (1.0.17)
* github (0.2.4)
* glob (7.0.0)
* googleapis (2.1.7)
* handlebars (4.0.5)
* heroku (0.1.3)
* hoek (3.0.4)
* instagram-node (0.5.8)
* intrusive (1.0.1)
* irc (0.4.1)
* joi (8.0.1)
* jsdom (8.0.2)
* jshint (2.9.1)
* json-socket (0.1.2)
* jsonwebtoken (5.5.4)
* koa (1.1.2)
* lazy (1.0.11)
* lodash (4.1.0)
* lodash.assign (4.0.2)
* lru-cache (4.0.0)
* mandrill-api (1.0.45)
* marked (0.3.5)
* merge (1.2.0)
* mime (1.3.4)
* moment (2.11.1)
* mongodb (2.1.7)
* mongoose (4.4.3)
* natural (0.2.1)
* node-fieldbook (1.0.7)
* node-foursquare (0.3.0)
* node-linkedin (0.5.3)
* node-uuid (1.4.7)
* node-wikipedia (0.0.2)
* node-xmpp-client (3.0.0)
* nodemailer (2.1.0)
* numeric (1.2.6)
* once (1.3.3)
* papaparse (4.1.2)
* passport (0.3.2)
* pg (4.4.5)
* pinterest-api (1.1.4)
* q (1.4.1)
* ramda (0.19.1)
* redis (2.4.2)
* redux (3.3.1)
* request (2.69.0)
* requestify (0.1.17)
* restify (4.0.4)
* science (1.9.3)
* sequelize (3.19.2)
* shortid (2.2.4)
* slack (5.2.2)
* split (1.0.0)
* spotify-web-api-node (2.2.0)
* stream-buffers (3.0.0)
* stripe (4.3.0)
* superagent (1.7.2)
* syntax-error (1.1.5)
* through (2.3.8)
* through2 (2.0.1)
* traverse (0.6.6)
* tumblr (0.4.1)
* twilio (2.9.0)
* twitter (1.2.5)
* underscore (1.8.3)
* underscore.string (3.2.3)
* us-census-api (0.0.5)
* validator (4.8.0)
* vimeo (1.1.4)
* winston (2.1.1)
* wordpress (1.1.2)
* wundergroundnode (0.9.0)
* xlsx (0.8.0)
* xml2js (0.4.16)
* yahoo-finance (0.2.12)
* yargs (4.1.0)
* yelp (1.0.1)

Is your favorite module missing? Let us know using the “Message us” button in
the app.
