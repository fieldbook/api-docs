# Using Fieldbook codelets to handle webhooks from GitHub

At Fieldbook, we dogfood our app for task tracking. All of our stories go into
a book that looks [something like
this](https://fieldbook.com/books/56c3aa4d1faa5a030071abf8).

[screen shot]

For a long time, we were been fully manual with updating that "Stories" sheet,
and the pull request process was pretty cumbersome. It went like this:

1. Create a PR on GitHub, paste the story's record link into the description
2. Copy the PR link
3. Mark the record as "pull request"
4. Paste the PR link into the record

Of course, steps 2-4 are easy to forget, so our stories sheet was often out of
sync with our GitHub repo. Similarly, it was easy to forget to mark a story as
"done" once the PR was merged.

This irked me. Why are we manually executing a rote process? Are we savages, or
are we engineers?

# The Old Way: A tiny Heroku app

GitHub has a fantastic set of webhook triggers, including "pull request changed
state". So I initially solved this problem by creating a small Heroku app.

That worked pretty well. I set up a little [Express](http://expressjs.com/)
server that could field the requests from GitHub, parse out the record ID and
PR status, and update the record through our API.

But that involved a lot of boilerplate. I had to:

1. Set up a new project
2. Install node modules
3. Set up a bunch of Express boilerplate (routing, middleware, etc.)
4. Add an API key to the book
5. Configure the Fieldbook client in my app
6. Write the code that actually does something
7. Deploy to Heroku
8. Set up the webhook on GitHub

Ugh. All of that, and the only part that's actually interesting is step 6.

# The New Way: Codelets!

We recently introduced [codelets](link-to-codelets-page), which let you
instantly create a one-off webserver with a single endpoint. This means we can
eliminate all of those steps besides 6 and 8.

# Creating a codelet

To add a codelet, we first click the API button to pull up the API panel.

[screen-shot]

We open up the "Codelets" tab and click "New", which gives us a new codelet
filled in with the default example.

Let's clear that out, and just start out with the simplest possible codelet.

```js
exports.endpoint = function (request, response) {
  return 'Oh, hello there!';
}
```

The `exports.endpoint` function is our request handler. If you publish now, and
copy the url into a new tab, you'll see the response "Oh, hello there!"

In other words, steps 1-5 and 7 from above have been eliminated with a couple
of clicks.

## Getting the content of the request

Codelets automatically parse the request body and give you an object on
`request.body`. Let's grab that and pull some stuff off of it:

```js
exports.endpoint = function (request, response) {
  var data = request.body;

  if (!data.action) return 'Nothing to do; not an action';
  var pr = data.pull_request;

  /* Do something with the PR data */
}
```

When we first hook up the webhook, GitHub is going to immediately send it a
request to make sure it works. They won't include an `action` parameter though,
so we'll just check that and return early if it's missing.

(Note that the only reason we return a string is that we have to return
*something*. GitHub keeps a log of its webhook requests, so returning an
informative string makes it really easy to see what happened when you look at
that log)

## Parsing out the record link

GitHub includes the PR description when they send the webhook, and we want to
find the record URL in that. We want to be able to include other stuff too, so
we'll use a regex to find it. Let's add a helper function to do that:

```js
function getRecordIdFromBody(body) {
  // Find something that looks like a link to a record, and extract the id from it
  var recordLinkPattern = /^http.*\/records\/([0-9a-fA-F]{24})\b/m;
  var match = body.match(recordLinkPattern);
  if (match) return match[1];
}
```

Basically, we're saying:

>Find a line that starts with "http", then has "/records/" after a bit,
followed by a Mongo ID, and capture that ID.

This is pretty lazy, but it works well enough.

Now that we can grab the record ID off the description, we need to check if
there even is one. If not, we'll just stop here:

```js
// Find out what record is linked from the PR
var recordId = getRecordIdFromBody(pr.body);

// If there's no record ID, just don't do anything
if (!recordId) return 'Nothing to do; no record link';
```

## Figure out the new status

Next, we need to figure out what status to set on the record. If the PR is
merged, we mark the record as "done"; otherwise we mark it as "pull request".

```js
// If the PR is merged, mark the record as done. If not, mark it as having a pull request
var status;
if (pr.merged) {
  status = 'done';
} else {
  status = 'pull request';
}

/* update the record */
```

## Actually update the record

Finally, we're going to use the Fieldbook API to update the record. When you
create a codelet, you automatically get a Fieldbook API client initialized with
your book. You don't have to set up any API keys or anything. It's just there
as the global `client`.

```js
  // Update the record with the status and PR url, then return 'OK'.
  return client.update('stories', recordId, {status, pr: pr.html_url})
    .thenResolve(`Record ${recordId} updated`);
```

This says "tell Fieldbook to update the record, setting the status and PR
fields".

The `client.update(...)` call will return a promise. Codelets are fine with
that; they'll wait for the promise to fulfill, and send the result as the
response. We use `thenResolve` to make the output nicer, so we can look in the
GitHub logs and see what happened.

(And oh yeah, that's an ES6 template string. Codelets run on Node 5.5.0, so you
can make use of many ES6 features)

## The end result

We're done! We now have a little web server with a single endpoint capable of
handling webhook requests from GitHub.

Here's our finished codelet:

```js
exports.endpoint = function (request, response) {
  var data = request.body;

  if (!data.action) return 'Nothing to do; not an action';
  var pr = data.pull_request;

  // Find out what record is linked from the PR
  var recordId = getRecordIdFromBody(pr.body);

  // If there's no record ID, just don't do anything
  if (!recordId) return 'Nothing to do; no record link';

  // If the PR is merged, mark the record as done. If not, mark it as having a pull request
  var status;
  if (pr.merged) {
    status = 'done';
  } else {
    status = 'pull request';
  }

  // Update the record with the status and PR url, then return 'OK'.
  return client.update('stories', recordId, {status, pr: pr.html_url})
    .thenResolve(`Record ${recordId} updated`);
}

function getRecordIdFromBody(body) {
  // Find something that looks like a link to a record, and extract the id from it
  var recordLinkPattern = /^http.*\/records\/([0-9a-fA-F]{24})\b/m;
  var match = body.match(recordLinkPattern);
  if (match) return match[1];
}
```

# Hooking it up to GitHub

Now all we have to do is add this webhook to our GitHub repo. Go to the
"Settings" panel of your repo:

[screenshot]

Click on "Webhooks & services", and then click "Add webhook".

[screenshot]

Paste your codelet URL into the "Payload URL" field, then click "Let me select
individual events." Make sure that only "Pull Request" is checked, and click
"Add Webhook".

[screenshot]

And we're done!

# In conclusion...

How easy is that? When I first did this as a Heroku app, the whole thing took
about an hour. When I ported it to a codelet, I rewrote it from scratch, and
the whole thing took about 10 minutes.

[The demo book](https://fieldbook.com/books/56c3aa4d1faa5a030071abf8) is
public, so you can make a copy and hook it up to your own repo and hack on it
as you like.
