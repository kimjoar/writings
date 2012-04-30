Integration testing Backbone.js
===============================

This blog post is published at
[open.bekk.no](http://open.bekk.no/integration-testing-backbone-js/),
but Wordpress died, so have redirected it here temporarily.

Throughout my last project we have had an interesting approach to
testing our Backbone.js code. Instead of unit testing each and every bit
of the application, we mock out Ajax requests and test that the
application works end-to-end. Of course, for complex methods we also
write unit tests. Our main goal, however, is not to write a lot of
tests, but to be more confident that the application work as expected
from a user's point of view.

We experience three primary benefits from these tests:

* Rather than being implementation-oriented, they focus on the end
  result. This means that restructuring and changing code rarely breaks
  tests, as long as we don't break the application's end-to-end
  behaviour.
* They are very easy to write using Backbone's abstractions. They are
  also more natural to write as they focus on the end result.
* They are very fast, so we could run them all the freakin' time.

Let's look at an example:

```javascript
it("should list all persons in response", function() {

  // fetch an Ajax response
  var response = readFixtures("responses/persons.json");
  var options = {}; // no additional options for the Ajax request

  // create our view, sending in an empty collection
  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show header and other static stuff

  // mock out all requests (this is our core test abstraction),
  // i.e. what this function does is responding to all Ajax requests in
  // the callback with the same response and with the same options.
  // Thus, when `fakeResponse` is finished, all Ajax requests will have
  // responded, and as we listen for finished Ajax requests in our view,
  // we will then have properly set up our PersonsView.
  fakeResponse(response, options, function() {
    view.collection.fetch(); // trigger Ajax request
  });

  // ensure that all persons are present
  expect(view.$('.persons li').length).toEqual(20);

});
```

So, basically, we test the end result given a specific response.

We used the following libraries for writing tests:

* [Jasmine](http://pivotal.github.com/jasmine/) as our test framework.
* [Jasmine-jQuery](https://github.com/velesin/jasmine-jquery) for
  fixtures.
* [Sinon.js](http://sinonjs.org/) for test spies, stubs and mocks.

The core test abstraction in the example is `fakeResponse`. This method
creates mock responses for Ajax requests using Sinon.js. This is a
simplified implementation of the function:

```javascript
function fakeResponse(response, options, callback) {
  var statusCode, headers, server, resp;

  // some default values, so we don't have to set status code and
  // content type all the time.
  statusCode = options.statusCode || 200;
  headers = options.headers || { "Content-Type": "application/json" }

  // we create what Sinon.js calls a fake server. This is basically just
  // a name for mocking out all XMLHttpRequests. (There are no actual
  // servers involved.)
  server = sinon.fakeServer.create();
  
  // we tell Sinon.js what we want to respond with
  server.respondWith([statusCode, headers, response]);

  callback();

  // this actually makes Sinon.js respond to the Ajax request. As we can
  // choose when to respond to a request, it is for example possible to
  // test that spinners start and stop, that we handle timeouts
  // properly, and so on.
  server.respond();

  server.restore();
}
```

With this relatively simple abstraction we can do a lot of powerful
stuff in our tests. Let's look at a new example where we interact with
the view and perform several Ajax requests:

```javascript
it("should show error message when pagination fails", function() {

  // ajax responses
  var response = readFixtures("responses/persons.json");
  var errorResponse = readFixtures("responses/errors.json");

  // create our initial view
  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show header and other static stuff

  // mock out all requests
  fakeResponse(response, headers, function() {
    view.collection.fetch(); // trigger Ajax request
  });

  // now we have set up our initial state, and can go on to doing things
  // in the view.

  // we ensure that errors are not present
  expect(view.$('.errors')).not.toExist();

  // we set up an error response ...
  fakeResponse(errorResponse, { statusCode: 503 }, function() {

    // ... and we trigger the error by clicking the next button
    // (in our view we listen for the click event and start the
    // pagination process)
    view.$('a.more').click();

  });

  // ensure that errors are present
  expect(view.$('.errors')).toExist();
});
```

We often initialize views the same way for several tests, as with
`PersonsView` in the examples above. It is usually a good idea to create
abstractions for these at some point. For example we can create a
`getPersonsViewFromResponse` as follows:

```javascript
function getPersonsViewFromResponse(response, options) {
  var view = new PersonsView({ collection: new Persons() });
  view.render();

  fakeResponse(response, options, function() {
    view.collection.fetch();
  });

  return view;
}
```

Thus, we can call `getPersonsViewFromResponse` to initialize our view,
which makes our tests even easier to write.

---

We have written more than 200 of these tests, and, as they don't change
the DOM, they are blazingly fast. Ours run in about 1.3 seconds.
Additionally, they work wonders for our confidence and our code.
