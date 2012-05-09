A view inside a view inside a view — mastering a JavaScript abstraction
=======================================================================

In [my blog post]() on a view's responsibilities I mentioned that one of
its responsibities is creating subviews. However, in that blog post I
didn't further elaborate on them, so let's now take a look at what
subviews are and how they work, both conceptually and through code.

What's a sub-view?
------------------



Creating subviews
-----------------

---

Notes:

* A view should never ever attach itself to the DOM. It's always the
  parent view's resposibility to create and attach a view to the DOM.
  Why? What happens when you want to move the view? Or use it in two
  different locations on two different pages? It's far easier to reuse a
  subview when it's the parent which actually places it in the DOM.
* Every subview is just a black box
* Events are "fire and forget". It's not and will never be the view's
  responsibility that someone actually listens for the event.
* We had ONE global `$`, it looked like this: `$("body")`, and it was
  the container for the entire app, i.e. the `el` for our Backbone view
  named `appView`. Because of this, we could, theoretically, easily
  place our entire app into another page with lots of stuff in the DOM
  around it.
* Should a view be responsible for starting to fetch the information it
  needs? No. The router/controller. The view should be as context
  independent as possible.
* It's exceedingly important that a view has no knowledge about its
  subviews except for their `el`, other dependencies, and their API,
  i.e. the events the view trigger. This simplifies testing, it
  simplifies reuse and it simplifies maintenance.
