Fieldbook API Client Examples
=============================

Here are examples of how to access the Fieldbook API using different clients.

In all the example below, substitute:

* your API key for `$KEY`
* its secret for `$SECRET`
* a sheet URL for the example one used, `https://api.fieldbook.com/v1/56789abc0000000000000001/tasks`

curl
----

List records:

```bash
$ curl -H "Accept: application/json" -u $KEY:$SECRET https://api.fieldbook.com/v1/56789abc0000000000000001/tasks
```

Create a record:

```bash
$ curl -H "Accept: application/json" -H "Content-Type: application/json" -u $KEY:$SECRET \
    https://api.fieldbook.com/v1/56789abc0000000000000001/tasks \
    -d '{"name":"New task","owner":"Alice","priority":1}'
```

jQuery
------

```js
// List records:

$.ajax({
  url: 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
  headers: {
    'Accept': 'application/json',
    'Authorization': 'Basic ' + btoa('$KEY:$SECRET')
  },
  success: function (data) {
    console.log(data.length + ' items');
  },
  error: function (error) {
    console.log('error', error);
  }
});

// Create a record:

var record = {
  name: "New task",
  owner: "Alice",
  priority: 1
};

$.ajax({
  method: 'POST',
  url: 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
  headers: {
    'Accept': 'application/json',
    'Content-Type': 'application/json',
    'Authorization': 'Basic ' + btoa('$KEY:$SECRET')
  },
  data: JSON.stringify(record),
  success: function (data) {
    console.log('created record', data);
  },
  error: function (error) {
    console.log('error', error);
  }
});
```

Note: This uses `btoa` to do base-64 encoding for the authorization header. This will work in [browsers other than IE 8 and 9](http://caniuse.com/#search=btoa); for older versions of IE you can [use a polyfill](https://github.com/davidchambers/Base64.js).

WARNING: If you do this in a web page, anyone who loads the web page will have your API key and secret and will be able to access the entire book.

Node
----

Using [the npm `request` module](https://github.com/request/request):

```js
var request = require('request');

// List records

var options = {
  url: 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
  json: true,
  auth: {
    username: '$KEY',
    password: '$SECRET'
  }
};

request(options, function (error, response, body) {
  if (error) {
    console.log('error making request', error);
  } else if (response.statusCode >= 400) {
    console.log('HTTP error response', response.statusCode, body.message);
  } else {
    console.log(body.length + ' items');
  }
});

// Create a record

var record = {
  name: "New task",
  owner: "Alice",
  priority: 1
};

options = {
  url: 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
  json: true,
  body: record,
  auth: {
    username: '$KEY',
    password: '$SECRET'
  }
};

request.post(options, function (error, response, body) {
  if (error) {
    console.log('error making request', error);
  } else if (response.statusCode >= 400) {
    console.log('HTTP error response', response.statusCode, body.message);
  } else {
    console.log('created record', body);
  }
});
```

Ruby
----

Using Net::HTTP (as per [this cheat sheet](http://www.rubyinside.com/nethttp-cheat-sheet-2940.html)):

```rb
require "net/http"
require "uri"
require "json"

uri = URI.parse("https://api.fieldbook.com/v1/56789abc0000000000000001/tasks")

http = Net::HTTP.new(uri.host, uri.port)
request = Net::HTTP::Get.new(uri.request_uri)
request.basic_auth("$KEY", "$SECRET")
http.use_ssl = true
response = http.request(request)

items = JSON.parse(response.body)
puts "#{items.length} items"
```

Python
------

Using [the Requests library](http://docs.python-requests.org/en/latest/):

```python
import requests
request = requests.get('https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
    auth=("$KEY", "$SECRET"))
print len(request.json()), "items"
```

PHP
---

Using [PHP cURL](http://php.net/manual/en/ref.curl.php):

```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_USERPWD, $KEY . ':' . $SECRET);
curl_setopt($ch, CURLOPT_URL, 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks');
$result = curl_exec($ch);
curl_close($ch);

$obj = json_decode($result);
echo count($obj) . ' items';
```

Google Apps Script
------------------

Using [UrlFetchApp Class](https://developers.google.com/apps-script/reference/url-fetch/url-fetch-app)

```js
UrlFetchApp.fetch('https://api.fieldbook.com/v1/56789abc0000000000000001/tasks', {
  method: 'post',
  headers: {
    "Accept": "application/json",
    "Authorization": "Basic " + Utilities.base64Encode(key + ":" + secret)
  },
  payload: JSON.stringify({
    task_name_or_identifier: "task1"
  })
});
```
