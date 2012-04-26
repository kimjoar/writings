Integration testing Backbone.js
===============================

Throuhout my last project we had a special approach to testing our
JavaScript code. Instead of unit testing each and every bit of our
application, we mocked out Ajax requests and tested that the application
worked end-to-end. Of course, for complex methods we also wrote unit
tests. Our main goal, however, was not to write a lot of tests, but to
be more certain that the application worked as expected from a user's
standpoint.

We got three primary benefits from these tests:

* They were not implementation-oriented, but focused on the end result.
  Thus, when changing code we seldom got failing tests when the
  application worked as before end-to-end (I have fixing tests when I
  actually haven't broken anything).
* They were very easy to write using Backbone's abstractions. They
  also felt easier to write as they entailed a focus on the end result.
* They were fast, so we could run them all the freakin' time.

Let's look at an example:

```javascript
it("should list all persons in response", function() {

  // fetch an ajax response
  var response = readFixtures("responses/persons.json");
  var headers = {};

  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show headline and other static stuff

  // mock out all requests (this is our core test abstraction)
  fakeResponse(response, headers, function() {
    view.collection.fetch(); // trigger AJAX request
  });

  // ensure that all persons are present
  expect(view.$('.persons li').length).toEqual(20);

});
```

So, basically, we test the end result given a specific response.

To write these tests we used the libraries:

* [Jasmine](http://pivotal.github.com/jasmine/) as our test framework.
* [Jasmine-jQuery](https://github.com/velesin/jasmine-jquery) for
  fixtures.
* [Sinon.js](http://sinonjs.org/) for test spies, stubs and mocks.

The core test abstraction in the example is `fakeResponse`. What this
method does, is mocking out all AJAX requests using Sinon.js. This is a
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

With this, relatively simple, abstraction, we can do a lot of powerful
stuff in our tests. Let's look at a new example were we interact with
the view and have several AJAX requests:

```javascript
it("should show error message when pagination fails", function() {

  // AJAX responses
  var response = readFixtures("responses/persons.json");
  var errorResponse = readFixtures("responses/errors.json");

  // create our initial view
  var view = new PersonsView({ collection: new Persons() });
  view.render(); // show headline and other static stuff

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

For those with experience with Ruby, the `fakeResponse` function almost
feels like a block.

We have written more than 200 of these tests, and, as they are not
dependent on putting things in the DOM, they are blazingly fast. Ours
run in about 1.3 seconds.
