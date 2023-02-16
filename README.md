# Concurrency in Javascript

## Why it's needed

There is a [**main thread**](https://developer.mozilla.org/en-US/docs/Glossary/Main_thread):

> The main thread is where a browser processes user events and paints. 
> By default, the browser uses a single thread to run all the JavaScript 
> in your page, as well as to perform layout, reflows, and garbage collection. 
> This means that long-running JavaScript functions can block the thread, 
> leading to an unresponsive page and a bad user experience.

If you have a long-running operation -- such as loading data on the web, or
performing long computation, whatever -- and you perform it sequentially on the main
thread, you "freeze" your browser. 

To avoid that, JS API propose register callback schemes deliver promises.

**TODO.** Explain better what DOESN'T exist, to protect the main thread,
since we are sharing the main thread with important actions that we cannot
delay (too much). Example of how we could starve the main thread despite
all these efforts? (**Without** burning the CPU).

For example a `sleep` doesn't exist since it would starve the main thread.

## Callbacks

HTML document

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      document.body.innerHTML += " world!";
    </script>
  </head>
  <body>
    Hello
  </body>
</html>
```

Likely error:

```
Uncaught TypeError: document.body is null
```

The HTML renderer has not processed the document yet when the main thread 
executes the javascript code.

Several solutions. Here we could: 

  - [`defer`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#attr-defer) the execution of the script (requires the script to be externalized).

  - use [Javascript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) (also requires the script the be externalized)

[`defer`]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script#attr-defer 
[Javascript modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules

One other solution, more explicit solution and less ad-hoc: 
register the body modification to happen only when the [`DOMContentLoaded`] event
is triggered. 

[`DomContentLoaded`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/DOMContentLoaded_event

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      function onDOMContentLoaded(event) {
        document.body.innerHTML += " world!";
      }
      addEventListener("DOMContentLoaded", onDOMContentLoaded);
    </script>
  </head>
  <body>
    Hello
  </body>
</html>
```

or the similar code with an anonymous function

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      addEventListener(
        "DOMContentLoaded",
        (event) => (document.body.innerHTML += " world!")
      );
    </script>
  </head>
  <body>
    Hello
  </body>
</html>
```

DOM Events and the corresponding event listeners and callbacks are also used
to react to user actions, such as a button click.

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      let button = document.querySelector("button");
      let count = 0;
      button.addEventListener("click", (event) => {
        count += 1;
        button.textContent = `Click count: ${count}`;
      });
    </script>
  </head>
  <body>
    <button>Click me!</button>
    <p>Click count: 0</p>
  </body>
</html>
```

... but this code doesn't work. Maybe you guess what the issue here:

```
Uncaught TypeError: button is null
```

The fix is *conceptually* straightforward ...

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      addEventListener("DOMContentLoaded", (event) => {
        let button = document.querySelector("button");
        let count = 0;
        button.addEventListener("click", (event) => {
          count += 1;
          button.textContent = `Click me! count: ${count}`;
        });
      });
    </script>
  </head>
  <body>
    <button>Click me!</button>
  </body>
</html>
```

And now, let's say that you want to delay the count update by 1 second.
Easy, use `setTimeout`! This function will register a callback to be called
when the duration is elapsed.

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      addEventListener("DOMContentLoaded", (event) => {
        let button = document.querySelector("button");
        let count = 0;
        button.addEventListener("click", (event) => {
          count += 1;
          button.textContent = `Click me! count: ?`;
          setTimeout((event) => {
            button.textContent = `Click me! count: ${count}`;
          }, 1000);
        });
      });
    </script>
  </head>
  <body>
    <button>Click me!</button>
  </body>
</html>
```

... and you start to feel that this kind of programming,
with callbacks within callbacks within callbacks ... is going to be
painful to write and hard to read & understand. Welcome to [callback hell]!
Also called "the pyramid of doom" ...

The structure of your code probably doesn't match at all how you think of
all these processes and their dependencies.

[callback hell]: http://callbackhell.com/

## TODO: Promises and Async/Await

## Real-life example to study:



### MathJax

Warning: doesn't work. Doesn't print anything and doesn't prevent the rendering.

```html
<!DOCTYPE html>
<html>
  <head>
    <script>
      MathJax = {
        ready: () => console.log("***"),
      };
    </script>
    <script src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
  </head>
  <body>
    $$ \int_0^1 f(x) \, dx $$
  </body>
</html>
```

Warning: this is far more complex than I was expecting ; the callback deals
with Promises ... Have a look at it again in detail.

> The pageReady() function is performed when MathJax is ready (all its components are loaded, and the internal objects have been created), and the page itself is ready (i.e., it is OK to typeset the page). The default is for pageReady() to perform the initial typesetting of the page, but you can override that to perform other actions instead, such as delaying the initial typesetting while other content is loaded dynamically, for example. The ready() function sets up the call to pageReady() as part of its default action.
> The return value of pageReady() is a promise that is resolved when the initial typesetting is finished (it is the return value of the initial MathJax.typesetPromise() call). If you override the pageReady() method, your function should return a promise as well. If your function calls MathJax.startup.defaultPageReady(), then you should return the promise that it returns (or a promise obtained from its then() or catch() methods). The MathJax.startup.promise will resolve when the promise you return is resolved; if you don’t return a promise, MathJax.startup.promise will resolve immediately, which may mean that it resolves too early.

### Python in the browser

```html
<!doctype html>
<html>
  <head>
    <script src="https://cdn.jsdelivr.net/pyodide/v0.22.1/full/pyodide.js"></script>
  </head>
  <body>
    Pyodide test page <br>
    Open your browser console to see Pyodide output
    <script type="text/javascript">
      async function main(){
        let pyodide = await loadPyodide();
        console.log(pyodide.runPython(`
            import sys
            sys.version
        `));
        pyodide.runPython("print(1 + 2)");
      }
      main();
    </script>
  </body>
</html>
```


**TODO**:

  - study the rendez-vous of PyOdide and MathJax

  - deal with this and a sync client. I think of my initial motivation here:
    have a Mithril component whose view depends on PyOdide and MathJax being
    ready to go. The "solution" could involve a default rendering (loading 
    message) if the stuff is not ready yet and for PyOdide and MathJax the
    scheduling of a `m.redraw()` when they are ready.