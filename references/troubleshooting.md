---
title: Troubleshooting
impact: LOW
tags: troubleshooting, debugging, freeze, missed-actions, stack-traces, common-errors
---

# Troubleshooting

## App Freezes After Adding a Saga

**Cause:** Missing `yield` before an effect in a loop.

```javascript
// BUG: synchronous infinite loop
while (true) {
  const action = take('REQUEST') // missing yield
}

// FIX
while (true) {
  const action = yield take('REQUEST')
}
```

## Saga Misses Dispatched Actions

**Cause:** Saga blocked on a `call` effect when the action arrives.

**Fix:** Use `fork` for non-blocking execution so the saga remains responsive to other actions.

```javascript
// Instead of blocking call:
const result = yield call(api.fetch, url) // blocked — misses actions

// Use fork + take:
const task = yield fork(fetchWorker, url)
const action = yield take(['CANCEL', 'SUCCESS'])
if (action.type === 'CANCEL') yield cancel(task)
```

## Unreadable Error Stack Traces

**Fix 1:** Use `onError` with `sagaStack`:

```javascript
createSagaMiddleware({
  onError: (error, { sagaStack }) => {
    console.error(error.message)
    console.error(sagaStack)
  },
})
```

**Fix 2:** Add `babel-plugin-redux-saga` for dev:

```json
{ "plugins": ["redux-saga"] }
```

## Saga Doesn't Start

**Cause:** `sagaMiddleware.run()` called before store is created.

```javascript
// WRONG order
sagaMiddleware.run(rootSaga)
const store = configureStore(...)

// CORRECT order
const store = configureStore(...)
sagaMiddleware.run(rootSaga)
```

## TypeScript: yield Returns `any`

**Cause:** TypeScript can't infer generator yield types.

**Fix:** Type the result manually:

```typescript
const user: User = yield call(api.fetchUser, id)
```

Or use `typed-redux-saga`:

```typescript
import { call } from 'typed-redux-saga'
const user = yield* call(api.fetchUser, id) // properly typed
```

## Undefined Reaches a Saga Through a Channel

**Cause:** An event source emitted `null` or `undefined` into an `eventChannel`. The docs make
sanitizing the source the subscriber's job: "You need to sanitize your event sources as to not
pass null or undefined through the event channel. While it's fine to pass numbers through, we'd
recommend structuring your event channel data like your redux actions. `{ number }` over
`number`."

**Fix:** Guard at the emit site, not at the taker, and give the channel action-shaped payloads
rather than bare primitives:

```javascript
// BUG: forwards whatever the source produces
socket.on('message', (data) => emit(data))

// FIX
socket.on('message', (data) => { if (data != null) emit({ data }) })
```

## Effect Not Executing

**Cause:** Using effect creator result without yielding it.

```javascript
// BUG: creates a plain effect object describing the dispatch, nothing more
put(myAction()) // never dispatched — the middleware only runs effects that are yielded

// FIX
yield put(myAction())
```
