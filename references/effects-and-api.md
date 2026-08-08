---
title: Effects and API Reference
impact: CRITICAL
tags: effects, api, call, put, take, fork, spawn, select, race, all
---

# Effects and API Reference

## Middleware Setup

```javascript
import createSagaMiddleware from 'redux-saga'
import { configureStore } from '@reduxjs/toolkit'
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware({
  // Optional: context available via getContext/setContext
  context: { api: apiClient },
  // Optional: handle uncaught errors
  onError: (error, { sagaStack }) => {
    console.error('Saga error:', error)
    console.error('Saga stack:', sagaStack)
  },
})

const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefault) => getDefault().concat(sagaMiddleware),
})

sagaMiddleware.run(rootSaga)
```

`middleware.run(saga, ...args)` returns a Task descriptor. Only call after store is created.

## Effect Creators — Detailed

### take(pattern)

Suspends the generator until a matching action is dispatched.

```javascript
// Wait for specific action
const action = yield take('USER_REQUESTED')

// Wait for any of several actions
const action = yield take(['LOGOUT', 'LOGIN_ERROR'])

// Custom predicate
const action = yield take(action => action.type.startsWith('USER_'))

// Wait for all actions
const action = yield take('*')
```

`take()` with no argument matches all actions. `take(channel)` takes from a channel instead of
the store; a `multicastChannel` is the exception there — it requires an explicit pattern,
`take(chan, '*')` rather than `take(chan)`.

`takeMaybe(pattern)` — same but receives `END` object instead of auto-terminating.

### takeEvery(pattern, saga, ...args)

Spawns a saga on every matching action. Multiple instances run concurrently.

```javascript
function* watchFetch() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}

function* fetchData(action) {
  const data = yield call(api.fetch, action.payload.url)
  yield put({ type: 'FETCH_SUCCEEDED', data })
}
```

### takeLatest(pattern, saga, ...args)

Spawns a saga on matching action, auto-cancels any previous running instance. Use when only the latest result matters (e.g., search-as-you-type).

```javascript
yield takeLatest('SEARCH_INPUT_CHANGED', performSearch)
```

### takeLeading(pattern, saga, ...args)

Spawns a saga on first matching action, ignores subsequent actions until the saga completes. Use for preventing duplicate submissions.

```javascript
yield takeLeading('SUBMIT_FORM', submitForm)
```

### put(action)

Dispatch an action to the Redux store. Non-blocking.

```javascript
yield put({ type: 'FETCH_SUCCEEDED', payload: data })

// Action creator
yield put(fetchSucceeded(data))
```

`putResolve(action)` — blocking variant that waits for the dispatch promise to resolve.

### call(fn, ...args)

Call a function and block until it returns/resolves.

```javascript
// Regular function
const result = yield call(myFunction, arg1, arg2)

// Promise-returning function
const data = yield call(fetch, '/api/users')

// Generator function (runs as sub-saga)
yield call(anotherSaga, param)

// Method invocation with context
yield call([obj, obj.method], arg1)
yield call([obj, 'methodName'], arg1)
yield call({ context: obj, fn: obj.method }, arg1)
```

`apply(context, fn, [args])` — alias for `call([context, fn], ...args)`.

### cps(fn, ...args)

Call a Node-style callback function: `fn(...args, (error, result) => {})`.

```javascript
const content = yield cps(fs.readFile, '/path/to/file')
```

### fork(fn, ...args)

Non-blocking. Creates an attached child task. Returns a Task object.

```javascript
const task = yield fork(backgroundSync)

// Parent waits for all forks to complete before returning
// Errors in forks bubble up and cancel the parent
```

### spawn(fn, ...args)

Non-blocking. Creates a detached task — independent lifecycle, errors don't bubble.

```javascript
const task = yield spawn(independentWorker)
// Parent cancellation does NOT cancel spawned tasks
// Errors in spawned tasks do NOT affect parent
```

### join(task) / join([...tasks])

Block until a forked task completes. Returns the task result.

```javascript
const task = yield fork(longRunning)
// ... do other work ...
const result = yield join(task)
```

### cancel(task) / cancel([...tasks]) / cancel()

Cancel a running task. The cancelled generator jumps to its `finally` block.

```javascript
const task = yield fork(bgSync)
yield take('STOP_SYNC')
yield cancel(task)

// Self-cancellation
yield cancel() // cancels the current saga
```

### select(selector, ...args)

Query the Redux store state.

```javascript
// Full state
const state = yield select()

// With selector
const user = yield select(getUser)
const item = yield select(getItemById, itemId)
```

### delay(ms, [val])

Block for `ms` milliseconds, then return `val` (default `true`).

```javascript
yield delay(1000) // pause 1 second
```

### actionChannel(pattern, [buffer])

Create a channel that buffers actions matching the pattern.

```javascript
const chan = yield actionChannel('REQUEST')
while (true) {
  const action = yield take(chan)
  yield call(handleRequest, action) // sequential processing
}
```

### flush(channel)

Extract all buffered messages from a channel.

```javascript
const queuedActions = yield flush(chan)
```

### cancelled()

Check if the current saga was cancelled. Use in `finally` blocks.

```javascript
function* saga() {
  try {
    // ...work...
  } finally {
    if (yield cancelled()) {
      // cancellation-specific cleanup
    }
    // common cleanup
  }
}
```

### setContext(props) / getContext(prop)

```javascript
yield setContext({ locale: 'en-US' })
const locale = yield getContext('locale')
```

## Effect Combinators

### race(effects)

Run multiple effects, first to complete wins. Losers are auto-cancelled.

```javascript
const { response, timeout } = yield race({
  response: call(fetchApi, url),
  timeout: delay(5000),
})

if (timeout) {
  yield put({ type: 'TIMEOUT_ERROR' })
}
```

Array form: `const [res1, res2] = yield race([effect1, effect2])` — winner gets result, losers get `undefined`.

### all([...effects]) / all(effects)

Run effects in parallel, wait for all to complete (like `Promise.all`).

```javascript
const [users, repos] = yield all([
  call(fetchUsers),
  call(fetchRepos),
])

// Object form
const { users, repos } = yield all({
  users: call(fetchUsers),
  repos: call(fetchRepos),
})
```

If any effect rejects, remaining effects are cancelled and the error is thrown.

## Concurrency Helpers

### throttle(ms, pattern, saga, ...args)

Spawn saga on a matching action, then hold off spawning again for `ms`
milliseconds. Actions arriving inside that window are not discarded: the most
recent one is buffered and spawns a trailing invocation once the window elapses.
So a burst produces a leading call plus one trailing call with the latest action
— at most one spawn per `ms`, but usually two per burst rather than one.

```javascript
yield throttle(500, 'SCROLL_EVENT', handleScroll)
// SCROLL_EVENT at t=0 and t=100 -> handleScroll runs twice:
// once at t=0 with the first action, once at t=500 with the second.
```

### debounce(ms, pattern, saga, ...args)

Wait for `ms` of silence after last matching action, then spawn saga.

```javascript
yield debounce(300, 'SEARCH_INPUT', performSearch)
```

### retry(maxTries, delay, fn, ...args)

Retry a function up to `maxTries` times with `delay` between attempts.

```javascript
const data = yield retry(3, 2000, fetchApi, '/endpoint')
```

## runSaga(options, saga, ...args)

Execute a saga outside Redux middleware. For connecting to external I/O or testing.

```javascript
import { runSaga, stdChannel } from 'redux-saga'
import { EventEmitter } from 'events'

const emitter = new EventEmitter()
const channel = stdChannel()
emitter.on('action', channel.put)

const task = runSaga(
  {
    channel,
    dispatch: (output) => emitter.emit('action', output),
    getState: () => currentState,
  },
  mySaga,
)
```

## Blocking vs Non-Blocking

| Blocking (saga pauses) | Non-Blocking (saga continues) |
|------------------------|------------------------------|
| take, takeMaybe | takeEvery, takeLatest, takeLeading |
| call, apply, cps | put, fork, spawn, cancel |
| join | select, flush, actionChannel, throttle, debounce |
| cancelled, delay, retry | setContext |
| race | |

`all` blocks only if the array or object it is given contains a blocking effect.

`select` and `flush` resolve synchronously, so neither ever suspends the
generator — and neither is usable inside `race`. A race whose only other arm is
`select` settles immediately and the other branches never run.
