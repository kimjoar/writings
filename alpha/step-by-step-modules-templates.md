Step by step to Backbone.js &mdash; Modules, Templates and Minification
=======================================================================

** THIS IS NOT READY YET! **

TODO

* Push app to Heroku, fix links
* Get feedback
* Chrome Developer Tools image of text plugin
* Create a StatusView for a single status in StatusesView? To show a
  view with no template, and with tagName set to `li`

In [my last step by step article][stepbystep] I took a piece of regular
jQuery-based JavaScript code and transformed it into idiomatic Backbone
using Models, Collections, Views and Events.  In this blog post I'll
build on the code, and step by step create modules using Require.js and
then show my currently preferred way of handling templates in
Backbone.js apps. We'll finish off with creating a production ready
version of the code, minified into a single JavaScript file.

Initial setup
-------------

This article starts off where we finished last time around. The app is
up and running [here][appinit], and here is the final JavaScript from
last time:

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

And this is the HTML:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Step by step</title>
    <meta charset="utf-8">
    <script src="vendor/jquery-1.8.3.js"></script>
    <script src="vendor/underscore-1.4.2.js"></script>
    <script src="vendor/backbone-0.9.2.js"></script>
    <script src="monologue.js"></script>
  </head>
  <body>
    <div id="new-status">
      <h2>New monolog</h2>
      <form>
        <textarea></textarea><br>
        <input type="submit" value="Post"/>
      </form>
    </div>

    <div id="statuses">
      <h2>Monologs</h2>
      <ul></ul>
    </div>
  </body>
</html>
```

Modules using Require.js
------------------------

Most of us have written 1000+ lines of JavaScript code in a single file.
For large projects this is a pain to work with, it's a pain to test, and
it's a pain to reuse and extend the code.

The code we have above looks good for now, but gradually the size and
complexity will increase, and suddenly the file is too long and
unwieldy. In this blog post we'll use Require.js to split the code into
several files. Require.js uses the Asynchronous Module Definition (AMD)
API for handling JavaScript modules, which you can read more about in 
[their documentation][whyamd].

So, let's start using Require.js. First of all we must include the
library and tell it what will be our main application entry point. In
the HTML this can be done as follows:

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>Step by step</title>
     <meta charset="utf-8">
-    <script src="vendor/jquery-1.8.3.js"></script>
-    <script src="vendor/underscore-1.4.2.js"></script>
-    <script src="vendor/backbone-0.9.2.js"></script>
-    <script src="monologue.js"></script>
+    <script data-main="monologue.js" src="vendor/require-2.1.2.js"></script>
   </head>
   <body>
     <div id="new-status">
       <h2>New monolog</h2>
       <form>
         <textarea></textarea><br>
         <input type="submit" value="Post"/>
       </form>
     </div>
 
     <div id="statuses">
       <h2>Monologs</h2>
       <ul></ul>
     </div>
   </body>
 </html>
```

Now we must wrap our JavaScript, `monologue.js`, in a little Require.js
setup:

```diff
+requirejs.config({
+    paths: {
+        'jquery': 'vendor/jquery-1.8.3'
+      , 'underscore': 'vendor/underscore-1.4.2'
+      , 'backbone': 'vendor/backbone-0.9.2'
+    },
+    shim: {
+        'backbone': {
+            deps: ['underscore', 'jquery'],
+            exports: 'Backbone'
+        },
+        'underscore': {
+            exports: '_'
+        }
+    }
+});
+
+require([
+    'jquery'
+  , 'backbone'
+], function($, Backbone) {
+
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
+
+});
```

Splitting out modules
---------------------

Now that we have our base setup and our app is still [up and
running][apprequirejs], we can start moving the separate parts out of
`monologue.js`. Lets start by creating a `modules/status` folder, then
we can start by moving the Status model into this folder:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery'
   , 'backbone'
+  , 'modules/status/status'
-], function($, Backbone) {
+], function($, Backbone, Status) {
-
-    var Status = Backbone.Model.extend({
-        url: '/status'
-    });
 
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
 
 });
```

As you can see above we include `modules/status/status.js`, so we copy
the Status model into this file:

```javascript
define(['backbone'], function(Backbone) {

    var Status = Backbone.Model.extend({
        url: '/status'
    });

    return Status;

});
```

Let's do the same with the `Statuses` collection.

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery'
   , 'backbone'
-  , 'modules/status/status'
+  , 'modules/status/statuses'
-], function($, Backbone, Status) {
+], function($, Backbone, Statuses) {
-
-    var Statuses = Backbone.Collection.extend({
-        model: Status
-    });
     
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
 
 });
```

As you can see, we no longer depend on the `Status` model in
`monologue.js` as it is only needed in the Statuses collection, so we no
longer include it. Our `modules/status/statuses.js`:

```javascript
define([
    'backbone',
    'modules/status/status'
], function(Backbone, Status) {

    var Statuses = Backbone.Collection.extend({
        model: Status
    });

    return Statuses;

});
```

We do the same with `NewStatusView`:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery' 
   , 'backbone'
   , 'modules/status/statuses'
   , 'modules/status/newStatusView'
-], function($, Backbone, Statuses) {
+], function($, Backbone, Statuses, NewStatusView) {
-
-    var NewStatusView = Backbone.View.extend({
-        events: {
-            "submit form": "addStatus"
-        },
-
-        initialize: function(options) {
-            this.collection.on("add", this.clearInput, this);
-        },
-
-        addStatus: function(e) {
-            e.preventDefault();
-
-            this.collection.create({ text: this.$('textarea').val() });
-        },
-
-        clearInput: function() {
-            this.$('textarea').val('');
-        }
-    });
     
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
 
 });
```

`modules/status/newStatusView.js`:

```javascript
define([
    'backbone'
], function(Backbone) {

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

    return NewStatusView;

});
```

And then StatusesView:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery' 
   , 'backbone'
   , 'modules/status/statuses'
   , 'modules/status/newStatusView'
+  , 'modules/status/statusesView'
-], function($, Backbone, Statuses, NewStatusView) {
+], function($, Backbone, Statuses, NewStatusView, StatusesView) {
-
-    var StatusesView = Backbone.View.extend({
-        initialize: function(options) {
-            this.collection.on("add", this.appendStatus, this);
-        },
-
-        appendStatus: function(status) {
-            this.$('ul').append('<li>' + status.escape("text") + '</li>');
-        }
-    });
 
     $(document).ready(function() {
         var statuses = new Statuses();
         new NewStatusView({ el: $('#new-status'), collection: statuses });
         new StatusesView({ el: $('#statuses'), collection: statuses });
     });
 
 });
```

`modules/status/statusesView.js`:

```javascript
define([
    'backbone'
], function(Backbone) {

    var StatusesView = Backbone.View.extend({
        initialize: function(options) {
            this.collection.on("add", this.appendStatus, this);
        },

        appendStatus: function(status) {
            this.$('ul').append('<li>' + status.escape("text") + '</li>');
        }
    });

    return StatusesView;

});
```

And now `monologue.js` looks quite good:

```javascript
requirejs.config({
    paths: {
        'jquery': 'vendor/jquery-1.8.3'
      , 'underscore': 'vendor/underscore-1.4.2'
      , 'backbone': 'vendor/backbone-0.9.2'
    },
    shim: {
        'backbone': {
            deps: ['underscore', 'jquery'],
            exports: 'Backbone'
        },
        'underscore': {
            exports: '_'
        }
    }
});

require([
    'jquery'
  , 'modules/status/statuses'
  , 'modules/status/newStatusView'
  , 'modules/status/statusesView'
], function($, Statuses, NewStatusView, StatusesView) {

    $(document).ready(function() {
        var statuses = new Statuses();
        new NewStatusView({ el: $('#new-status'), collection: statuses });
        new StatusesView({ el: $('#statuses'), collection: statuses });
    });

});
```

Pretty sweet. This file is now focused on kickstarting our application.
Outside of this file, none of the other files fetch anything directly
from the DOM. One of the primary benefits of this, is that it
significantly increases the testability of the code. I've written [a
little bit][responsibility] before about this.

Templates
---------

In our application the HTML is already present in `index.html`, but in
most applications that won't be the case. In most applications we render
HTML for example when some data is loaded, when we go to another page,
and so on.

Our first step is getting `NewStatusView` and `StatusesView` to render
into `#new-status` and `#statuses`. We'll start with removing all
`NewStatusView` related HTML from `index.html`:

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>Step by step &mdash; Templates in Backbone</title>
     <meta charset="utf-8">
     <script data-main="monologue.js" src="vendor/require-2.1.2.js"></script>
   </head>
   <body>
     <div id="new-status">
-      <h2>New monolog</h2>
-      <form>
-        <textarea></textarea><br>
-        <input type="submit" value="Post"/>
-      </form>
     </div>
 
     <div id="statuses">
       <h2>Monologs</h2>
       <ul></ul>
     </div>
   </body>
 </html>
```

Instead of it always being present in `index.html`, we'll move the HTML
into the view, adding it to the HTML when `render` is called on the
view. `newStatusView.js`:

```diff
 define([
     'backbone'
 ], function(Backbone) {
 
     var NewStatusView = Backbone.View.extend({
+        template: '<h2>New monolog</h2>' +
+                  '<form>' +
+                  '  <textarea></textarea><br>' +
+                  '  <input type="submit" value="Post"/>' +
+                  '</form>',
+
         events: {
             "submit form": "addStatus"
         },
 
         initialize: function(options) {
             this.collection.on("add", this.clearInput, this);
         },
 
+        render: function() {
+            this.$el.html(this.template);
+        },
+
         addStatus: function(e) {
             e.preventDefault();
 
             this.collection.create({ text: this.$('textarea').val() });
         },
 
         clearInput: function() {
             this.$('textarea').val('');
         }
     });
 
     return NewStatusView;
 
 });
```

To get the HTML into `index.html` we now have to call `render` on the
view in `monologue.js`:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery'
   , 'backbone'
   , 'modules/status/statuses'
   , 'modules/status/newStatusView'
   , 'modules/status/statusesView'
 ], function($, Backbone, Statuses, NewStatusView, StatusesView) {
 
     $(document).ready(function() {
         var statuses = new Statuses();
-        new NewStatusView({ el: $('#new-status'), collection: statuses });
+
+        var newStatusView = new NewStatusView({ el: $('#new-status'), collection: statuses });
+        newStatusView.render();
+
         new StatusesView({ el: $('#statuses'), collection: statuses });
     });
 
 });
```

We do the same with `StatusesView`:

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>Step by step &mdash; Templates in Backbone</title>
     <meta charset="utf-8">
     <script data-main="monologue.js" src="vendor/require-2.1.2.js"></script>
   </head>
   <body>
     <div id="new-status">
     </div>
 
     <div id="statuses">
-      <h2>Monologs</h2>
-      <ul></ul>
     </div>
   </body>
 </html>
```

`statusesView.js`:

```diff
 define([
     'backbone'
 ], function(Backbone) {
 
     var StatusesView = Backbone.View.extend({
+        template: '<h2>Monologs</h2>' +
+                  '<ul></ul>',
+
         initialize: function(options) {
             this.collection.on("add", this.appendStatus, this);
         },
 
+        render: function() {
+            this.$el.html(this.template);
+        },
+
         appendStatus: function(status) {
             this.$('ul').append('<li>' + status.escape("text") + '</li>');
         }
     });
 
     return StatusesView;
 
 });
```

And, again, call `render` in `monologue.js`:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery'
   , 'backbone'
   , 'modules/status/statuses'
   , 'modules/status/newStatusView'
   , 'modules/status/statusesView'
 ], function($, Backbone, Statuses, NewStatusView, StatusesView) {
 
     $(document).ready(function() {
         var statuses = new Statuses();
 
         var newStatusView = new NewStatusView({ el: $('#new-status'), collection: statuses });
         newStatusView.render();
 
-        new StatusesView({ el: $('#statuses'), collection: statuses });
+        var statusesView = new StatusesView({ el: $('#statuses'), collection: statuses });
+        statusesView.render();
     });
 
 });
```

We have now moved templates out of `index.html` and into our views.
This is a great first step, but we have one significant problem: what
happens when our templates grow. Having templates longer than a few
lines, i.e. most templates, will clutter the view.

Moving templates out of the views
---------------------------------

The solution, of course, is to move the templates out of the views and
into their own files. We'll do this using the Require.js [text
plugin][text]. We start by including the plugin in `monologue.js`:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3'
       , 'underscore': 'vendor/underscore-1.4.2'
       , 'backbone': 'vendor/backbone-0.9.2'
+      , 'text': 'vendor/text-2.0.3'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery',
     'backbone',
     'modules/status/statuses',
     'modules/status/newStatusView',
     'modules/status/statusesView'
 ], function($, Backbone, Statuses, NewStatusView, StatusesView) {
 
     $(document).ready(function() {
         var statuses = new Statuses();
 
         var newStatusView = new NewStatusView({ el: $('#new-status'), collection: statuses });
         newStatusView.render();
 
         var statusesView = new StatusesView({ el: $('#statuses'), collection: statuses });
         statusesView.render();
     });
 
 });
```

The plugin loads text resources which are defined as a dependency using
the `text!` prefix, e.g. `text!test.html` loads `test.html` and makes it
available as a string.

TODO: Chrome Developer Tools image of this

First, let's create a file for the `NewStatusView` template,
`modules/status/newStatusView.html`:

```html
<h2>New monolog</h2>
<form>
  <textarea></textarea><br>
  <input type="submit" value="Post"/>
</form>
```

And then, in `newStatusView.js`:

```diff
 define([
     'backbone'
+  , 'text!modules/status/newStatusView.html'
-], function(Backbone) {
+], function(Backbone, newStatusViewTemplate) {
 
     var NewStatusView = Backbone.View.extend({
-        template: '<h2>New monolog</h2>' +
-                  '<form>' +
-                  '  <textarea></textarea><br>' +
-                  '  <input type="submit" value="Post"/>' +
-                  '</form>',
+        template: newStatusViewTemplate,
 
         events: {
             "submit form": "addStatus"
         },
 
         initialize: function(options) {
             this.collection.on("add", this.clearInput, this);
         },
 
         render: function() {
             this.$el.html(this.template);
         },
 
         addStatus: function(e) {
             e.preventDefault();
 
             this.collection.create({ text: this.$('textarea').val() });
         },
 
         clearInput: function() {
             this.$('textarea').val('');
         }
     });
 
     return NewStatusView;
 
 });
```

Boyah &mdash; external templates which don't clutter the view at all.

Let's do the same with `StatusesView`:

```diff
 define([
     'backbone'
+  , 'text!modules/status/statuses.html'
-], function(Backbone) {
+], function(Backbone, statusesTemplate) {
 
     var StatusesView = Backbone.View.extend({
-        template: '<h2>Monologs</h2>' +
-                  '<ul></ul>',
+        template: statusesTemplate,
 
         initialize: function(options) {
             this.collection.on("add", this.appendStatus, this);
         },
 
         render: function() {
             this.$el.html(this.template);
         },
 
         appendStatus: function(status) {
             this.$('ul').append('<li>' + status.escape("text") + '</li>');
         }
     });
 
     return StatusesView;
 
 });
```

And the statuses template, `modules/status/statuses.html`:

```html
<h2>Monologs</h2>
<ul></ul>
```

Template engines
----------------

Usually a template contains both HTML and some logic, so you would
almost always end up using some form of template engine, such as
[Hogan.js][hogan] or [Handlebars.js][handlebars], to generate the final
HTML. Luckily, it is dead simple to include this in our current setup.
As an example, let's start by including Hogan.js in `monolog.js`:

```diff
 requirejs.config({
     paths: {
         'jquery': 'vendor/jquery-1.8.3',
         'underscore': 'vendor/underscore-1.4.2',
         'backbone': 'vendor/backbone-0.9.2',
-        'text': 'vendor/text-2.0.3'
+        'text': 'vendor/text-2.0.3',
+        'hogan': 'vendor/hogan-2.0.0'
     },
     shim: {
         'backbone': {
             deps: ['underscore', 'jquery'],
             exports: 'Backbone'
         },
         'underscore': {
             exports: '_'
         }
     }
 });
 
 require([
     'jquery',
     'backbone',
     'modules/status/statuses',
     'modules/status/newStatusView',
     'modules/status/statusesView'
 ], function($, Backbone, Statuses, NewStatusView, StatusesView) {
 
     $(document).ready(function() {
         var statuses = new Statuses();
 
         var newStatusView = new NewStatusView({ el: $('#new-status'), collection: statuses });
         newStatusView.render();
 
         var statusesView = new StatusesView({ el: $('#statuses'), collection: statuses });
         statusesView.render();
     });
 
 });
```

And now we can use Hogan.js on our template in for example
`NewStatusView`:

```diff
 define([
     'backbone'
   , 'text!modules/status/newStatusView.html'
+  , 'hogan'
-], function(Backbone, newStatusViewTemplate) {
+], function(Backbone, newStatusViewTemplate, hogan) {
 
     var NewStatusView = Backbone.View.extend({
-        template: newStatusViewTemplate,
+        template: hogan.compile(newStatusViewTemplate),
 
         events: {
             "submit form": "addStatus"
         },
 
         initialize: function(options) {
             this.collection.on("add", this.clearInput, this);
         },
 
         render: function() {
-            this.$el.html(this.template);
+            this.$el.html(this.template.render());
         },
 
         addStatus: function(e) {
             e.preventDefault();
 
             this.collection.create({ text: this.$('textarea').val() });
         },
 
         clearInput: function() {
             this.$('textarea').val('');
         }
     });
 
     return NewStatusView;
 
 });
```

As we are now using Hogan.js, which implements the [Mustache
spec][mustachespec], we should rename `newStatusView.html` to
`newStatusView.mustache`. Now we can start passing data to the view and
start adding some lovely Mustache to our view.

Going even further with templates
---------------------------------

Our handling of templates have come a long way, but there are still a
possibility we haven't explored: using template engine plugins for
Require.js, e.g. this [Hogan.js plugin][hgn]. This has a couple of
benefits:

* We don't need to `compile` the templates ourselves, we receive a
  compiled template ready for use. This cleans up our code slightly.
* More importantly, however, we get a sweet bonus when optimizing our
  code for production: the templates are compiled into JavaScript and
  becomes a part of the single minified JavaScript for the app. Thus, we
  no longer have to compile the templates at run-time in the browser.

It should be quite simple to include the plugin, so I'll not do it here.
Just follow the steps used when including the text plugin, just be sure
that you use the correct plugin prefix (e.g. `hgn` for the Hogan.js
plugin).

Getting ready for production
----------------------------

As we no longer have only one JavaScript file, we need to concatenate
our files when preparing our code for production. Additionally, we want
this process to inline our templates into the minfified JavaScript file
so we don't have to keep fetching them dynamically in production. When
using Require.js the natural choice is using its minifier, [r.js][rjs].

For Require.js we need to create a config file for the minification,
`config/buildconfig.js`:

```javascript
({
    // all modules are located relative to this path
    baseUrl: '../public',

    // name of file which kickstarts the application, aka the main file
    name: 'monologue',

    // use the main JS file configuration so we don't need to duplicate the values
    mainConfigFile: '../public/monologue.js',

    // additionally include Require.js itself as a dependency
    include: ['vendor/require-2.1.2.js'],

    // name the optimized file
    out: '../build/monologue.js',

    // inlines the text for any text! dependencies
    inlineText: true,

    // we don't need the text plugin in production, as it is inlined
    stubModules: ['text'],

    // keep 'em comments
    preserveLicenseComments: true
})
```

There is quite a lot of options for the build config, so I recommend
checking out this [file][buildconfig], which contains all the options
and a whole lot of documentation.

Run the build using Node.js (remember to run from the project root):

```sh
$ node public/vendor/r.js -o config/buildconfig.js
```

Or using Java (still, from the project root):

```sh
$ java -classpath lib/rhino/js.jar:lib/closure/compiler.jar org.mozilla.javascript.tools.shell.Main public/vendor/r.js -o config/buildconfig.js
```

And now we should see something similar to:

```
Tracing dependencies for: monologue
Uglifying file: /Users/kimjoar/dev/monologue/build/monologue.js

/Users/kimjoar/dev/monologue/build/monologue.js
----------------
/Users/kimjoar/dev/monologue/public/vendor/require-2.1.2.js
/Users/kimjoar/dev/monologue/public/vendor/jquery-1.8.3.js
/Users/kimjoar/dev/monologue/public/vendor/underscore-1.4.2.js
/Users/kimjoar/dev/monologue/public/vendor/backbone-0.9.2.js
/Users/kimjoar/dev/monologue/public/modules/status/status.js
/Users/kimjoar/dev/monologue/public/modules/status/statuses.js
/Users/kimjoar/dev/monologue/public/vendor/text-2.0.3.js
text!modules/status/newStatusView.html
/Users/kimjoar/dev/monologue/public/vendor/hogan-2.0.0.js
/Users/kimjoar/dev/monologue/public/modules/status/newStatusView.js
text!modules/status/statusesTemplate.html
/Users/kimjoar/dev/monologue/public/modules/status/statusesView.js
/Users/kimjoar/dev/monologue/public/monologue.js
```

And we'll have a minified JavaScript file in `build/monologue.js`. If
you don't want to optimize the code, check out the [`optimize`
setting][optimize].

Making a production ready `index.html` is now as simple as using the
minified JavaScript file:

```diff
 <!DOCTYPE html>
 <html>
   <head>
     <title>Step by step &mdash; Templates in Backbone</title>
     <meta charset="utf-8">
-    <script data-main="monologue.js" src="vendor/require-2.1.2.js"></script>
+    <script src="monologue.js"></script>
   </head>
   <body>
     <div id="new-status">
     </div>
 
     <div id="statuses">
     </div>
   </body>
 </html>
```

As you can see we only need to include `monologue.js` and nothing else.
Now Require.js will fetch our JavaScript files as they are needed in
development, while we have a single file which contains everything in
production.

Finishing up
------------

In this blog post we have taken some steps further from my initial step
by step introduction to Backbone.js, and introduced Require.js and
template handling into the mix. There are still many things that need to
be done when setting up a large-scale JavaScript application, but we
have taken some significant steps further. We have also created a
production-ready version of our app.

If you want a setup similar to this in a Java-only world, you can find a
lot of inspiration in [this setup][js-java-setup].

[responsibility]: http://open.bekk.no/a-views-responsibility/
[stepbystep]: https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md
[appinit]: heroku
[apprequirejs]: heroku
[amd]: https://github.com/amdjs/amdjs-api/wiki/AMD
[whyamd]: http://requirejs.org/docs/whyamd.html
[text]: https://github.com/requirejs/text
[hogan]: http://twitter.github.com/hogan.js/
[handlebars]: http://handlebarsjs.com/
[rjs]: https://github.com/jrburke/r.js/
[mustachespec]: http://mustache.github.com/mustache.5.html
[hgn]: https://github.com/millermedeiros/requirejs-hogan-plugin
[buildconfig]: https://github.com/jrburke/r.js/blob/master/build/example.build.js
[optimize]: https://github.com/jrburke/r.js/blob/c1be5af39ee8a0c0bdb74ce1df4ffe35277b2f49/build/example.build.js#L80-L91
[js-java-setup]: https://github.com/kjbekkelund/js-java-setup
