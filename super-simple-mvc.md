Creating a simplistic MVC in 50 lines of JavaScript
===================================================

I've been using Backbone.js for several projects that last year, but
recently I started thinking about how easy it is to create some of the
core abstractions in Backbone.js.

For me these are the most important aspects of Backbone.js:

* A simple way of creating new components which inherit functionality
  from a parent.
* Creating [responsible views](), so each view *owns* an HTML element.
* Views which wrap DOM-events, so we can specify all our DOM bindings
  for a view in one place.
* Models that abstract Ajax, so we don't have to perform `$.ajax`
  ourselves.
* Built-in events handling, so we can be notified e.g. when Ajax
  responses are received. This is at the core of decoupling JavaScript
  code.

Let's go through these in order, and see how we can create a super
simple MVC in less than 50 lines of code while using some great existing
libraries. (The essence of this is not that it is short, but that it is
suprisingly simple to do.)

Creating new components
-----------------------


Responsible views
-----------------


Wrapping DOM events
-------------------


Abstracting Ajax
----------------


Built-in events
---------------


