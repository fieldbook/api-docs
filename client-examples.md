Fieldbook API Client Examples
=============================

Here are examples of how to access the Fieldbook API using different clients.

In all the example below, substitute:

* your API key for `$KEY`
* its secret for `$SECRET`
* a sheet URL for the example one used, `https://api.fieldbook.com/v1/56789abc0000000000000001/tasks`

curl
----

```
$ curl -H "Accept: application/json" -u $KEY:$SECRET https://api.fieldbook.com/v1/56789abc0000000000000001/tasks
```

jQuery
------

```
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
```

Note: This uses `btoa` to do base-64 encoding for the authorization header. This will work in [browsers other than IE 8 and 9](http://caniuse.com/#search=btoa); for older versions of IE you can [use a polyfill](https://github.com/davidchambers/Base64.js).

WARNING: If you do this in a web page, anyone who loads the web page will have your API key and secret and will be able to access the entire book.

Node
----

Using [the npm `request` module](https://github.com/request/request):

```
var request = require('request');

var options = {
  url: 'https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
  headers: {
    'Accept': 'application/json'
  },
  auth: {
    username: '$KEY',
    password: '$SECRET'
  }
};

request(options, function (error, response, body) {
  if (error) {
    console.log('error', error);
  } else {
    console.log(body.length + ' items');
  }
});
```

Ruby
----

Using Net::HTTP (as per [this cheat sheet](http://www.rubyinside.com/nethttp-cheat-sheet-2940.html)):

```
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

```
import requests
request = requests.get('https://api.fieldbook.com/v1/56789abc0000000000000001/tasks',
    auth=("$KEY", "$SECRET"))
print len(request.json()), "items"
```

Google Apps Script
------------------

Using [UrlFetchApp Class](https://developers.google.com/apps-script/reference/url-fetch/url-fetch-app)

```
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
