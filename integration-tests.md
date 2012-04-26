Integration testing Backbone.js
===============================

Throughout my last project we had an interesting approach to testing our
JavaScript code. Instead of unit testing each and every bit of the
application, we mocked out Ajax requests and tested that the application
worked end-to-end. Of course, for complex methods we also wrote unit
tests. Our main goal, however, was not to write a lot of tests, but to
be more confident that the application worked as expected from a user's
standpoint.

We experienced three primary benefits from these tests:

* Rather than being implementation-oriented, they focus on the end result.
  This means that restrcturing and changing code rarely makes tests break, 
  as long as it doesn't break the application's end-to-end behaviour.
* They were very easy to write using Backbone's abstractions. They
  also more natural to write as they focus on the end result.
* They were fast, so we could run them all the freakin' time.

Let's look at an example:

```javascript
it("should list all persons in response", function() {

  // fetch an ajax response
  var response = readFixtures("responses/persons.json");
  var headers = {};

  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show header and other static stuff

  // mock out all requests (this is our core test abstraction)
  fakeResponse(response, headers, function() {
    view.collection.fetch(); // trigger AJAX request
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

The core test abstraction in the example is `fakeResponse`. This
method creates mocks responses for AJAX requests using Sinon.js. This is a
simplified implementation of the function:

```javascript
function fakeResponse(response, options, callback) {
  var statusCode, headers, server, resp;

  statusCode = options.statusCode || 200;
  headers = options.headers || { "Content-Type": "application/json" }

  server = sinon.fakeServer.create();
  server.respondWith([statusCode, headers, response]);

  callback();

  server.respond();
  server.restore();
}
```

With this relatively simple abstraction we can do a lot of powerful
stuff in our tests. Let's look at a new example where we interact with
the view and do several AJAX requests:

```javascript
it("should show error message when pagination fails", function() {

  // AJAX responses
  var response = readFixtures("responses/persons.json");
  var errorResponse = readFixtures("responses/errors.json");

  // create our initial view
  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show header and other static stuff

  // mock out all requests
  fakeResponse(response, headers, function() {
    view.collection.fetch(); // trigger AJAX request
  });

  // we ensure that errors are not present
  expect(view.$('.errors')).not.toExist();

  // we create an error response ...
  fakeResponse(errorResponse, { statusCode: 503 }, function() {

    // ... and we trigger the error by clicking the next button
    // (in our view we listen for the click event and start the pagination)
    view.$('a.more').click();

  });

  // ensure that errors are present
  expect(view.$('.errors')).toExist();
});
```

We often initialize views the same way for several tests, as with `PersonsView` 
in the examples above. It is usually a good idea to create abstractions for these at some point.
For example we can create a `getPersonsViewFromResponse` as follows:

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

We have written more than 200 of these tests, and, as they are not
dependent on putting things in the DOM, they are blazingly fast. Ours
run in about 1.3 seconds. Additionally, they work wonders for our
confidence and our code.
