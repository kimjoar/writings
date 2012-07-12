Understanding Backbone.js
=========================

I've seen several people struggle the first few times they've tried to
use Backbone.js. In this blog post I will gradually refactor a bit of
code from the way I see most JavaScript code written into proper
Backbone.js code, including the use of models, collections, views and
events.

First, let's start with the code we're going to work with. This example
is based on an example in Brandon Keepers great talk on JavaScript at ….
But his focus was on testing, while mine is explaining the Backbone.js
core abstractions. So, the code:

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

Basically, we wait for the DOM to be ready, set a submit listener, and
when the form is submitted we send the input to the server, append the
response to the list of statuses and reset the input.

We'll start breaking the coupling between Ajax and DOM handling by
creating an `addStatus` method:

```javascript
function addStatus(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

jQuery(function() {
    $('#new-status').submit(function(e) {
        e.preventDefault();

        addStatus({
          text: $(this).find('textarea').val(),
          success: function(data) {
            $('#statuses').append('<li>' + data.text + '</li>');
            $(this).find('textarea').val('');
          }
        });
    });
});
```

We then use the
[constructor pattern with prototypes](http://addyosmani.com/resources/essentialjsdesignpatterns/book/#constructorpatternjavascript)
to introduce a Statuses "class":

```javascript
var Statuses = function() {};
Statuses.prototype.add = function(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

jQuery(function() {
    var statuses = new Statuses();

    $('#new-status').submit(function(e) {
        e.preventDefault();

        statuses.add({
          text: $(this).find('textarea').val(),
          success: function(data) {
            $('#statuses').append('<li>' + data.text + '</li>');
            $(this).find('textarea').val('');
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

```javascript
var Statuses = function() {};
Statuses.prototype.add = function(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

var NewStatusView = function(options) {
    var statuses = options.statuses;

    $('#new-status').submit(function(e) {
        e.preventDefault();

        statuses.add({
          text: $(this).find('textarea').val(),
          success: function(data) {
            $('#statuses').append('<li>' + data.text + '</li>');
            $(this).find('textarea').val('');
          }
        });
    });
}

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ statuses: statuses });
});
```

Now, our jQuery ready handling is much cleaner than when we started.
To clean up the `NewStatusView` we can start by splitting the handling
of the submit into its own `addStatus` method.

```javascript
var Statuses = function() {};
Statuses.prototype.add = function(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;

  var add = $.proxy(this.addStatus, this);
  $('#new-status').submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add({
    text: $('#new-status').find('textarea').val(),
    success: function(data) {
      $('#statuses').append('<li>' + data.text + '</li>');
      $('#new-status').find('textarea').val('');
    }
  });
}

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ statuses: statuses });
});
```

To get this working we need to use `$.proxy` for `this` to mean the
right thing when calling `addStatus`, as we need to reach
`this.statuses`. At the same time we change `$(this).find('textarea')`
into `$('#new-status').find('textarea')` as `this` now means something
else.

Let's make the `success` callback call methods instead of working
directly on the DOM:

```javascript
var Statuses = function() {};
Statuses.prototype.add = function(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;

  var add = $.proxy(this.addStatus, this);
  $('#new-status').submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add({
    text: $('#new-status').find('textarea').val(),
    success: function(data) {
      this.appendStatus(data.text);
      this.clearInput();
    }
  });
}
NewStatusView.prototype.appendStatus = function(text) {
    $('#statuses').append('<li>' + data.text + '</li>');
};
NewStatusView.prototype.clearInput = function() {
    $('#new-status').find('textarea').val('');
}

jQuery(function() {
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

```javascript
var events = _.clone(Backbone.Events);

var Statuses = function() {};
Statuses.prototype.add = function(options) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: options.text },
    success: options.success
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;

  events.on('status:added', this.appendStatus, this);
  events.on('status:added', this.clearInput, this);

  var add = $.proxy(this.addStatus, this);
  $('#new-status').submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add({
    text: $('#new-status').find('textarea').val(),
    success: function(data) {
      events.trigger('status:added', data.text);
    }
  });
}
NewStatusView.prototype.appendStatus = function(text) {
    $('#statuses').append('<li>' + data.text + '</li>');
};
NewStatusView.prototype.clearInput = function() {
    $('#new-status').find('textarea').val('');
}

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ statuses: statuses });
});
```

Now, we don't have special handling in the `success` callback, so we can
move the triggering of the event into the `add` method on `Statuses`:

```javascript
var events = _.clone(Backbone.Events);

var Statuses = function() {};
Statuses.prototype.add = function(text) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: text },
    success: function(data) {
      events.trigger('status:added', data.text);
    }
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;

  events.on('status:added', this.appendStatus, this);
  events.on('status:added', this.clearInput, this);

  var add = $.proxy(this.addStatus, this);
  $('#new-status').submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add($('#new-status').find('textarea').val());
}
NewStatusView.prototype.appendStatus = function(text) {
    $('#statuses').append('<li>' + text + '</li>');
};
NewStatusView.prototype.clearInput = function() {
    $('#new-status').find('textarea').val('');
}

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ statuses: statuses });
});
```

Looking at `appendStatus` and `clearInput` in `NewStatusView`, we see
that these focus on two different DOM elements. This does not adhere to
the principles I outline in my blog post on
[a views responsibilities]().
Let's pull a `StatusesView` out of `NewStatusView`. This is especially
simple now as we use events. If we had used a regular `success`
callback, this would be harder.

```javascript
var events = _.clone(Backbone.Events);

var Statuses = function() {};
Statuses.prototype.add = function(text) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: text },
    success: function(data) {
      events.trigger('status:added', data.text);
    }
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;

  events.on('status:added', this.clearInput, this);

  var add = $.proxy(this.addStatus, this);
  $('#new-status').submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add($('#new-status').find('textarea').val());
}
NewStatusView.prototype.clearInput = function() {
    $('#new-status').find('textarea').val('');
}

var StatusesView = function(options) {
  this.statuses = options.statuses;

  events.on('status:added', this.appendStatus, this);
}
StatusesView.prototype.appendStatus = function(text) {
    $('#statuses').append('<li>' + data.text + '</li>');
};

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ statuses: statuses });
    new StatusesView({ statuses: statuses });
});
```

Our views, `NewStatusView` and `StatusesView` is still difficult to
test because they depend on having the HTML present, e.g. to find
`$('#statuses')`. To remedy this we can pass in its DOM dependencies
when we instantiate a view. We will call these dependencies `el`, and
each view is responsible for one such dependency, i.e. one HTML element.

```javascript
var events = _.clone(Backbone.Events);

var Statuses = function() {};
Statuses.prototype.add = function(text) {
  $.ajax({
    url: '/status',
    type: 'POST',
    dataType: 'json',
    data: { text: text },
    success: function(data) {
      events.trigger('status:added', data.text);
    }
  });
}

var NewStatusView = function(options) {
  this.statuses = options.statuses;
  this.el = options.el;

  events.on('status:added', this.clearInput, this);

  var add = $.proxy(this.addStatus, this);
  this.el.submit(add);
}
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add(this.el.find('textarea').val());
}
NewStatusView.prototype.clearInput = function() {
    this.el.find('textarea').val('');
}

var StatusesView = function(options) {
  this.statuses = options.statuses;
  this.el = options.el;

  events.on('status:added', this.appendStatus, this);
}
StatusesView.prototype.appendStatus = function(text) {
    this.el.append('<li>' + data.text + '</li>');
};

jQuery(function() {
    var statuses = new Statuses();
    new NewStatusView({ el: $('#new-status'), statuses: statuses });
    new StatusesView({ el: $('#statuses'), statuses: statuses });
});
```

Then:

```javascript
var events = _.clone(Backbone.Events);

var Statuses = function() {};
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
}

var NewStatusView = Backbone.View.extend({
  initialize: function(options) {
    this.statuses = options.statuses;
    this.el = options.el;

    events.on("status:added", this.clearInput, this);

    var add = $.proxy(this.addStatus, this);
    this.el.submit(add);
  }
});
NewStatusView.prototype.addStatus = function(e) {
  e.preventDefault();

  this.statuses.add(this.el.find('textarea').val());
}
NewStatusView.prototype.clearInput = function() {
  this.el.find("textarea").val("");
}

var StatusesView = function(options) {
  this.statuses = options.statuses;
  this.el = options.el;

  events.on("status:added", this.appendStatus, this);
}
StatusesView.prototype.appendStatus = function(text) {
  this.el.append('<li>' + data.text + '</li>');
};

jQuery(function() {
  var statuses = new Statuses();
  new NewStatusView({ el: $("#new-status"), statuses: statuses });
  new StatusesView({ el: $("#statuses"), statuses: statuses });
});
```
