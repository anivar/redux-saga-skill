---
title: Channels and External I/O
impact: MEDIUM
tags: channels, eventChannel, actionChannel, multicast, websocket, external-io, buffers
---

# Channels and External I/O

## Channel Types

### actionChannel — Buffer Redux Actions

Queue Redux actions for sequential processing.

```javascript
import { actionChannel, take, call } from 'redux-saga/effects'

function* watchSaves() {
  const chan = yield actionChannel('SAVE_DOCUMENT')
  while (true) {
    const action = yield take(chan)
    yield call(saveDocument, action) // one at a time
  }
}
```

### eventChannel — External Event Sources

Bridge WebSockets, DOM events, timers into sagas.

```javascript
import { eventChannel, END } from 'redux-saga'

function createSocketChannel(socket) {
  return eventChannel((emit) => {
    // Never emit null/undefined — sanitize the source, and prefer
    // action-shaped payloads (`{ data }`) over bare primitives.
    socket.on('message', (data) => { if (data != null) emit({ data }) })
    socket.on('error', (err) => emit({ error: err }))
    socket.on('close', () => emit(END))
    return () => socket.close()
  })
}

function* watchSocket(socket) {
  const channel = yield call(createSocketChannel, socket)
  try {
    while (true) {
      const payload = yield take(channel)
      if (payload.error) {
        yield put(socketError(payload.error))
      } else {
        yield put(messageReceived(payload.data))
      }
    }
  } finally {
    channel.close()
  }
}
```

**Sanitize the event source.** The docs are explicit: "You need to sanitize your event sources
as to not pass null or undefined through the event channel. While it's fine to pass numbers
through, we'd recommend structuring your event channel data like your redux actions.
`{ number }` over `number`." Guard the emit rather than the taker.

**`eventChannel` is not buffered by default.** The `buffer` argument is optional and, per the
API docs, "if not provided, messages will not be buffered on this channel" — anything emitted
while no taker is registered is lost. In the loop above every effect between two takes is
synchronous, so the saga re-enters `take` in the same tick and nothing is dropped. Add any
async effect between takes (`yield call(api.save, payload)`, `yield delay(...)`) and messages
arriving in that window are gone. Pass a buffer when every message matters:

```javascript
const channel = eventChannel(subscribe, buffers.expanding(10))
```

The other two factories behave differently: `channel()` "will queue incoming messages up to 10
until interested takers are registered", and `actionChannel` "buffers all incoming messages
without limit".

### channel — Generic Saga-to-Saga Communication

Manual put/take for direct communication between sagas.

```javascript
import { channel } from 'redux-saga'

function* producer(chan) {
  while (true) {
    const data = yield call(fetchData)
    yield put(chan, data)
    yield delay(5000)
  }
}

function* consumer(chan) {
  while (true) {
    const data = yield take(chan)
    yield call(processData, data)
  }
}

function* rootSaga() {
  const chan = yield call(channel)
  yield fork(producer, chan)
  yield fork(consumer, chan)
}
```

### multicastChannel — Broadcast to Multiple Consumers

Each message is delivered to all takers, not just one.

`take` on a multicast channel requires a second `pattern` argument: "using `take` with the
`multicastChannel` requires us to pass the additional argument *pattern* — that you can use to
filter the actions to `take`." Unlike `actionChannel`/`eventChannel`/`channel`, the one-argument
form `take(chan)` fails at runtime here. Use `'*'` to receive everything.

```javascript
import { multicastChannel } from 'redux-saga'

function* workerA(chan) {
  while (true) {
    const payload = yield take(chan, '*') // pattern is REQUIRED here
    yield call(handleA, payload)
  }
}

function* workerB(chan) {
  while (true) {
    const payload = yield take(chan, 'BROADCAST') // or filter by type
    yield call(handleB, payload)
  }
}

function* root() {
  const chan = yield call(multicastChannel)
  yield fork(workerA, chan) // both receive every matching message
  yield fork(workerB, chan)
  yield put(chan, { type: 'BROADCAST' })
}
```

## Buffer Strategies

```javascript
import { buffers } from 'redux-saga'

// No buffer (what an eventChannel does when no buffer argument is passed):
// drops messages when no taker
eventChannel(sub, buffers.none())

// Fixed buffer: errors if queue exceeds limit
eventChannel(sub, buffers.fixed(10))

// Expanding buffer: grows dynamically
eventChannel(sub, buffers.expanding(10))

// Dropping buffer: silently drops new messages on overflow
eventChannel(sub, buffers.dropping(10))

// Sliding buffer: drops oldest messages on overflow
eventChannel(sub, buffers.sliding(10))
```

| Buffer | On Overflow | Use When |
|--------|------------|----------|
| `none()` (eventChannel default) | Drops message | Fire-and-forget events |
| `fixed(n)` | Throws error | Must process all, overflow is a bug |
| `expanding(n)` | Grows | Must process all, variable load |
| `dropping(n)` | Drops newest | Latest data not critical |
| `sliding(n)` | Drops oldest | Only latest messages matter |

## flush — Drain a Channel

```javascript
function* processBatch(chan) {
  while (true) {
    yield delay(5000) // wait 5 seconds
    const messages = yield flush(chan) // grab everything buffered
    if (messages.length > 0) {
      yield call(api.sendBatch, messages)
    }
  }
}
```

## runSaga — Connect to External I/O

Run a saga outside Redux, connected to any event system:

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
    getState: () => ({ /* app state */ }),
  },
  mySaga,
)
```
