Step by step to Backbone.js &mdash; Handling templates
======================================================

After first [transforming a piece of regular jQuery-based JavaScript
into Backbone][stepbystep] and then [adding Require.js to the
mix][stepmodules], this blog post will show my preferred way of handling
templates when using Backbone.js. Additionally, we'll finish off with
creating a production ready version of the code, minifying even the
templates into the single production-ready JavaScript file.

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
view. We'll start in the simplest way possible.

`newStatusView.js`:

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
happens when our templates grow. As soon as the template grows past a
couple of lines our current solution will clutter the view.

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
Require.js, such as this [Hogan.js plugin][hgn]. This has a couple of
benefits:

* We don't need to `compile` the templates ourselves, we receive a
  compiled template ready for use. This cleans up our code slightly.
* More importantly, however, we get a sweet bonus when optimizing our
  code for production: the templates are compiled into JavaScript and
  becomes a part of the single minified JavaScript for the app. So now
  we no longer have to compile the templates at run-time in the browser.

It should be quite simple to include the plugin, so I'll not do it here.
Just follow the steps used when including the text plugin, just be sure
that you use the correct plugin prefix (e.g. `hgn` for the Hogan.js
plugin).

Getting ready for production
----------------------------

Additionally, we want this process to inline our templates into the
minfified JavaScript file so we don't have to keep fetching them
dynamically in production. 

As explained in the [previous blog post][stepmodules], we use
Require.js's optimizer, [r.js][rjs], to prepare our code for production.
We run our build with Node.js:

```bash
$ node public/vendor/r.js -o config/buildconfig.js
```

And the result:

```
Tracing dependencies for: monologue
Uglifying file: /Users/kjbekkelund/dev/monologue/build/monologue.js

/Users/kjbekkelund/dev/monologue/build/monologue.js
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

And we'll have a minified JavaScript file in `build/monologue.js`.

[responsibility]: http://open.bekk.no/a-views-responsibility/
[stepbystep]: https://github.com/kjbekkelund/writings/blob/master/published/understanding-backbone.md
[appinit]: http://monologue-2.herokuapp.com/
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
[shim]: http://requirejs.org/docs/api.html#config-shim
[monologue]: https://github.com/kjbekkelund/monologue/tree/modules-start
[monologuedone]: https://github.com/kjbekkelund/monologue/tree/modules-end
[stepmodules]: https://github.com/kjbekkelund/writings/blob/master/published/step-by-step-modules.md
