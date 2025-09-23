## 1. Callback Functions

A callback function is passed as an argument to another function, executed after a task completes. Callbacks enable asynchronous, non-blocking operations, such as event handling.

```javascript
setTimeout(() => console.log("Delayed execution"), 1000);
```

Here, the arrow function is a callback executed after a 1-second delay.

## 2. How JavaScript Runs in a Browser

JavaScript runs via a browser’s JavaScript engine (e.g., V8 in Chrome). The browser provides a runtime with the DOM for webpage manipulation, Web APIs for asynchronous tasks (e.g., `fetch`), and a single-threaded execution model. The engine parses, compiles, and executes code, interacting with APIs for events and rendering.

## 3. Event Loops in JavaScript

The event loop enables asynchronous execution in JavaScript’s single-threaded environment. It monitors the call stack and task queue, pushing tasks (e.g., callbacks) to the stack when the stack is empty, ensuring non-blocking behavior.

## 4. Synchronous vs. Asynchronous in JavaScript

- **Synchronous**: Code executes sequentially, blocking until a task completes (e.g., a `for` loop).
- **Asynchronous**: Code runs non-blocking; tasks like timers or API calls use callbacks, promises, or `async/await`.

```javascript
console.log("Start"); // Synchronous
setTimeout(() => console.log("Async"), 0); // Asynchronous
console.log("End"); // Synchronous
```

Output: `Start`, `End`, `Async`.


## 5. Closures in JavaScript

A closure is a function that retains access to its lexical scope’s variables after the outer function finishes. Closures enable data encapsulation and state persistence.

```javascript
function outer() {
  let count = 0;
  return function inner() {
    return count++;
  };
}
const counter = outer();
console.log(counter()); // 0
console.log(counter()); // 1
```

## 6. Just-In-Time Compilation in JavaScript

Just-In-Time (JIT) compilation optimizes JavaScript by interpreting code initially and compiling frequently executed code into machine code during runtime. This balances speed and memory, improving performance over pure interpretation.

## 7. Types of Functions in JavaScript

JavaScript supports:

- **Function Declarations**: Hoisted, defined with `function`.

  ```javascript
  function add(a, b) { return a + b; }
  ```
- **Function Expressions**: Assigned to variables, not hoisted.

  ```javascript
  const multiply = function(a, b) { return a * b; };
  ```
- **Arrow Functions**: Concise, no `this` binding.

  ```javascript
  const divide = (a, b) => a / b;
  ```
- **IIFE**: Run immediately upon definition.

  ```javascript
  (function() { console.log("IIFE"); })();
  ```
- **Async Functions**: Handle asynchronous tasks.

  ```javascript
  async function fetchData() { return await fetch("url"); }
  ```

## 8. Promise.all() Method in JavaScript

`Promise.all()` takes an iterable of promises and returns a promise that resolves when all input promises resolve, or rejects if any fail. It enables parallel asynchronous tasks.

```javascript
const promises = [
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3)
];
Promise.all(promises).then(values => console.log(values)); // [1, 2, 3]
```

## Conclusion

These JavaScript concepts—callbacks, execution models, event loops, synchronous/asynchronous operations, closures, JIT compilation, function types, and `Promise.all()`—are essential for building efficient web applications. They enable non-blocking, performant, and modular code.
