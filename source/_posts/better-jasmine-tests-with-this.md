title: "Better Jasmine Tests With `this`"
date: 2015-03-28 23:00:20
tags: javascript, testing
---

<small>Republished from [this original gist](https://gist.githubusercontent.com/traviskaufman/11131303/raw/cc0ebcd1911317ae01c4fef4e9175c7da94bc1c8/jasmine-this-vars.md)</small>

On the Refinery29 Mobile Web Team, codenamed "Bicycle", all of our unit tests are written using [Jasmine](http://jasmine.github.io/2.0/introduction.html), an awesome BDD library written by [Pivotal Labs](http://pivotallabs.com/). We recently switched how we set up data for tests from declaring and assigning to closures, to assigning properties to each test case's `this` object, and we've seen some awesome benefits from doing such.

## The old way

Up until recently, a typical unit test for us looked something like this:

```javascript
describe('views.Card', function() {
  'use strict';

  var model, view;

  beforeEach(function() {
    model = {};
    view = new CardView(model);
  });

  describe('.render', function() {
    beforeEach(function() {
      model.title = 'An Article';
      view.render();
    });

    it('creates a "cardTitle" h3 element set to the model\'s title', function() {
      expect(view.$el.find('.cardTitle')).toContainText(model.title);
    });

    describe('when the model card type is "author_card"', function() {
      beforeEach(function() {
        model.type = 'author_card';
        view.render();
      });

      it('adds an "authorCard" class to its $el', function() {
        expect(view.$el).toHaveClass('authorCard');
      });
    });
  });

  // ...
});
```

We set up state by declaring variables within a `describe` block's lexical scope. We then assign/modify them as necessary in subsequent `beforeEach`/`afterEach` statements.

We noticed, however, that as our tests became more complex, this pattern became increasingly difficult. We found ourselves jumping around spec files, trying to find out where a given variable was initially defined. We also ran into subtle bugs due to clobbering variables with common names (i.e. `model`, `view`) within a given scope, failing to realize they had _already_ been defined. Furthermore, our declaration statements in `describe` blocks started looking something like

```javascript
var firstVar, secondVar, thirdVar, fourthVar, fifthVar, ..., nthVar
```

Which was ugly and hard to parse. Finally, we would sometimes run into flaky tests due to "leaks" - test-specific variables that were not properly cleaned up after each case.

## The new, better way
For every test (and their `beforeEach`/`afterEach` hooks), jasmine sets the receiver of each function to an [initially empty object](http://jasmine.github.io/2.0/introduction.html#section-The_<code>this</code>_keyword). This object, which is called `userContext` within Jasmine's source code, can have properties assigned to it, and gets blown away at the end of each test. In an attempt to address the issues we were having, we recently switched over to assigning variables to this object, rather than declaring them within `describe` and then assigning them. So our original code above now looked something like this:

```javascript
describe('views.Card', function() {
  'use strict';

  beforeEach(function() {
    this.model = {};
    this.view = new CardView(this.model);
  });

  describe('.render', function() {
    beforeEach(function() {
      this.model.title = 'An Article';
      this.view.render();
    });

    it('creates a "cardTitle" h3 element set to the model\'s title', function() {
      expect(this.view.$el.find('.cardTitle')).toContainText(this.model.title);
    });

    describe('when the model card type is "author_card"', function() {
      beforeEach(function() {
        this.model.type = 'author_card';
        this.view.render();
      });

      it('adds an "authorCard" class to its $el', function() {
        expect(this.view.$el).toHaveClass('authorCard');
      });
    });
  });

  // ...
});
```

## Why the new way rocks

Switching over to this pattern has yielded a significant amount of benefits for us, including:

### No more global leaks
Because jasmine instantiates a new `userContext` object for each test case, we didn't have to worry about test pollution any more. This helped ensure isolation between our tests, making them a lot more reliable.

### Clear meaning
Every time we see a `this.<PROP>` reference in our tests, we know that it refers to data set up in a `beforeEach`/`afterEach` statement. That, coupled with removing exhaustive var declarations in `describe` blocks, have made even our largest tests clear and understandable.

### Improved code reuse via dynamic invocation.
One of the best things about javascript is its facilities for [dynamic invocation](http://dailyjs.com/2012/10/26/di1/). Because we now use the `userContext` object to store all of our variables, and jasmine dynamically invokes each test function and hook with this context, we could begin to refactor our boilerplate code into helper functions that could be reused across specs.

Consider the initial beforeEach in our test case above:
```javascript
beforeEach(function() {
  this.model = {};
  this.view = new CardView(this.model);
});
```

Chances are we're going to want something like this in most of our view specs: a blank model and a view instantiated with the model as its argument. We can turn this logic into a reusable function like so:

```javascript
// spec_helper.js
beforeEach(function() {
  this.setupViewModel = function setupViewModel(ViewCtor) {
    this.model = {};
    this.view = new ViewCtor(this.model);
  };
});
```

Now, in any of our view tests, we could just write the following:

```javascript
beforeEach(function() {
  this.setupViewModel(SomeView);
});
```

And we'd be good to go!

### Reduced Code Duplication via Lazy Evaluation
This is where things get really awesome.

It's unfortunate in our original spec that we have to call `render()` twice. We call it once in the top-level `beforeEach`, and then again when we change the model's state in the nested `describe()`. We could call render _directly_ within the tests, but we'd rather keep all business logic within `beforeEach` statements, making only assertions within tests themselves. RSpec has this really nice [let() helper method](https://www.relishapp.com/rspec/rspec-core/v/2-6/docs/helper-methods/let-and-let) which will lazily evaluate a variable. This way even if a variable relies on some context, it need not have to be declared more than once.

Thankfully, because we're now assigning our variables to `this`, we can take advantage of [ES5 accessor properties](http://blog.kitsonkelly.com/2013/02/es5-accessor-properties/) to implement this lazy evaluation in our javascript specs!! Take a look at how our new render test cases would look when leveraging this:

```javascript
describe('.render', function() {
  beforeEach(function() {
    var _renderedView;
    this.model.title = 'An Article';
    // When we access this property, it will call `render()` on `this.view` and give
    // us a fresh, updated copy per test. No code duplication FTW!!
    Object.defineProperty(this, 'renderedView', {
      get: function() {
        if (!_renderedView) {
          _renderedView = this.view.render();
        }
        return _renderedView;
      },
      set: function() {}, //NOP
      enumerable: true,
      configurable: true
    });
  });

  it('creates a "cardTitle" h3 element set to the model\'s title', function() {
    expect(this.renderedView.$el.find('.cardTitle')).toContainText(this.model.title);
  });

  describe('when the model card type is "author_card"', function() {
    beforeEach(function() {
      this.model.type = 'author_card';
      // No need to re-render the view here!
    });

    it('adds an "authorCard" class to its $el', function() {
      expect(this.renderedView.$el).toHaveClass('authorCard');
    });
  });
});
```

Taking this a step further, we can genericize this within a spec helper file:

```javascript
// spec_helper.js
beforeEach(function() {
  // need to use "_" suffix since 'let' is a token in ES6
  this.let_ = function(propName, getter) {
    var _lazy;

    Object.defineProperty(this, propName, {
      get: function() {
        if (!_lazy) {
          _lazy = getter.call(this);
        }

        return _lazy;
      },
      set: function() {},
      enumerable: true,
      configurable: true
    });
  };
});
```

Now in your specs:

```javascript
describe('.render', function() {
  beforeEach(function() {
    this.let_('renderedView', function() {
      return this.view.render();
    });
    this.model.title = 'An Article';
  });

  // ...
});
```

Pretty slick if you ask us. However, it is important to [_be judicious_ when doing things such as this](http://robots.thoughtbot.com/lets-not). The method above uses a fair amount of black magic, and can lead to confusion if your team members aren't that familiar with newer ES5 concepts. However, taking advantage of these mechanisms in a smart manner can lead to a lot of reduced code duplication and therefore much higher test maintainability. We'd consider that a win for any team!
