# Section 1: JavaScript + TypeScript (10 Qs)

> [← Back to index](README.md)


### 1. Event Loop, Call Stack, Microtask vs Macrotask
- **Call Stack:** where JS runs functions, one at a time (single-threaded).
- **Web APIs / libuv:** timers, network calls and file I/O run outside the stack.
- **Microtask queue:** Promise `.then`, `await` continuations, `queueMicrotask`, `process.nextTick` (Node, runs even earlier).
- **Macrotask queue:** `setTimeout`, `setInterval`, `setImmediate`, I/O callbacks, UI events.
- **Rule:** when the stack is empty, the event loop runs **all microtasks**, then **one macrotask**, then all microtasks again, and repeats.

```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
// Output: 1, 4, 3, 2
```
**Interview line:** "Microtasks always run before the next macrotask. That's why a promise resolves before `setTimeout(0)`. A long chain of microtasks can starve rendering, so I keep CPU-heavy work off the main thread (worker threads or a queue)."

### 2. Closure — real use case
A closure is a function that **remembers the variables of its outer scope** even after the outer function has returned.
```js
function createRateLimiter(limit) {
  let count = 0;                 // private, kept alive by the closure
  return () => (count < limit ? (++count, true) : false);
}
const canCall = createRateLimiter(3);
```
**Real uses:** the debounce function (keeps the `timer` variable), React hooks (event handlers closing over state), private variables in modules, memoization caches, and middleware factories like `auth(role) => (req,res,next) => ...`.
**Watch out:** the *stale closure* bug in React, where `useEffect`/`setInterval` reads old state. Fix it with a dependency array, functional `setState(prev => ...)`, or `useRef`.

### 3. Promise.all vs allSettled vs race (plus any)
| Method | Resolves when | Rejects when | Use case |
|---|---|---|---|
| `Promise.all` | all succeed | **any one fails** (fail-fast) | Load user + orders + cart together; all are required |
| `Promise.allSettled` | all finish (success or fail) | never | Send 50 notifications and report which failed |
| `Promise.race` | first to **settle** | first settles with an error | Timeout: `race([fetch(), timeout(5000)])` |
| `Promise.any` | first to **succeed** | all fail | Fastest of several mirrors/CDNs |

### 4. Debounce vs Throttle
- **Debounce:** wait until the user **stops** for X ms, then run once. *Search box autocomplete, auto-save of a form.*
- **Throttle:** run **at most once every** X ms. *Scroll/resize handlers, sending a "typing…" event in chat, location updates.*
```js
const debounce = (fn, ms) => { let t; return (...a) => { clearTimeout(t); t = setTimeout(() => fn(...a), ms); }; };
const throttle = (fn, ms) => { let last = 0; return (...a) => { const now = Date.now(); if (now - last >= ms) { last = now; fn(...a); } }; };
```
**[My project]:** "I debounced the product search API by 300 ms, which cut API calls by about 80%. I throttled the socket 'typing' event to once every 2 s."

### 5. var vs let vs const, Hoisting
| | Scope | Re-assign | Hoisted? |
|---|---|---|---|
| `var` | function | yes | yes, initialised to `undefined` |
| `let` | block | yes | yes, but in the **Temporal Dead Zone** (ReferenceError before the declaration) |
| `const` | block | no (but objects can still be mutated) | same as `let` |

- **Hoisting:** declarations are moved to the top at compile time. Function declarations are hoisted **with** their body. Function expressions/arrow functions are not.
- Classic bug: `for (var i...) setTimeout(() => console.log(i))` prints `3,3,3`. With `let` it prints `0,1,2`, because `let` creates a new binding on each iteration.
- **My rule:** `const` by default, `let` when I need to reassign, never `var`.

### 6. Deep copy vs Shallow copy
- **Shallow:** copies only the top level. Nested objects are still shared references. `{...obj}`, `Object.assign`, `arr.slice()`.
- **Deep:** fully independent copy. `structuredClone(obj)` (modern, handles Date/Map/Set/cycles), `JSON.parse(JSON.stringify(obj))` (loses Date, undefined and functions), `lodash.cloneDeep`.
- **Why it matters:** in React/Redux, mutating a nested object in a shallow copy means no re-render, or bugs from shared state.

### 7. `this` in arrow vs normal function
- **Normal function:** `this` depends on **how it is called**. `obj.fn()` gives `obj`. Called alone, it is `undefined` (strict) or `window`. It can be changed with `call/apply/bind`.
- **Arrow function:** has **no own `this`**. It takes `this` from the surrounding scope (lexical), and `bind` cannot change it.
```js
const user = {
  name: 'A',
  normal() { setTimeout(function () { console.log(this.name); }, 0); }, // undefined
  arrow()  { setTimeout(() => console.log(this.name), 0); },           // 'A'
};
```
- Don't use arrow functions as object methods or prototype methods when you need `this`. Do use them for callbacks inside methods.

### 8. TypeScript: Interface vs Type, Generics, any vs unknown
- **Interface:** describes object shapes, can be **extended** and **merged** (declaration merging). Good for public APIs and classes.
- **Type:** can do unions, intersections, tuples, mapped and conditional types. `type Status = 'PENDING' | 'PAID' | 'FAILED'`.
- **Generics:** reusable code that is still type-safe.
```ts
type ApiResponse<T> = { success: boolean; data: T; error?: string };
async function get<T>(url: string): Promise<ApiResponse<T>> { /* ... */ }
const res = await get<Order[]>('/orders');   // res.data is Order[]
```
- **`any`:** turns off type checking (unsafe). **`unknown`:** "could be anything, but you must **narrow** it before use." Use `unknown` for `catch (e)`, JSON from APIs and webhook bodies, then validate with Zod.

### 9. Optional chaining + Nullish coalescing
- `user?.address?.city` returns `undefined` instead of crashing if any part is null/undefined. Also `fn?.()` and `arr?.[0]`.
- `??` falls back **only for null/undefined**. `||` falls back for **any falsy value** (0, '', false).
```js
const qty = input.qty ?? 1;   // qty = 0 stays 0  ✅
const qty2 = input.qty || 1;  // qty = 0 becomes 1 ❌ (bug for prices/quantities)
```

### 10. Memory leaks in JS — causes and fixes
**Common causes:**
1. Event listeners / socket listeners not removed (`socket.on` inside `useEffect` with no cleanup).
2. `setInterval` never cleared.
3. Global variables / caches that grow forever (an in-memory Map with no TTL).
4. Closures holding large objects.
5. Detached DOM nodes still referenced.

**Fixes:** cleanup in `useEffect`, `removeEventListener`, `clearInterval`, `AbortController` for fetch, an LRU cache with a max size or Redis with TTL, `WeakMap` for object-keyed caches.
**How to detect:** Chrome DevTools → Memory → heap snapshots (compare two snapshots). In Node: `process.memoryUsage()`, `--inspect`, `clinic.js`, and watching container memory graphs that keep climbing.

