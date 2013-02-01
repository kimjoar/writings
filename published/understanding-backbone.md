Step by step from jQuery to Backbone
====================================

I've seen many struggle when they first meet
[Backbone.js](http://backbonejs.org/). In this blog post I will
gradually refactor a bit of code from how I used to write JavaScript
before, into proper Backbone.js code using models, collections, views and
events. Hopefully this process will give you a firm understanding of the
core abstractions in Backbone.js.

Let's start with the code we're going to work with throughout this blog
post:

```javascript
$(document).ready(function() {
    $('#new-status form').submit(function(e) {
        e.preventDefault();

        $.ajax({
            url: '/status',
            type: 'POST',
            dataType: 'json',
            data: { text: $('#new-status').find('textarea').val() },
            success: function(data) {
                $('#statuses').append('<li>' + data.text + '</li>');
                $('#new-status').find('textarea').val('');
            }
        });
    });
});
```

And here's the application up and running:
[Monologue](http://monologue-js.herokuapp.com/). This simple
application is based on a
[great JavaScript presentation](http://opensoul.org/blog/archives/2012/05/16/the-plight-of-pinocchio/)
by [Brandon Keepers](http://opensoul.org). However, while his focus is
primarily on testing and design patterns — of which he does an amazing
job — mine is on explaining the core Backbone.js abstractions one step
at a time.

Basically, the app lets you to write some text, click on "Post", and see
what you've written show up in the list while the textarea is prepared
for new input. Translating this to code, we start by waiting for the DOM
to load, set a submit listener, and when the form is submitted we send
the input to the server, append the response to the list of statuses and
reset the input.

But, what's the problem? This code does a lot of stuff at the same time.
It listens for page events, user events, network events, it does network
IO, handles user input, parses the response, and does HTML templating.
All in 16 lines of code. It looks like most JavaScript code I wrote a
year ago. Throughout this blog post I will gradually work my way to code
that follows the single responsibility principle, and which is far
easier to test, maintain, reuse and extend.

There are three things we want to achieve:

* We want to move as much as possible out of `$(document).ready`, so we
  can trigger the code ourselves — we only want to bootstrap the
  application when the DOM is ready. In its current state the code is
  nearly impossible to test.
* We want to adhere to the single responsibility principle, and
  make the code more reusable and easier to test.
* We want to break the coupling between the DOM and Ajax.

Separating DOM and Ajax
-----------------------

We start with splitting Ajax and DOM from each other, and the first step
is to create an `addStatus` function:

```diff
+function addStatus(options) {
+    $.ajax({
+        url: '/status',
+        type: 'POST',
+        dataType: 'json',
+        data: { text: $('#new-status textarea').val() },
+        success: function(data) {
+            $('#statuses ul').append('<li>' + data.text + '</li>');
+            $('#new-status textarea').val('');
+        }
+    });
+}
+
 $(document).ready(function() {
     $('#new-status form').submit(function(e) {
         e.preventDefault();
 
-        $.ajax({
-            url: '/status',
-            type: 'POST',
-            dataType: 'json',
-            data: { text: $('#new-status textarea').val() },
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
+        addStatus();
     });
 });
```

However, both within `data` and `success` we work with the DOM. We can
break this coupling by sending these as arguments to `addStatus`:

```diff
 function addStatus(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
-        data: { text: $('#new-status textarea').val() },
-        success: function(data) {
-            $('#statuses ul').append('<li>' + data.text + '</li>');
-            $('#new-status textarea').val('');
-        }
+        data: { text: options.text },
+        success: options.success
     });
 }
 
 $(document).ready(function() {
     $('#new-status form').submit(function(e) {
         e.preventDefault();
 
-        addStatus();
+        addStatus({
+            text: $('#new-status textarea').val(),
+            success: function(data) {
+                $('#statuses ul').append('<li>' + data.text + '</li>');
+                $('#new-status textarea').val('');
+            }
+        });
     });
 });
```

However, I want to wrap these statuses in an object and be able to write
`statuses.add` instead of `addStatus`. To achieve this we can use the
[constructor pattern with prototypes](http://addyosmani.com/resources/essentialjsdesignpatterns/book/#constructorpatternjavascript)
to introduce a Statuses "class":

```diff
-function addStatus(options) {
+var Statuses = function() {
+};
+Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
-}
+};
 
 $(document).ready(function() {
+    var statuses = new Statuses();
+
     $('#new-status form').submit(function(e) {
         e.preventDefault();
 
-        addStatus({
+        statuses.add({
             text: $('#new-status textarea').val(),
             success: function(data) {
                 $('#statuses ul').append('<li>' + data.text + '</li>');
                 $('#new-status textarea').val('');
             }
         });
     });
 });
```

Creating a view
---------------

Our submit handler now has one dependency, the `statuses` variable, and
everything else within it is focused on the DOM. Let's move the submit
handler and everything inside it into its own class, `NewStatusView`:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };
 
+var NewStatusView = function(options) {
+    var statuses = options.statuses;
+
+    $('#new-status form').submit(function(e) {
+        e.preventDefault();
+
+        statuses.add({
+            text: $('#new-status textarea').val(),
+            success: function(data) {
+                $('#statuses ul').append('<li>' + data.text + '</li>');
+                $('#new-status textarea').val('');
+            }
+        });
+    });
+};
+
 $(document).ready(function() {
     var statuses = new Statuses();
 
-    $('#new-status form').submit(function(e) {
-        e.preventDefault();
-
-        statuses.add({
-            text: $('#new-status textarea').val(),
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
-    });
+    new NewStatusView({ statuses: statuses });
 });
```

Now we only bootstrap our application when the DOM is loaded, and
everything else is moved out of `$(document).ready`. The steps we have
taken so far has given us two components which are easier to test and
have more well-defined responsibilities. However, there is still much to clean
up. Let's start by splitting the submit handler in `NewStatusView` into
its own `addStatus` method:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };
 
 var NewStatusView = function(options) {
-    var statuses = options.statuses;
+    this.statuses = options.statuses;
 
-    $('#new-status form').submit(function(e) {
-        e.preventDefault();
-        statuses.add({
-            text: $('#new-status textarea').val(),
-            success: function(data) {
-                $('#statuses ul').append('<li>' + data.text + '</li>');
-                $('#new-status textarea').val('');
-            }
-        });
-    });
+    $('#new-status form').submit(this.addStatus);
 };
+NewStatusView.prototype.addStatus = function(e) {
+    e.preventDefault();
+
+    this.statuses.add({
+        text: $('#new-status textarea').val(),
+        success: function(data) {
+            $('#statuses ul').append('<li>' + data.text + '</li>');
+            $('#new-status textarea').val('');
+        }
+    });
+};
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Running this in Chrome we get this error:

    Uncaught TypeError: Cannot call method 'add' of undefined

We get this error as `this` means different things in the constructor
and the `addStatus` method, as it is jQuery that actually calls the
latter when the user submits the form. (If you don't fully grasp how
`this` works, I recommend reading
[Understanding JavaScript Function Invocation and “this”](http://yehudakatz.com/2011/08/11/understanding-javascript-function-invocation-and-this/).)
To solve this problem we can use
[`$.proxy`](http://api.jquery.com/jQuery.proxy/), which creates a
function where `this` is always the same — the context you specify as
the second argument.

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
-    $('#new-status form').submit(this.addStatus);
+    var add = $.proxy(this.addStatus, this);
+    $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
             $('#statuses ul').append('<li>' + data.text + '</li>');
             $('#new-status textarea').val('');
         }
     });
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Let's make the `success` callback call methods instead of working
directly on the DOM, which makes the callback easier to read and work
with:

```diff
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
+    var that = this;
+
     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
-            $('#statuses ul').append('<li>' + data.text + '</li>');
-            $('#new-status textarea').val('');
+            that.appendStatus(data.text);
+            that.clearInput();
         }
     });
 };
+NewStatusView.prototype.appendStatus = function(text) {
+    $('#statuses ul').append('<li>' + text + '</li>');
+};
+NewStatusView.prototype.clearInput = function() {
+    $('#new-status textarea').val('');
+};
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

This is much easier to test and to work with as our code
grows. We are also far on our way to understand Backbone.js.

Adding events
-------------

For the next step we need to introduce our first bit of Backbone:
events. Events are basically just a way to say: "Hi, I want to know when
some action occurs" and "Hi, you know what? The action you're waiting
for just occurred!" We are used to this idea from jQuery DOM events such
as listening for `click` and `submit`, e.g.

```javascript
$('form').bind('submit', function() {
  alert('User submitted form'); 
});
```

The Backbone.js documentation describes
[`Backbone.Events`](http://backbonejs.org/#Events) as follows: *"Events
is a module that can be mixed in to any object, giving the object the
ability to bind and trigger custom named events."* The docs also shows
us how we can use [Underscore.js](http://underscorejs.org/) to create an
event dispatcher, i.e. a component on which we can bind and trigger
events:

```javascript
var events = _.clone(Backbone.Events);
```

With this little bit of functionality we can let the `success` callback
trigger an event instead of calling methods. We can also declare in the
constructor which methods we want to call when the event is triggered:

```diff
+var events = _.clone(Backbone.Events);
+
 var Statuses = function() {
 };
 Statuses.prototype.add = function(options) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: options.text },
         success: options.success
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
+    events.on('status:add', this.appendStatus, this);
+    events.on('status:add', this.clearInput, this);
+
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
-    var that = this;
-
     this.statuses.add({
         text: $('#new-status textarea').val(),
         success: function(data) {
-            that.appendStatus(data.text);
-            that.clearInput();
+            events.trigger('status:add', data.text);
         }
     });
 };
 NewStatusView.prototype.appendStatus = function(text) {
     $('#statuses ul').append('<li>' + text + '</li>');
 };
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

Now we can declare in the constructor what we want to happen when a
status is added, instead of `addStatus` being responsible for handling
success. The only responsibility `addStatus` should have is backend
communication, not updating the DOM.

As we no longer deal with the view in the `success` callback we can move
the triggering of the event into the `add` method on `Statuses`:

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
-Statuses.prototype.add = function(options) {
+Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
-        data: { text: options.text },
-        success: options.success
+        data: { text: text },
+        success: function(data) {
+            events.trigger('status:add', data.text);
+        }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
     events.on('status:add', this.appendStatus, this);
     events.on('status:add', this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
-    this.statuses.add({
-        text: $('#new-status textarea').val(),
-        success: function(data) {
-            events.trigger('status:add', data.text);
-        }
-    });
+    this.statuses.add($('#new-status textarea').val());
 };
 NewStatusView.prototype.appendStatus = function(text) {
     $('#statuses ul').append('<li>' + text + '</li>');
 };
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
 });
```

A view's responsibilities
-------------------------

Looking at `appendStatus` and `clearInput` in `NewStatusView`, we see
that these methods focus on two different DOM elements, `#statuses` and
`#new-status`, respectively. I've given them different colors in the
[app](http://monologue-js.herokuapp.com/), so you can see the
difference. Working on two elements in a view does not adhere to the
principles I outline in my blog post on
[a view's responsibilities](https://open.bekk.no/a-views-responsibility/).
Let's pull a `StatusesView` out of `NewStatusView`, and let it be
responsible for `#statuses`. Separating these responsibilities is
especially simple now that we use events — with a regular `success`
callback this would be far more difficult.

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
-    events.on('status:add', this.appendStatus, this);
     events.on('status:add', this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
     this.statuses.add($('#new-status textarea').val());
 };
-NewStatusView.prototype.appendStatus = function(text) {
-    $('#statuses ul').append('<li>' + text + '</li>');
-};
 NewStatusView.prototype.clearInput = function() {
     $('#new-status textarea').val('');
 };
 
+var StatusesView = function() {
+    events.on('status:add', this.appendStatus, this);
+};
+StatusesView.prototype.appendStatus = function(text) {
+    $('#statuses ul').append('<li>' + text + '</li>');
+};
+
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
+    new StatusesView();
 });
```

As each view is now responsible for only one HTML element, we can specify
them when instantiating the views:

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
+    this.el = $('#new-status');
 
     events.on('status:add', this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
-    $('#new-status form').submit(add);
+    this.el.find('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
-    this.statuses.add($('#new-status textarea').val());
+    this.statuses.add(this.el.find('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
-    $('#new-status textarea').val('');
+    this.el.find('textarea').val('');
 };
 
 var StatusesView = function() {
+    this.el = $('#statuses');
+
     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
-    $('#statuses ul').append('<li>' + text + '</li>');
+    this.el.find('ul').append('<li>' + text + '</li>');
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ statuses: statuses });
     new StatusesView();
 });
```

Our views, `NewStatusView` and `StatusesView` are still difficult to
test because they depend on having the HTML present, e.g. in order to
find `$('#statuses')`. To remedy this we can pass in its DOM
dependencies when we instantiate a view.

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
-    this.el = $('#new-status');
+    this.el = options.el;
 
     events.on('status:add', this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
     this.el.find('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
     this.statuses.add(this.el.find('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
     this.el.find('textarea').val('');
 };
 
-var StatusesView = function() {
-    this.el = $('#statuses');
+var StatusesView = function(options) {
+    this.el = options.el;
 
     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
     this.el.find('ul').append('<li>' + text + '</li>');
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
-    new NewStatusView({ statuses: statuses });
-    new StatusesView();
+    new NewStatusView({ el: $('#new-status'), statuses: statuses });
+    new StatusesView({ el: $('#statuses') });
 });
```

Now, this is easy to test! With this change we can use a
[jQuery trick](http://api.jquery.com/jQuery/#jQuery2) to test our views.
Instead of initiating our views by passing in for example
`$('#new-status')`, we can pass in the necessary HTML wrapped in jQuery,
e.g. `$('<div><form>…</form></div>')`. jQuery will then create the
needed DOM elements on the fly. This ensures blazingly fast tests — on
my current project our nearly 200 tests run in less than one second.

Our next step is introducing a helper to clean up our views a little
bit. Instead of writing `this.el.find` we can create a simple helper so
we can write `this.$` instead. With this little change it feels like we
are saying, I want to use jQuery to look for something locally on this
view instead of globally in the entire HTML. And it's so easy to add:

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
     this.el = options.el;
 
     events.on('status:add', this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
-    this.el.find('form').submit(add);
+    this.$('form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
-    this.statuses.add(this.el.find('textarea').val());
+    this.statuses.add(this.$('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
-    this.el.find('textarea').val('');
+    this.$('textarea').val('');
 };
+NewStatusView.prototype.$ = function(selector) {
+    return this.el.find(selector);
+};
 
 var StatusesView = function(options) {
     this.el = options.el;
 
     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
-    this.el.find('ul').append('<li>' + text + '</li>');
+    this.$('ul').append('<li>' + text + '</li>');
 };
+StatusesView.prototype.$ = function(selector) {
+    return this.el.find(selector);
+};
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

However, adding this functionality for every view is a pain. That's one
of the reasons to use [Backbone.js views](http://backbonejs.org/#View) —
reusing functionality across views.

Getting started with views in Backbone
--------------------------------------

With the current state of our code, it's just a couple of lines of
change needed to add Backbone.js views:

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
-var NewStatusView = function(options) {
-    this.statuses = options.statuses;
-    this.el = options.el;
-
-    events.on('status:add', this.clearInput, this);
-
-    var add = $.proxy(this.addStatus, this);
-    this.$('form').submit(add);
-};
+var NewStatusView = Backbone.View.extend({
+    initialize: function(options) {
+        this.statuses = options.statuses;
+        this.el = options.el;
+
+        events.on('status:add', this.clearInput, this);
+
+        var add = $.proxy(this.addStatus, this);
+        this.$('form').submit(add);
+    }
+});
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
     this.statuses.add(this.$('textarea').val());
 };
 NewStatusView.prototype.clearInput = function() {
     this.$('textarea').val('');
 };
 NewStatusView.prototype.$ = function(selector) {
     return this.el.find(selector);
 };
 
 var StatusesView = function(options) {
     this.el = options.el;
 
     events.on('status:add', this.appendStatus, this);
 };
 StatusesView.prototype.appendStatus = function(text) {
     this.$('ul').append('<li>' + text + '</li>');
 };
 StatusesView.prototype.$ = function(selector) {
     return this.el.find(selector);
 };
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

As you can see from the code, we use `Backbone.View.extend` to create a
new view class in Backbone. Within `extend` we can specify instance
methods such as `initialize`, which is the name of the constructor.

Now that we have started the move over to Backbone.js views, let's go on
and move both views fully over:

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
         this.el = options.el;
 
         events.on('status:add', this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
-    }
+    },
+
+    addStatus: function(e) {
+        e.preventDefault();
+
+        this.statuses.add(this.$('textarea').val());
+    },
+
+    clearInput: function() {
+        this.$('textarea').val('');
+    },
+
+    $: function(selector) {
+        return this.el.find(selector);
+    }
 });
-NewStatusView.prototype.addStatus = function(e) {
-    e.preventDefault();
-
-    this.statuses.add(this.$('textarea').val());
-};
-NewStatusView.prototype.clearInput = function() {
-    this.$('textarea').val('');
-};
-NewStatusView.prototype.$ = function(selector) {
-    return this.el.find(selector);
-};
 
-var StatusesView = function(options) {
-    this.el = options.el;
-
-    events.on('status:add', this.appendStatus, this);
-};
-StatusesView.prototype.appendStatus = function(text) {
-    this.$('ul').append('<li>' + text + '</li>');
-};
-StatusesView.prototype.$ = function(selector) {
-    return this.el.find(selector);
-};
+var StatusesView = Backbone.View.extend({
+    initialize: function(options) {
+        this.el = options.el;
+
+        events.on('status:add', this.appendStatus, this);
+    },
+
+    appendStatus: function(text) {
+        this.$('ul').append('<li>' + text + '</li>');
+    },
+
+    $: function(selector) {
+        return this.el.find(selector);
+    }
+});
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Now that we use Backbone.js views we can remove the `this.$` helper, as it
already exists in Backbone. We also no longer need to set `this.el`
ourselves, as Backbone.js does it automatically when a view is instantiated
with an HTML element.

```diff
 var events = _.clone(Backbone.Events);
 
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
     $.ajax({
         url: '/status',
         type: 'POST',
         dataType: 'json',
         data: { text: text },
         success: function(data) {
             events.trigger('status:add', data.text);
         }
     });
 };
 
 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
-        this.el = options.el;
 
         events.on('status:add', this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
         this.statuses.add(this.$('textarea').val());
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     },
-
-    $: function(selector) {
-        return this.el.find(selector);
-    }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
-        this.el = options.el;
-
         events.on('status:add', this.appendStatus, this);
     },
 
     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     },
-
-    $: function(selector) {
-        return this.el.find(selector);
-    }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Let's use a model
-----------------

The next step is introducing models, which are responsible for the
network traffic, i.e. Ajax requests and responses. As Backbone nicely
abstracts Ajax, we don't need to specify `type`, `dataType` and `data`
anymore. Now we only need to specify the URL and call `save` on the
model. The `save` method accepts the data we want to save as the first
parameter, and options, such as the `success` callback, as the second
parameter.

```diff
 var events = _.clone(Backbone.Events);
 
+var Status = Backbone.Model.extend({
+    url: '/status'
+});
+
 var Statuses = function() {
 };
 Statuses.prototype.add = function(text) {
-    $.ajax({
-        url: '/status',
-        type: 'POST',
-        dataType: 'json',
-        data: { text: text },
-        success: function(data) {
-            events.trigger('status:add', data.text);
-        }
-    });
+    var status = new Status();
+    status.save({ text: text }, {
+        success: function(model, data) {
+            events.trigger('status:add', data.text);
+        }
+    });
 };
 
 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
 
         events.on('status:add', this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
         this.statuses.add(this.$('textarea').val());
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         events.on('status:add', this.appendStatus, this);
     },
 
     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses') });
 });
```

Handling several models
-----------------------

Now that we have introduced models, we need a concept for a list of
models, such as the list of statuses in our application. In Backbone.js 
this concept is called a collection.

One really cool thing about collections is that they have scoped events.
Basically, this just means that we can bind and trigger events directly
on a collection instead of using our `events` variable — our events will
live on `statuses` instead of `events`. As we now start firing events
directly on statuses there's no need for "status" in the event name, so
we rename it from "status:add" to "add".

```diff
-var events = _.clone(Backbone.Events);
-
 var Status = Backbone.Model.extend({
     url: '/status'
 });
 
-var Statuses = function() {
-};
-Statuses.prototype.add = function(text) {
-    var status = new Status();
-    status.save({ text: text }, {
-        success: function(model, data) {
-            events.trigger("status:add", data.text);
-        }
-    });
-};
+var Statuses = Backbone.Collection.extend({
+    add: function(text) {
+        var that = this;
+        var status = new Status();
+        status.save({ text: text }, {
+            success: function(model, data) {
+                that.trigger("add", data.text);
+            }
+        });
+    }
+});
 
 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
 
-        events.on("status:add", this.clearInput, this);
+        this.statuses.on("add", this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
         this.statuses.add(this.$('textarea').val());
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
+        this.statuses = options.statuses;
+
-        events.on("status:add", this.appendStatus, this);
+        this.statuses.on("add", this.appendStatus, this);
     },
 
     appendStatus: function(text) {
         this.$('ul').append('<li>' + text + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
-    new StatusesView({ el: $('#statuses') });
+    new StatusesView({ el: $('#statuses'), statuses: statuses });
 });
```

We can simplify this even more by using Backbone's `create` method. It
creates a new model instance, adds it to the collection and saves it to
the server. Therefore we must specify what type of model the collection
should handle. There are two things we need to change to use Backbone
collections:

1. When creating the status we need to pass in an options hash with the
   attributes we want to save, instead of only passing the text.
2. The built in `create` also triggers an "add" event, but rather than
   passing only the text, as we have done so far, it passes the newly
   created model. We can get the text from the model by calling
   `model.get("text")`.

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });
 
 var Statuses = Backbone.Collection.extend({
-    add: function(text) {
-        var that = this;
-        var status = new Status();
-        status.save({ text: text }, {
-            success: function(model, data) {
-                that.trigger("add", data.text);
-            }
-        });
-    }
+    model: Status
 });
 
 var NewStatusView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
 
         this.statuses.on("add", this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
-        this.statuses.add(this.$('textarea').val());
+        this.statuses.create({ text: this.$('textarea').val() });
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         this.statuses = options.statuses;
 
         this.statuses.on("add", this.appendStatus, this);
     },
 
-    appendStatus: function(text) {
+    appendStatus: function(status) {
-        this.$('ul').append('<li>' + text + '</li>');
+        this.$('ul').append('<li>' + status.get("text") + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), statuses: statuses });
     new StatusesView({ el: $('#statuses'), statuses: statuses });
 });
```

As with `el` earlier, Backbone.js automatically sets `this.collection`
when `collection` is passed. Therefore we rename `statuses` to
`collection` in our view:

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });
 
 var Statuses = Backbone.Collection.extend({
     model: Status
 });
 
 var NewStatusView = Backbone.View.extend({
-    initialize: function(options) {
+    initialize: function() {
-        this.statuses = options.statuses;
-
-        this.statuses.on('add', this.clearInput, this);
+        this.collection.on('add', this.clearInput, this);
 
         var add = $.proxy(this.addStatus, this);
         this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
-        this.statuses.add({ text: this.$('textarea').val() });
+        this.collection.create({ text: this.$('textarea').val() });
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
-    initialize: function(options) {
+    initialize: function() {
-        this.statuses = options.statuses;
-
-        this.statuses.on('add', this.appendStatus, this);
+        this.collection.on('add', this.appendStatus, this);
     },
 
     appendStatus: function(status) {
         this.$('ul').append('<li>' + status.get('text') + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
-    new NewStatusView({ el: $('#new-status'), statuses: statuses });
+    new NewStatusView({ el: $('#new-status'), collection: statuses });
-    new StatusesView({ el: $('#statuses'), statuses: statuses });
+    new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

Evented views
-------------

Now, let's get rid of that nasty `$.proxy` stuff. We can do this by
letting Backbone.js [delegate our events](http://backbonejs.org/#View-delegateEvents)
by specifying them in an `events` hash in the view. This hash is of the
format `{"event selector": "callback"}`:

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });
 
 var Statuses = Backbone.Collection.extend({
     model: Status
 });
 
 var NewStatusView = Backbone.View.extend({
+    events: {
+        'submit form': 'addStatus'
+    },
+
     initialize: function() {
         this.collection.on('add', this.clearInput, this);
-
-        var add = $.proxy(this.addStatus, this);
-        this.$('form').submit(add);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
         this.collection.create({ text: this.$('textarea').val() });
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function() {
         this.collection.on('add', this.appendStatus, this);
     },
 
     appendStatus: function(status) {
         this.$('ul').append('<li>' + status.get('text') + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), collection: statuses });
     new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

Escape it!
----------

As our last step we're going to prevent XSS-exploits. Instead of using
`model.get('text')` let's use the
[built-in escape handling](http://backbonejs.org/#Model-escape) and
write `model.escape('text')` instead. If you're using
[Handlebars](http://handlebarsjs.com/),
[Mustache](https://github.com/janl/mustache.js) or similar templating
engines, you might get this functionality out of the box.

```diff
 var Status = Backbone.Model.extend({
     url: '/status'
 });
 
 var Statuses = Backbone.Collection.extend({
     model: Status
 });
 
 var NewStatusView = Backbone.View.extend({
     events: {
         "submit form": "addStatus"
     },
 
     initialize: function(options) {
         this.collection.on("add", this.clearInput, this);
     },
 
     addStatus: function(e) {
         e.preventDefault();
 
         this.collection.create({ text: this.$('textarea').val() });
     },
 
     clearInput: function() {
         this.$('textarea').val('');
     }
 });
 
 var StatusesView = Backbone.View.extend({
     initialize: function(options) {
         this.collection.on("add", this.appendStatus, this);
     },
 
     appendStatus: function(status) {
-        this.$('ul').append('<li>' + status.get("text") + '</li>');
+        this.$('ul').append('<li>' + status.escape("text") + '</li>');
     }
 });
 
 $(document).ready(function() {
     var statuses = new Statuses();
     new NewStatusView({ el: $('#new-status'), collection: statuses });
     new StatusesView({ el: $('#statuses'), collection: statuses });
 });
```

And we're done!
---------------

This is our final code:

```javascript
var Status = Backbone.Model.extend({
    url: '/status'
});

var Statuses = Backbone.Collection.extend({
    model: Status
});

var NewStatusView = Backbone.View.extend({
    events: {
        'submit form': 'addStatus'
    },

    initialize: function() {
        this.collection.on('add', this.clearInput, this);
    },

    addStatus: function(e) {
        e.preventDefault();

        this.collection.create({ text: this.$('textarea').val() });
    },

    clearInput: function() {
        this.$('textarea').val('');
    }
});

var StatusesView = Backbone.View.extend({
    initialize: function() {
        this.collection.on('add', this.appendStatus, this);
    },

    appendStatus: function(status) {
        this.$('ul').append('<li>' + status.escape('text') + '</li>');
    }
});

$(document).ready(function() {
    var statuses = new Statuses();
    new NewStatusView({ el: $('#new-status'), collection: statuses });
    new StatusesView({ el: $('#statuses'), collection: statuses });
});
```

And [here's](http://monologue-js.herokuapp.com/?step=22) the application
running with the refactored code. Yeah, it's still the exact same
application from a user's point of view.

However, the code has increased from 16 lines to more than 40, so why do
I think this is better? Because we are now working on a higher level of
abstraction. This code is more maintainable, easier to reuse and extend,
and easier to test. What I've seen is that Backbone.js helps improve the
structure of my JavaScript applications considerably, and in my
experience the end result is often less complex and has fewer lines
of code than my "regular JavaScript".

Want to learn more?
-------------------

By now you should understand far more of Backbone.js than when you did
an hour ago. There are some great resources for learning Backbone.js out
there, but there's also a whole lot of crap. Actually, the
[Backbone.js documentation](http://backbonejs.org/) is superb as soon as
you have a better understanding of how the framework works at a higher
level. Addy Osmani's
[Developing Backbone.js Applications](http://addyosmani.github.com/backbone-fundamentals/)
is another good source.

Reading the
[annotated source](http://documentcloud.github.com/backbone/docs/backbone.html)
of Backbone.js is also a great way to get a better understanding of how
it works. I found this to be an enlightening experience when I had used
Backbone.js for some weeks.

If you want to get started with Require.js in a Backbone app, you can
check out
[the follow-up](https://github.com/kjbekkelund/writings/blob/master/published/step-by-step-modules.md)
to this blog post.

And, lastly, if you want to get a better understanding of the creation
of a framework such as Backbone.js, I recommend the magnificent
[JavaScript Web Applications](http://www.amazon.com/JavaScript-Web-Applications-Alex-MacCaw/dp/144930351X)
by [Alex MacCaw](http://alexmaccaw.com/).
