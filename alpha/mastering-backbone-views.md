Mastering Backbone.js views
===========================

Recently [I introduced]() how I look at views in JavaScript. In this
blog post I will elaborate further on what views are and the how they
work, both conceptually and through code, with a primary focus on how I
use them when using Backbone.js.

From the Backbone.js docs: "Backbone views are almost more convention
than they are code". But what are these conventions and which patterns
are important to understand and use?  Backbone.js views can be used in
many different ways, this is my way of working with them, which has
proved successful on several large-scale JavaScript applications.

So first, what is a view?
-------------------------

The essence of views is that they help split the DOM into several
smaller responsibilities. From the outside a view is just a black box
responsible for one HTML element and its content. Additionally, views
can themselves create even smaller black boxes responsible for a subset
of the views HTML element.

Throughout this blog post I will use an example application called
Monolog to introduce the different concepts. First of all, lets start
with a figure that illustrates what a view is:

FIGUR!

I've drawn borders on the boxes that represent views.

Creating views
--------------

A view should never ever attach itself to the DOM. It's always the
parent view's resposibility to create and attach a view to the DOM.

Ehm, what does that mean?

```javascript
// example
```

So, why is this important? What happens when you want to move the view?
Or use it in two different locations on two different pages? It's far
easier to reuse a view when it's the creator who actually places it in
the DOM. It is also far easier to test, which we will discuss more in
depth later in this blog post. Less coupled to the DOM.

Should a view be responsible for starting to fetch the information it
needs? No. The router/controller. The view should be as context
independent as possible. Increased reusability. More control. Easier to
test. Do as little as possible in the constructor, which is an accepted
"pattern" in Java et al.

Never ever ever ever use `$` directly to find something in the DOM.
Always use `this.$`.

* We had ONE global `$`, it looked like this: `$("body")`, and it was
  the container for the entire app, i.e. the `el` for our Backbone view
  named `appView`. Because of this, we could, theoretically, easily
  place our entire app into another page with lots of stuff in the DOM
  around it.

Share functionality between views
---------------------------------

Layers, mixins

Shared handling of errors, spinners, and so on

Subviews
--------

It is important that a view has no knowledge about its subviews except
for their `el`, other dependencies, and their API, i.e.  the events the
view trigger. This simplifies testing, it simplifies reuse and it
simplifies maintenance. A subview is just a black box, and they are used
to split up the responbilities in an application.

Handling DOM events
-------------------

Declarative.

Event-driven
------------

Events are "fire and forget". It's not and will never be the view's
responsibility that someone actually listens for the event.

Destroying views
----------------

Events, GC, subviews

Dependencies
------------

Constructor injection, e.g. pass in models, collection, whatever -- but
not subviews, they are created in the view. They are just internal black
boxes.

Communicating between views
---------------------------

View events vs model events vs global events

Testing views
-------------

Coarse tests, which go through the entire front-end stack. I usually
mock Ajax and test that the view works as intended. Why?

We don't need no DOM. jQuery ftw. `this.$` makes this possible.

Ensure that subviews are present. Otherwise each subview is tested by
itself.

Putting it all together
-----------------------


