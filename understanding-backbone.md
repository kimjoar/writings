Going from jQuery JavaScript to Backbone.js
===========================================

I've seen several people struggle the first few times they've tried to
use Backbone.js. In this blog post I will gradually refactor a bit of
code from the way I see most JavaScript code written into proper
Backbone.js code using of models, collections, views and events.

First, let's start with the code we're going to work with. This simple
application is based on [Brandon Keepers](http://opensoul) in a
[great JavaScript presentation](http://opensoul.org/blog/archives/2012/05/16/the-plight-of-pinocchio/)
by .org/). However, while his focus was
primarily on testing — of which he does an amazing job — mine is on
explaining the core Backbone.js abstractions one step at a time.

These 16 lines of code is what we're going to work with throughout this
blog post:

```javascript
Query(function() {
    $('#new-status').submit(function(e) {
        e.preventDefault();

        $.ajax({
            url: '/status',
            type: 'POST',
            dataType: 'json',
            data: { text: $(this).find('textarea').val() },
            success: function(data) {
                $('#statuses').append('<li>' + data.text + '</li>');
                $(this).find('textarea').val('');
            }
        });
    });
});
```

[Check out the application](...). Basically, you can write some text,
click on "Post", and see what you've written show up in the list while
the textarea is prepared for new input. In this little bit of code we
do this by waiting for the DOM to be ready, set a submit listener, and
when the form is submitted we send the input to the server, append the
response to the list of statuses and reset the input.

Our first goal is to split Ajax and DOM from each other. The first is
creating an `addStatus` method:

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

However, both within `data` and `success` we work with the DOM.
We can break this coupling by sending these as arguments:

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

We then use the
[constructor pattern with prototypes](http://addyosmani.com/resources/essentialjsdesignpatterns/book/#constructorpatternjavascript)
to introduce a Statuses "class":

```diff
-function addStatus(options) {
-    $.ajax({
-        url: '/status',
-        type: 'POST',
-        dataType: 'json',
-        data: { text: options.text },
-        success: options.success
-    });
-}
+var Statuses = function() {
+};
+Statuses.prototype.add = function(options) {
+    $.ajax({
+        url: '/status',
+        type: 'POST',
+        dataType: 'json',
+        data: { text: options.text },
+        success: options.success
+    });
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

Our submit handler now has one dependency, `statuses`, and everything
else within it relates to the
[responsibilities of a view](http://open.bekk.no/a-views-responsibility/).
Let's move the submit handler and everything inside it into a
`NewStatusView`:

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

Now, our jQuery ready handling is much cleaner than when we started.
To clean up the `NewStatusView` we can start by splitting the handling
of the submit into its own `addStatus` method.

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

Running this in Chrome this gives me this error:

    Uncaught TypeError: Cannot call method 'add' of undefined

The reason is that `this` means different things in the constructor and
the `addStatus` method, as it is jQuery that actually calls the latter.
(If you don't understand this, read
[Understanding JavaScript Function Invocation and “this”](http://yehudakatz.com/2011/08/11/understanding-javascript-function-invocation-and-this/))

To get this working we need to use
[`$.proxy`](http://api.jquery.com/jQuery.proxy/) for `this` to mean the
right thing when calling `addStatus`, as we need to reach
`this.statuses`:

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
directly on the DOM:

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

This is much easier to test, and much easier to work with as our code
grows. We are also far on our way to understand Backbone.js. For the
next step we need to introduce our first bit of Backbone: events. Events
are basically just a way to say: "Hi, I want to know when some action
occurs" and "Hi, you know what? The action you're waiting for just
occurred!" We are used to this idea from jQuery DOM events such as
listening for `click` and `submit`.

The Backbone documentation describes `Backbone.Events` as follows:
"Events is a module that can be mixed in to any object, giving the
object the ability to bind and trigger custom named events." The docs
also shows us how we can create an event dispatcher:

```javascript
var events = _.clone(Backbone.Events);
```

With this little bit of functionality we can let the `success` callback
trigger an event instead of calling methods. We can also set up in the
constructor which methods we want called when the success event is
triggered:

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
 
+    events.on("status:added", this.appendStatus, this);
+    events.on("status:added", this.clearInput, this);
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
+            events.trigger("status:added", data.text);
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

Now, we don't have special handling in the `success` callback, so we can
move the triggering of the event into the `add` method on `Statuses`:

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
+            events.trigger("status:added", data.text);
+        }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
     events.on("status:added", this.appendStatus, this);
     events.on("status:added", this.clearInput, this);
 
     var add = $.proxy(this.addStatus, this);
     $('#new-status form').submit(add);
 };
 NewStatusView.prototype.addStatus = function(e) {
     e.preventDefault();
 
-    this.statuses.add({
-        text: $('#new-status textarea').val(),
-        success: function(data) {
-            events.trigger("status:added", data.text);
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

Looking at `appendStatus` and `clearInput` in `NewStatusView`, we see
that these focus on two different DOM elements. This does not adhere to
the principles I outline in my blog post on
[a views responsibilities](https://open.bekk.no/a-views-responsibility/).
Let's pull a `StatusesView` out of `NewStatusView`. This is especially
simple now as we use events. If we had used a regular `success`
callback, this would be harder.

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
             events.trigger("status:added", data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
 
-    events.on("status:added", this.appendStatus, this);
     events.on("status:added", this.clearInput, this);
 
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
+    events.on("status:added", this.appendStatus, this);
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

As each view is responsible for one `el`, let's specify them when
instantiating the views:

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
             events.trigger("status:added", data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
+    this.el = $('#new-status');
 
     events.on("status:added", this.clearInput, this);
 
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
     events.on("status:added", this.appendStatus, this);
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
test because they depend on having the HTML present, e.g. to find
`$('#statuses')`. To remedy this we can pass in its DOM dependencies
when we instantiate a view.

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
             events.trigger("status:added", data.text);
         }
     });
 };
 
 var NewStatusView = function(options) {
     this.statuses = options.statuses;
-    this.el = $('#new-status');
+    this.el = options.el;
 
     events.on("status:added", this.clearInput, this);
 
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
 
     events.on("status:added", this.appendStatus, this);
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

# On 13
