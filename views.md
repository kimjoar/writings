A view's responsibility — lessons on JavaScript and the DOM
===========================================================

Throughout most JavaScript code I've seen, `$` is littered all over the
place. You need all list items from the members list? Just do
`$('.members li')`. And then you need to add something to the sidebar,
just `$('.sidebar').append("this is so easy")`. So — what's the problem?

First of all, how do you test these views? Or, this is JavaScript, so
you probably
[don't](https://twitter.com/#!/jasminebdd/status/182322290464276480).
Cheekiness aside, the basic problem is that you need the DOM present to
enable searching for selectors, and to have the DOM present you (often)
need to set up the entire application — which, obviously, slows down
tests considerably.

Secondly, who is allowed to change what? Can all functions change
whatever part of the DOM they want? This wreaks havoc for your tests and
creates uncertainty as to where something occurs. And once again, you
most likely need to set up the entire application to test the view.

My solution: **A view is responsible for one HTML element and everything
inside it.**

And, of course, a view may contain several sub-views which again are
responsible for themselves.

What's a view?
--------------

Basically, a view is a component which handles some part of a user
interface. My views have five primary responsibilities (Those familiar
with MVC and MVP will probably call these views *Controllers* or
*Presenters*, but the nomenclature is not important — splitting
responsibilities between different components is):

* Rendering the view, i.e. making changes to the DOM.
* Listening for DOM events, such as `click` and `submit`.
* Listening for events from the rest of my application, plus triggering
  events when the view is in certain states.
* Creating sub-views if they are needed.
* Updating models based on changes in the view (Don't you dare make
  `$.ajax` calls directly in views!)

A view is, however, never ever allowed to access something which is
outside its subset of the DOM.

Let's look at an example of a view:

```javascript
// a view constructor which accepts:
// - el, which is a jQuery object of the HTML element the view owns
// - user, which is a user model with key-value pairs
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

// initialize the view with the jQuery object the view owns and a user
var view = new UserView($('.user'), user);

// ... and now we can do stuff which changes the DOM
view.showImage();
```

This `view` is never ever to go outside of `.user`. Ever.

No frameworks or libraries are needed, just being strict with how you
write your code. With these small changes we have contained a subset of
the user interface to a specific view, and this `UserView` is easily
testable and can easily be moved around on the page. It can even be
removed without being afraid of how it impacts the rest of the
application.

Easily testable
---------------

Just as an example, to test this bit of code we can initialize it with
`$('<div></div>')` instead of `$('.user')`. This just means that we let
an empty `div` live in the jQuery object instead of the `.user` subset
of the DOM.

Using this trick, we can test the user view by calling `showImage` and
then check that the image is present. As everything lives in the jQuery
object we don't need to set up the DOM. Let's look at a code example
using Jasmine:

```javascript
describe('user view', function() {
  it('should be able to show image', function() {
    var user = {
      image: "user.png"
    };

    var view = new UserView($('<div></div>'), user);
    view.showImage();

    // remember that `view.el` is a jQuery object. So now we can call
    // `find` on it directly instead of looking for `img` in the entire
    // DOM.
    var image = view.el.find('img');
    
    expect(image.attr("src")).toEqual("user.png");
  });
});
```

With a small helper function the code becomes even easier to work with:

```javascript
UserView.prototype.$ = function(select) {
  return this.el.find(selector);
}
```

Now you can write `view.$('img')` instead of `view.el.find('img')`.

But I need to change something 'over there'
-------------------------------------------

So let's say you have created several of these views, but now one view
needs to change something in another view — but how? Remember, a view is
not allowed do anything outside its HTML element.

The solution is events.

Events are basically just a way to say: "Hi, I want to know when some
action occurs" and "Hi, you know what? The action you're waiting for
just occurred!" We are used to this idea from jQuery DOM events such as
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

Now, whenever you are interested in something outside a view, you can
listen for events, and you can also let other views know when something
has occurred. This creates a highly decoupled application which is easy
to test and easy to extend.

But, why?
---------

To sum up, there are three primary benefits of writing your JavaScript
views like this:

* You always know who is responsible for some subset of the DOM, and
  changing a view will *never* impact the DOM outside of its "walls".
* You can have localized DOM lookup. Instead of looking for `.user img`
  you can look for `img` on the user view. Based on the above example
  we can find the image by writing `userView.DOM('img')`.
* It's very simple to test. And your tests will be blazingly fast as
  they do not depend on the DOM or on the entire app being set up.
