The example
-----------

Throughout this blog post we'll use a simple library application as an
example. In this app a user can view a list of books:

![The library app](https://github.com/sveinung/writings/raw/gh-pages/pageobjects/img/1-library.png?raw=true)

He can click "add book" to get a new view for adding a book:

![The library app](https://github.com/sveinung/writings/raw/gh-pages/pageobjects/img/2-add-book-view.png?raw=true)

Then he can fill in some info about the book:

![The library app](https://github.com/sveinung/writings/raw/gh-pages/pageobjects/img/3-adding-a-book.png?raw=true)

And then the book appears in the list:

![The library app](https://github.com/sveinung/writings/raw/gh-pages/pageobjects/img/4-book-added.png?raw=true)

In this blog post we try to write tests for `AddBookView`, which is the
view that is opened when clicking "Add book" and closed when saving the
new book. Basically, we want to test that adding new books works as
intended. And specifically that means that we want to:

1. Insert the name of the author
2. Insert the title of the book
3. Choose a genre from the drop-down
4. Press "Confirm"
5. Ensure that the book is saved

A typical test for this Backbone view using
[Jasmine](https://github.com/pivotal/jasmine/) might look something like
this: (we've added some comments to clarify the intention)

```javascript
it('saves the book', function() {
    //  Create some genres for the drop-down
    var genres = new Genres([
        { name: "Crime novel" },
        { name: "Picaresco" }
    ]);

    //  The book we are going to save
    var book = new Book();

    //  The view we are testing
    var addBookView = new AddBookView({
        // we pass in an in-memory jQuery object so the view does not
        // end up working directly with the DOM
        el: $('<div></div>'),
        genres: genres,
        book: book
    });
    addBookView.render();

    //  Set the author field
    addBookView.$(".author-input").val("Miguel de Cervantes Saavedra");

    //  Set the title field
    addBookView.$(".title-input").val("Don Quixote");

    //  Choose a genre for the book
    var dropdown = addBookView.$(".genres-dropdown");
    dropdown.find(".dropdown-trigger").click();
    dropdown.find("a[data-value='Picaresco']").click();

    // We need to fake Ajax responses, so that we can see that the
    // correct request is sent
    var server = sinon.fakeServer.create();

    //  Save the book
    this.addBookView.$(".submit-button").click();

    //  Find the body of the last Ajax request
    var requestBody = server.queue[0].requestBody;

    // Respond to the Ajax request and restore XMLHttpRequest
    server.respond();
    server.restore();

    //  Check that we actually saved what we expect to have saved
    expect(JSON.parse(requestBody)).toEqual({
        author: "Miguel de Cervantes Saavedra",
        title: "Don Quixote",
        genre: "Picaresco"
    });
});
```

This a _lot_ of code, especially considering the functionality we're
testing is quite simple. Also, it's not easy to see precisely what is
tested here without reading all the code. Actually, in this test about a
fourth of the lines are dedicated to setting up the view &mdash; much of
which isn't even relevant for the functionality we are testing.

Clean up view creation
----------------------

There are several ways of cleaning up the setup in this test. When we
first started writing tests for our JavaScript we usually went with a
`beforeEach` for this type of setup. However, there are significant
problems with this approach.

First of all, when we have many tests in a single file it's often
difficult to find a setup that works for _all the tests_. Also, we often
needed to jump to the `beforeEach` to see the current state of the code.
This could be a hassle, especially when there was a lot of setup in the
`beforeEach`.

Additionally, it's also possible to have several `describe`s inside each
other in Jasmine, and therefore potentially several `beforeEach`s who
all work on setting up the state. In our experience it could at times be
quite hard to understand the state of the object we were testing.

Now we prefer to use creation methods instead. So, for us the first step
towards a cleaner test is to move view creation into a helper:

```diff
 it('saves the book', function() {
-    var genres = new Genres([
-        { name: "Crime novel" },
-        { name: "Picaresco" }
-    ]);
-
-    var book = new Book();
-
-    var addBookView = new AddBookView({
-        el: $('<div></div>'),
-        genres: genres,
-        book: book
-    });

+    var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

     //  The rest of the test
 });
```

The result:

```javascript
it('saves the book', function() {
    var addBookView = createAddBookView({ genres: ["Picaresco"] });
    addBookView.render();

    //  The rest of the test
});

function createAddBookView(options) {
    options = options || {};

    var genres = [];
    if (options.genres) {
        genres = _.map(options.genres, function(genre) {
            return { name: genre }
        });
    }

    return new AddBookView({
        el: $('<div></div>'),
        genres: new Genres(genres),
        book: new Book()
    });
}
```

Now our setup is a bit cleaner and can easily be reused across tests. We
can also move logic into the creation helper that can help us clarify
intent in our tests. For example, if we need genres in our tests we can
write:

```javascript
createAddBookView({ genres: ["Picaresco", "Crime novel"] })
```

But when we don't need one, we can use the default instead:

```javascript
createAddBookView()
```

In our experience this can be a very powerful technique for both
revealing intent and writing less brittle tests.

Hiding access to the markup
---------------------------

Althought we have improved the test setup, there are still problems with
our test. Writing a lot of jQuery selectors and triggering events on DOM
objects can easily make our tests brittle in the long run.

For example, if we remove the `.author-input` field and add an
`.author-firstname-input` field and an `.author-lastname-input` field
instead, we often have to change several tests. A better way to handle
this problem &ndash; and write more robust tests at the same time
&mdash; is to wrap the low-level jQuery code.

So, let's go back to our `AddBookView` test:

```javascript
it('saves the book', function() {
    var addBookView = createAddBookView({ genres: ["Picaresco"] });
    addBookView.render();

    //  Set the author field
    addBookView.$(".author-input").
        val("Miguel de Cervantes Saavedra").
        change();

    //  Set the title field
    addBookView.$(".title-input").
        val("Don Quixote").
        change();

    //  Choose a genre for the book
    var dropdown = addBookView.$(".genres-dropdown");
    dropdown.find(".dropdown-trigger").click();
    dropdown.find("a[data-value='Picaresco']").click();

    //  The rest of the test
});
```

Instead of calling `$(".author-input")`, `$(".title-input")`, and
`$(".genres-dropdown")` to set the _author_, _title_ and _genre_
respectively, we could do something like this:

```diff
 it('saves the book', function() {
     var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

-     addBookView.$(".author-input").
-         val("Miguel de Cervantes Saavedra").
-         change();
-
-     addBookView.$(".title-input").
-         val("Don Quixote").
-         change();
-
-    var dropdown = addBookView.$(".genres-dropdown");
-    dropdown.find(".dropdown-trigger").click();
-    dropdown.find("a[data-value='Picaresco']").click();

+    addBookViewPageObject(addBookView.$el).
+        author("Miguel de Cervantes Saavedra").
+        title("Don Quixote").
+        genre("Picaresco");

     //  The rest of the test
 });
```

Here we have created a higher-level helper, which we call
`addBookViewPageObject`, which wraps all the jQuery details. Because of
this helper it's easier to update the view and still have all the tests
running &mdash; we only need to update the jQuery selector in one place.
Additionally, as we are now working at a higher level of abstraction in
our tests it's often easier to understand the intent. (The more you
struggle with hundreds of tests in highly interactive JavaScript
applications the more you end up loving tests that are easy to
understand and maintain.)

This is how we wrap jQuery:

```diff
+function addBookViewPageObject($el) {
+    return {
+        author: function(author) {
+            $el.find(".author-input").
+                val(author).
+                change();
+            return this;
+        },
+        title: function(title) {
+            $el.find(".title-input").
+                val(title).
+                change();
+            return this;
+        },
+        genre: function(genre) {
+            var dropdown = $el.find(".genres-dropdown");
+            dropdown.find(".dropdown-trigger").click();
+            dropdown.find("a[data-value='" + genre + "']").click();
+            return this;
+        }
+    };
+};
```

We have chosen to call this abstraction of the DOM a _Page Object_. A
similar idea is [documented in the Selenium wiki](https://code.google.com/p/selenium/wiki/PageObjects)
as follows:

> PageObjects can be thought of as facing in two directions
> simultaneously. Facing towards the developer of a test, they represent
> the services offered by a particular page. Facing away from the
> developer, they should be the only thing that has a deep knowledge of
> the structure of the HTML of a page.

With this abstraction we have cleaned up our test quite a bit:

```javascript
 it('saves the book', function() {
     var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

     addBookViewPageObject(addBookView.$el).
         author("Miguel de Cervantes Saavedra").
         title("Don Quixote").
         genre("Picaresco");

     //  The rest of the test
 });
```

Now, for the next step in our test we need to handle the genre. However,
choosing a genre is actually implemented as a `DropDownView`, so we
don't want the `AddBookView` to have too much knowledge about the
implementation. That only ends up causing problems in the long run, as
we push knowledge about drop-down specific functionality into other
tests. Therefore we have implemented a `dropDownViewPageObject` that
deals with opening the drop-down and choosing options.

Using the `dropDownViewPageObject` the test could look like this:

```diff
 function addBookViewPageObject($el) {
     return {
         author: function(author) {
             $el.find(".author-input").
                 val(author).
                 change();
             return this;
         },
         title: function(title) {
             $el.find(".title-input").
                 val(title).
                 change();
             return this;
         },
         genre: function(genre) {
-            var dropdown = $el.find(".genres-dropdown");
-            dropdown.find(".dropdown-trigger").click();
-            dropdown.find("a[data-value='" + genre + "']").click();

+            dropDownViewPageObject($el.find(".genres-dropdown")).
+                openMenu().
+                chooseOption(genre);
             return this;
         }
     };
 };
```

The `dropDownViewPageObject` is just a simple helper:

```diff
+function dropDownViewPageObject($el) {
+    return {
+        openMenu: function() {
+            $el.find(".dropdown-trigger").click();
+            return this;
+        },
+        chooseOption: function(option) {
+            $el.find(".dropdown-menu a[data-value='" + option + "']").click();
+            return this;
+        }
+    }
+}
```

After applying both the creation helper and the page object, our test
has improved considerably:

```javascript
 it('saves the book', function() {
     var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

     addBookViewPageObject(addBookView.$el).
         author("Miguel de Cervantes Saavedra").
         title("Don Quixote").
         genre("Picaresco");

     //  Fake ajax responses
     var server = sinon.fakeServer.create();

     //  Save the book
     this.addBookView.$(".submit-button").click();

     //  Responding with what was sent in
     var requestBody = server.queue[0].requestBody;
     server.respond();
     server.restore();

     //  Check that we really save what we expect to have saved
     expect(JSON.parse(requestBody)).toEqual({
         author: "Miguel de Cervantes Saavedra",
         title: "Don Quixote",
         genre: "Picaresco"
     });
 });
```

The remaining clutter is related to handling Ajax requests with
[Sinon.JS](http://sinonjs.org/) and asserting that the saved book
matches the details we inserted. Let's start with cleaning up the Ajax
related code.

Hiding backend communication
----------------------------

Sinon.JS is a low-level helper, and it causes some of the same problems
as jQuery: A lot of low-level logic that isn't really relevant to the
test.

Our first step in solving this problem could be to wrap everything
inside a `save(book)` method in `addBookViewPageObject`, where the
provided `book` parameter is the JavaScript object we expect to have
saved:

```diff
 it('saves the book', function() {
     var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

     addBookViewPageObject(addBookView.$el).
         author("Miguel de Cervantes Saavedra").
         title("Don Quixote").
         genre("Picaresco").
+        save({
+            author: "Miguel de Cervantes Saavedra",
+            title: "Don Quixote",
+            genre: "Picaresco"
+        });

-    var server = sinon.fakeServer.create();
-
-    this.addBookView.$(".submit-button").click();
-
-    var requestBody = server.queue[0].requestBody;
-    server.respond();
-    server.restore();
-
-    expect(JSON.parse(requestBody)).toEqual({
-        author: "Miguel de Cervantes Saavedra",
-        title: "Don Quixote",
-        genre: "Picaresco"
-    });
 });
```

`addBookViewPageObject.js`:

```diff
 function addBookViewPageObject($el) {
     return {
         author: function(author) {
             $el.find(".author-input").
                 val(author).
                 change();
             return this;
         },
         title: function(title) {
             $el.find(".title-input").
                 val(title).
                 change();
             return this;
         },
         genre: function(genre) {
             dropDownViewPageObject($el.find(".genres-dropdown")).
                 openMenu().
                 chooseOption(genre);
             return this;
         },
+        save: function(book) {
+            var server = sinon.fakeServer.create();
+
+            $el.find(".submit-button").click();
+
+            var requestBody = server.queue[0].requestBody;
+
+            server.respond();
+            server.restore();
+
+            expect(JSON.parse(requestBody)).toEqual(book);
+        }
     };
 };
```

However, having the `save` method doing the assertion can be
problematic. Let's say that we for example want to assert that _no_ book
is saved if we add client-side validation. Another solution is to have
the `save` method return a new object that contains the relevant expect
methods:

```diff
 it('saves the book', function() {
     var addBookView = createAddBookView({ genres: ["Picaresco"] });
     addBookView.render();

     addBookViewPageObject(addBookView.$el).
         author("Miguel de Cervantes Saavedra").
         title("Don Quixote").
         genre("Picaresco").
-        save({
-            author: "Miguel de Cervantes Saavedra",
-            title: "Don Quixote",
-            genre: "Picaresco"
-        });
+        save().
+        expectToHaveSaved({
+            author: "Miguel de Cervantes Saavedra",
+            title: "Don Quixote",
+            genre: "Picaresco"
+        });
 });
```

`addBookViewPageObject.js`:

```diff
 function addBookViewPageObject($el) {
     return {
         author: function(author) {
             $el.find(".author-input").
                 val(author).
                 change();
             return this;
         },
         title: function(title) {
             $el.find(".title-input").
                 val(title).
                 change();
             return this;
         },
         genre: function(genre) {
             dropDownViewPageObject($el.find(".genres-dropdown")).
                 openMenu().
                 chooseOption(genre);
             return this;
         },
-        save: function(book) {
+        save: function() {
             var server = sinon.fakeServer.create();
 
             $el.find(".submit-button").click();
 
             var requestBody = server.queue[0].requestBody;
 
             server.respond();
             server.restore();
 
-            expect(JSON.parse(requestBody)).toEqual(book);
+            return {
+                expectToHaveSaved: function(attributes) {
+                    expect(JSON.parse(requestBody)).toEqual(attributes);
+                }
+            };
         }
     };
 };
```

And now we are getting somewhere! With this change our test looks like
this:

```javascript
it('saves the book', function() {
    var addBookView = createAddBookView({ genres: ["Picaresco"] });
    addBookView.render();

    addBookViewPageObject(addBookView.$el).
        author("Miguel de Cervantes Saavedra").
        title("Don Quixote").
        genre("Picaresco").
        save().
        expectToHaveSaved({
            author: "Miguel de Cervantes Saavedra",
            title: "Don Quixote",
            genre: "Picaresco"
        });
});
```

This test is easy to understand &mdash; it's succinct and reveals our
intent. It's also easy to keep up-to-date with changes in our
`AddBookView` and `DropDownView`. And perhaps best of all, there's
nearly no terms in this test that isn't relevant to the functionality
being tested.

Wrap up
-------

Generally, Page Objects should not know anything about your application
except for the DOM and Ajax requests &mdash; it should treat the
application as a black box.

In our experience Page Objects can considerably decrease the cost (and
pain) of writing JavaScript tests. We know that this might feel like a
heavy solution, but this is a way of handling hundreds or maybe
thousands of tests for a complex application and still being able to
maintain and (easily) understand the tests. Also, we can easily create
a couple of simple helpers for building Page Objects, as we have done
with [po.js](https://github.com/kjbekkelund/po.js).

To help you get started with Page Objects we have created a [sample
application](https://github.com/sveinung/pageobject-example) that you
can check out. Let us know about your experiences with writing
maintainable and understandable JavaScript tests!
