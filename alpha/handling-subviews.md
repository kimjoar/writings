Mastering JavaScript views
==========================

Recently [I introduced]() my idea of views in JavaScript. In this blog
post I will elaborate further on what views are and the how they work,
both conceptually and through code. I do, however, recommend that you
read my first blog post on views before reading this.

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

Share functionality between views
---------------------------------

Layers

Handling DOM events
-------------------

Dependencies
------------

Communicating between views
---------------------------

View events vs model events vs global events

Testing views
-------------

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
