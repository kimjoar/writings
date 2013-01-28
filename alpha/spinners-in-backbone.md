Handling spinners in Backbone.js
================================

THIS IS STILL UNDER DEVELOPMENT.

In every application that relies heavily on Ajax to fetch and post
information, spinners are needed to indicate progress, and for that
matter, successful completion. Here I'll show how I usually end up
solving this problem when using Backbone.js.

Let's start with an example view:

```javascript
var UserView = Backbone.View.extend({

  initialize: function() {
    this.collection.on("reset", this.render, this);
  },

  render: function() {
    // render template to `el`

    return this;
  }

});
```
