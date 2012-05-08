Mixins in Backbone
==================

In our Backbone.js code we have a few times seen the need to reuse
similar methods in several models, collections and views. I've seen many
create a component which they extend from, e.g. a
`PaginationCollection`, but we didn't like this solution as we only
wanted to share some code between objects, not add another layer to our
architecture (and, even worse, potentially even more layers if we wanted
to include several shared components). Basically we wanted a more
decoupled solution, and we solved it by using mixins.

So — what's a mixin?
--------------------

Basically, a mixin allows you to extend your Backbone components with
utility functions. Let's, for example, say that we have pagination which
is similar in many views, but not in all. How should we include this
functionality?

We have the following API for adding extra functionality:

```javascript
UsersView.mixin(Pagination);
AppView.mixin(Transitions);
```

We have, however, taken the concept of mixins one step further then what
I've seen before. In addition to including all the properties which are
present in the mixin, our implementation enable mixins to include their
own `initialize` which will extend the existing `initialize`. When
mixing into views a mixin can also include its own `render` and
`events`. Let's take a look at what this means in practice:

```javascript
var UserView = BaseView.extend({
    events: {
        "click h1": "user"
    },

    initialize: function() {
        console.log("user init");
    },

    user: function() {
        return this.model.get("name");
    }
});

// This is our mixin:
var Pagination = {

    // expects the view it's mixed into to have a link with class `next`
    // present in the DOM
    events: {
        "click a.next": "next"
    },

    initialize: function() {
        console.log("pagination init");
    },

    next: function() {
        return "next for: " + this.model.get("name");
        // yeah, you would absolutely use a collection as it's named
        // pagination, this was just to keep the example short ;)
    }
};

UserView.mixin(Pagination);

var model = new Backbone.Model();
model.set("name", "Kim Joar");

var view = new UserView({ model: model });
// this initialization console logs (in order):
// "user init"
// "pagination init"

console.log(view.user()); // "Kim Joar"
console.log(view.next()); // "next for: Kim Joar"
console.log(view.events); // {
                          //   "click a.next": "next",
                          //   "click h1": "user"
                          // }
```

As we can see from the code, a mixin has access to `this` in the same
way as the model, collection or view itself. Additionally, mixins can
potentially be self-contained, i.e. because `initialize`, `render` and
`events` can be extended the component we mix into does not need to know
anything about the the mixin. This makes it especially easy to include a
new mixin or remove one that is in use.

Implementation
--------------

Let's have a look at an example of how these mixins can be implemented.
Specifically we will take a look at an implementation of mixins for
views:

```javascript
Utils = {};

Utils.viewMixin = function(from) {
  var to = this.prototype;

  // we add those methods which exists on `from` but not on `to` to the latter
  _.defaults(to, from);
  // … and we do the same for events
  _.defaults(to.events, from.events);

  // we then extend `to`'s `initialize`
  Utils.extendMethod(to, from, "initialize");
  // … and its `render`
  Utils.extendMethod(to, from, "render");
};

// Helper method to extend an already existing method
Utils.extendMethod = function(to, from, methodName) {

  // if the method is defined on from ...
  if (!_.isUndefined(from[methodName])) {
    var old = to[methodName];
    
    // ... we create a new function on to
    to[methodName] = function() {

      // wherein we first call the method which exists on `to`
      var oldReturn = old.apply(this, arguments);

      // and then call the method on `from`
      from[methodName].apply(this, arguments);

      // and then return the expected result,
      // i.e. what the method on `to` returns
      return oldReturn;

    };
  }

};
```

Now we need to include this mixin in such a way that `this` means the
correct thing and we can use it like we saw in the example above. This
can be done in two ways, both of which utilize the `viewMixin` method we
created above:

1. As [mentioned](http://documentcloud.github.com/backbone/#FAQ-extending)
   in the Backbone docs it's okey to add methods directly to a Backbone
   component. We can use this idea and add `mixin` like this:

   ```javascript
   Backbone.View.mixin = Utils.viewMixin;
   ```
2. If you create a layered architecture you can include `viewMixin` in
   one of your layers. In our project we had a `BaseView` which we
   created all our views from. Remember, this is how Backbone views are
   defined:

   ```javascript
   Backbone.View.extend(properties, [classProperties])
   ```

   So, in order to create our desired API we can add `mixin` as a class
   property on our `BaseView`, for example:

   ```javascript
   var BaseView = Backbone.View.extend({
     // lots of methods
   }, {
     mixin: Utils.viewMixin
   })
   ```

In my project we chose to go for the latter solution as we already had a
`BaseView`.

Testing
-------

Mixins can be tested both by themselves and when mixed into a component.
We'll take a look at how the latter can be done.

Jasmine has a great way to include similar tests several places. Davis
W. Frank of Pivotal Labs [wrote a great blog
post](http://pivotallabs.com/users/dwfrank/blog/articles/1720-drying-up-jasmine-specs-with-shared-behavior)
about this a year ago. Basically, the point is to write a function that
contains regular Jasmine specs. For example:

```javascript
function sharedBehaviorForPagination() {
  describe("pagination", function() {
    it("should be able to paginate to the next page", function() {
      // ...
    };

    it("should not be able to paginate to the next page when on the last page", function() {
      // ...
    };
  });
}
```

In the specs for the component we mix into we can then call this
function:

```javascript
describe("users", function() {
  // ... lots of users specific specs

  sharedBehaviorForPagination();
});
```

We found this to be a great technique for ensuring that our mixins works
as expected in all the components which include them.

---

Mixins are a simple but potentially very powerful abstraction. We used
them for mainly for three things: pagination, local error handling and
handling transitions between pages. In each case we found them to help
decouple our code and make each component more focused.

In this blog post I've only looked at using them for views, but they are
also great for breaking up models and collections.

To give credit were credit is due, this blog post, and our solution to
the mixin problem, was heavily influenced by Dmitry Polushkin's [gist on
mixins](https://gist.github.com/1256695).
