Asynchronous context tracking

|        |                    |
| ------ | ------------------ |
| module | async_hooks        |
| code   | lib/async_hooks.js |
| Added  | **v8.0.0**         |

||

AsyncLocalStorage

(ASL from now on)

|            |                   |
| ---------- | ----------------- |
| class      | AsyncLocalStorage |
| Added      | **v13.10.0**      |
| Backported | **v12.17.0**      |
| Stabilized | **v16.4.0**       |

||

Documentation states:

> These classes (AsyncLocalStorage, AsyncResource) are used in associating and propagating state throughout callbacks and promise chains. They allow storing data throughout the lifetime of a web request or any other asynchronous duration. It is similar to thread-local storage in other languages.

To simplify the explanation further, AsyncLocalStorage allows you to store a state when executing an async function and then make it available to all the code paths within that function.

||

Why one shall use ASL?

Other languages (eg. PHP), create a new thread at every request . Each thread has its own memory. Thus storing data in global memory and access it anywhere is easy.

On the other side, node.js runs on a single thread and shares memory across all the HTTP requests, thus isolation cannot be achieved this way.

> ASL addresses this use case, as it allows isolated state between multiple async operations.

||

How did we make it up until now?

1. Pass the state around as function/class argument _(our approach so far)_
2. Implement a custom solution using async_hooks (after v8.x)
3. Use a node module as `cls-hooked` (which internally uses async_hooks)

---

API (1/2)

1. new AsyncLocalStorage()
   - creates a new instance of AsyncLocalStorage
2. asl.getStore()
   - returns the current store
3. asl.run(store, callback[,...args])
   - runs a function synchronously within a context and returns its return value
   - store is not accessible outside the callback
   - store is accessible to any asynchronous operations created within the callback
   - if the callback throws an error, it is thrown by `run` too

||

API (2/2)

4. _Experimental_ asl.enterWith(store)
   - Transitions into the context for the remainder of the current synchronous execution and then persists the store through any following asynchronous calls.
5. _Experimental_ asl.disable()
   - Disables the instance of AsyncLocalStorage. All subsequent calls to asyncLocalStorage.getStore() will return undefined until asyncLocalStorage.run() or asyncLocalStorage.enterWith() is called again.
   - Calling asyncLocalStorage.disable() is required before the asyncLocalStorage can be garbage collected. This does not apply to stores provided by the asyncLocalStorage, as those objects are garbage collected along with the corresponding async resources.
6. _Experimental_ asl.exit(callback[,...args])
   - Runs a function synchronously outside of a context and returns its return value.
   - The store is not accessible within the callback function or the asynchronous operations created within the callback.
   - Any getStore() call done within the callback function will always return undefined.

---

Code example

```js [2|4|6|2-6]
// storage.js
const { AsyncLocalStorage } = require("async_hooks");

const storage = new AsyncLocalStorage();

module.exports = storage;
```

```js [2-3|5-10|2-14]
// main.js
const storage = require("./storage");
const someModule = require("./someModule");

function run(id) {
  const state = { id };

  return storage.run(state, async () => {
    await someModule.func();
  });
}

module.exports = { run };
```

```js [2|4-6|2-8]
// someModule.js
const storage = require("./storage");

function func(id) {
  console.log(storage.getStore());
}

module.exports = { func };
```

```js
$ node -e "require('./main.js').run(1)"
1
```

---

Too good to be true?

- Never access ASL at the top level of any module (code will be executed only once!)
- Never access ASL inside static properties (they are evaluated as soon as the module is imported)
- If, whithin an async function, only one await call is to run withn a context
- Context loss
  - callback-based codebase: it is enough to promisify it with `util.promisify()`
  - if it needs to keep using callback-based API or custom thenable implementation, use `AsyncResource` to associate the async operation with the correct execution context.
- Performance overhead
  | | | |
  | --- | --- | --- |
  | ![cls-als](/images/cls-vs-als.png) <!-- .element width="400px" --> |![none-vs-als](/images/none-vs-als.png) <!-- .element width="400px" --> | ![none-vs-hook-vs-als](/images/none-vs-hook-vs-als.png) <!-- .element width="400px" -->|

---

Thank you! 😁

Discussion Time 🤔

---

Sources

1. [Node.js documentation](https://nodejs.org/api/all.html#all_async_context_asynchronous-context-tracking)
1. [AdonisJS blogpost](https://docs.adonisjs.com/guides/async-local-storage)
1. [Andrey Pechkurov benchmarks](https://twitter.com/andreypechkurov/status/1234189388436967426)
1. [Thread Local Storage](https://en.wikipedia.org/wiki/Thread-local_storage)
