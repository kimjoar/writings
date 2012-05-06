A view inside a view inside a view — mastering a JavaScript abstraction
=======================================================================

In [my blog post]() on a view's responsibilities I mentioned that one of
these responsibities is creating subviews. However, in that blog post I
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
