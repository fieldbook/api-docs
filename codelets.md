Fieldbook Codelets
==================

Getting Started
---------------

Fieldbook codelets are the simpliest way to host Node code from a public URL.  Codelets provide easy access to your Fieldbook data through a preinitialized [fieldbook-client](https://github.com/fieldbook/fieldbook-client) object.

To access codelets on your book, open the API modal.

![api-button](images/api-button.png)

Then open the codelets tab

![codelets-tab](images/codelets-tab.png)

From this pane, you can edit the code for your custom endpoint.  Using curl or a browser, you can fire a request against the given url to run the codelet.

A codelet should take no more than a few seconds to run.  Longer codelets may be terminated in the middle of their execution.  Timeout happens around 50 seconds.

The code should expose an exports.endpoint function that takes a request and a response.  That function may return a promise (Q and Bluebird are provided), may call response.send
r may take a third parameter which is a done callback.  It may also just return JSON objects or strings to return to the caller.

Response Examples
-----------------

