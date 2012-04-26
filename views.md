A view's responsibility — lessons on JavaScript and the DOM
===========================================================

Throughout most JavaScript code I've seen, `$` is littered all over the
place. You need all list items from the members list? Just do
`$('.members li')`. And then you need to add something to the sidebar,
just `$('.sidebar').append("cool!")`. So — what's the problem?

First of all, how do you test these views? Or, this is JavaScript, so
you probably
[don't](https://twitter.com/#!/jasminebdd/status/182322290464276480).
The basic problem is that you need the DOM present to enable searching
for selectors this way, and to have the DOM present you (often) need to
set up the entire application (which, obviously slows down tests).

Secondly, who is allowed to change what? Can all functions change
whatever part of the DOM they want? This wreaks havoc for your tests and
creates uncertainty as to where something occurs.

The solution: **A view is responsible for one HTML element and
everything inside it.**

And, of course, a view may contain several subviews which again are
responsible for themselves.

What's a view?
--------------

Basically, it's a component which handles user interface. In my
JavaScript code it is responsible for handling a subset of the DOM, e.g.
adding text, removing nodes, listening to DOM events and so on. But it
is never ever allowed access to something which is beyond its subset of
the DOM.

Let's look at an example of a view:

```javascript
var UserView = function(el, user) {
  this.el = el;
  this.user = user;
}

UserView.prototype.showImage = function() {
  this.el.append('<img src="' + user.image + '" />');
}

// let's just create a super simple user object
var user = {
  image: 'http://example.com/image.png'
};

// we initialize the view with its el and a user
var view = new UserView($('.user'), user);

// ... and now we can do stuff which changes the DOM
view.showImage();
```

No frameworks or libraries are needed, just being strict with how you
write your code. With these small changes, this `UserView` is easily
testable and can easily be moved around on the page, or even be removed
without being afraid of how it impacts the rest of the application.

Just as an example, to test this bit of code, we can initialize it with
`$('<div></div>)` instead of `$('.user')`. Thus, we can call `showImage`
and then check that the image is present in `view.el`, i.e. no DOM
needed as every thing lives in the jQuery object.

But I need to change something 'over there'
-------------------------------------------

So let's say you have created several of these views, but now one view
needs to change something in another view — but how? Remember, a view is
not allowed do anything outside its HTML element.

The solution is events.

Events are basically just a way to say: "Hi, I want to know when some
action occurs" and "Hi, you know what? The action you're waiting for
just occured!" We are used to this idea from jQuery DOM events such as
`click` and `submit`. Now we are just moving it into the rest of our
code.

Let's look at some examples using
[EventEmitter](https://github.com/Wolfy87/EventEmitter):

```javascript
var events = new EventEmitter();

var UserView = function(el, user) {
  this.el = el;
  this.user = user;

  // Let's listen for someone emitting the event 'user:showImage', which
  // we listen for and then show the user's image.
  // The first parameter is the event name, the second is the function
  // to call when the event is triggered, and the third is the context
  // the function is called with.
  events.addListener('user:showImage', this.showImage, this);

  // We can also emit events. Let's tell listeners that we have created
  // a user view.
  events.emit('user:created');
}
```

Thus, whenever you are interested in something outside a view, you can
listen for events, and you can also let other views know when something
has occured. This creates a highly decoupled application which is easy
to test and easy to extend.
