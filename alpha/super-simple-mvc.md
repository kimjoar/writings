Creating a simplistic MVC in JavaScript
=======================================

THIS IS STILL UNDER DEVELOPMENT.

I've been using Backbone.js for several projects that last year, but
recently I started thinking about how easy it is to create some of the
core abstractions in Backbone.js.

For me these are the most important aspects of Backbone.js:

* A simple way of creating new components which inherit functionality
  from a parent.
* Creating [views that own an HTML-element](views.md).
* Views which wrap DOM-events, so we can specify all our DOM bindings
  for a view in one place.
* Models that abstract Ajax, so we don't have to perform `$.ajax`
  ourselves.
* Built-in events handling, so we can be notified e.g. when Ajax
  responses are received. This is at the core of decoupling JavaScript
  code.

Let's go through these in order, and see how we can create a super
simple, albeit simplistic, MV* library while using some great and widely
used libraries. (The essence of this is not its brevity, but that it is
surprisingly easy to do, and that we can learn a lot about Backbone.js
and similar libraries in the process.)

Creating new components
-----------------------

Backbone.js uses
[`extend`](http://documentcloud.github.com/backbone/#Model-extend) to
create new components which inherit functionality from a parent. Let's
look at how we can, using Backbone.js, create a `UserView` which inherit
from the core `View` component and which define a `showUser` method:

```javascript
var UserView = Backbone.View.extend({

  // all properties that are specified here will end up as instance
  // methods on the UserView.

  showUser: function() {
    console.log("showing user");
  }
});

var userView = new UserView();
userView.showUser(); // showing user
```

To create something similar we first need a view constructor:

```javascript
var View = function() {};
```

We then need to add the `extend` method, which actually ends up at less
than 10 lines of code, and which also enable us to create subclasses of
subclasses:

```javascript
View.extend = function(properties) {
    var parent = this;

    // Create child constructor
    var child = function() {
        // … which only job is to call the parent constructor with all
        // the arguments
        parent.apply(this, arguments);
    };

    // Set the prototype chain so the child will inherit all properties
    // from the parent
    child.prototype = Object.create(parent.prototype);

    // Add the child's prototype properties, i.e. its instance properties
    $.extend(child.prototype, properties);

    // The child must be able to create new subclasses
    child.extend = parent.extend;

    return child;
};
```

We can now create a `UserView` using this abstraction:

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
  // pass through all the arguments to `initialize`
  this.initialize.apply(this, arguments);
};

// Empty initialize which should be overridden in subclasses if it
// is needed.
View.prototype.initialize = function() {};
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
  this.initialize.apply(this, arguments);
};
Model.prototype.initialize = function();

Model.extend = View.extend;
```

Responsible views
-----------------

First of all, read [my blog post on views](views.md).

```javascript
// render the view to `$el`
View.prototype.renderTemplate: function(data) {
  template = Mustache.to_html(this.template, data);
  this.$el = $(template);
};

// find something in the views DOM
View.prototype.DOM: function(selector) {
  return this.$el.find(selector);
};
```

Wrapping DOM events
-------------------


Abstracting Ajax
----------------


Built-in events
---------------


