Mastering JavaScript views
==========================

In [my recent blog post]() on views I mentioned that one of a view's
responsibities is creating subviews. However, in that blog post I didn't
further elaborate on them, so let's now take a look at what subviews are
and how they work, both conceptually and through code.

What's a sub-view?
------------------

A sub-view is just a view. That's it — nothing special. The essence of
sub-views, however, is that they help a view split its responsiblity —
its HTML element — into several minor components. So from the outside a
view should just be a black box that is responsible for its HTML
element, and which, itself, can create even smaller black boxes. The
only important aspect is that it receives its dependencies and that it
communicates through its public api, i.e. through events that other
components can subscribe to.

So let's start with a code example:

```javascript
// blabla
```

Creating subviews
-----------------

---

Notes:

* A view should never ever attach itself to the DOM. It's always the
  parent view's resposibility to create and attach a view to the DOM.
  Why? What happens when you want to move the view? Or use it in two
  different locations on two different pages? It's far easier to reuse a
  subview when it's the parent which actually places it in the DOM.
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
