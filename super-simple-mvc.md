Creating a simplistic MVC in JavaScript
=======================================

I've been using Backbone.js for several projects that last year, but
recently I started thinking about how easy it is to create some of the
core abstractions in Backbone.js.

For me these are the most important aspects of Backbone.js:

* A simple way of creating new components which inherit functionality
  from a parent.
* Creating [responsible views](views.md), so each view *owns* an HTML
  element.
* Views which wrap DOM-events, so we can specify all our DOM bindings
  for a view in one place.
* Models that abstract Ajax, so we don't have to perform `$.ajax`
  ourselves.
* Built-in events handling, so we can be notified e.g. when Ajax
  responses are received. This is at the core of decoupling JavaScript
  code.

Let's go through these in order, and see how we can create a super
simple, albeit simplistic, MVC while using some great and widely used
libraries. (The essence of this is not its brevity, but that it is
suprisingly easy to do.)

Creating new components
-----------------------

Backbone.js uses
[`extend`](http://documentcloud.github.com/backbone/#Model-extend) to
create new components. Let's look at how we can create a `UserView`
which inherit from `View` and has a `showUser` method using Backbone.js:

```javascript
var UserView = Backbone.View.extend({
  showUser: function() {
    console.log("showing user");
  }
});

var userView = new UserView();
userView.showUser(); // showing user
```

To create something similar, we can use
[`$.extend`](http://api.jquery.com/jQuery.extend/):

```javascript
var View = function() {};

View.extend = function (properties) {
  var child = $.extend.call({}, this.prototype, properties);
  return child.constructor;
};
```

Now we can create a `UserView` using this abstraction:

```javascript
var UserView = View.extend({
  showUser: function() {
    console.log("showing user");
  }
});

var userView = new UserView();
userView.showUser(); // showing user
```

In Backbone.js, however, `initialize` is called when a component is
instantiated. So let's add this little bit of functionality:

```javascript
var View = function() {
  if (this.initialize) {
    // ensure that `initialize` is called with the correct arguments
    this.initialize.apply(this, arguments);
  }
};

View.extend = function (properties) {
  var child = $.extend.call({}, this.prototype, properties);
  return child.constructor;
};
```

Now we can create a view which also have an `initialize` method which
receives all arguments send in when instantiating the view:

```javascript
var UserView = View.extend({
  initialize: function(options) {
    console.log(options);
  }
});

var userView = new UserView({ text: "test" });
// this logs { text: "test" } as expected
```

Now that we have a way of creating views, we can create models using the
exact same logic:

```javascript
var Model = function() {
  if (this.initialize) {
    // ensure that `initialize` is called with the correct arguments
    this.initialize.apply(this, arguments);
  }
};

Model.extend = function (properties) {
  var child = $.extend.call({}, this.prototype, properties);
  return child.constructor;
};
```

Responsible views
-----------------


Wrapping DOM events
-------------------


Abstracting Ajax
----------------


Built-in events
---------------


