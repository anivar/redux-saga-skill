---
title: Use eventChannel for External Events
impact: MEDIUM
description: Connect WebSockets, timers, and other non-Redux event sources to sagas via eventChannel.
tags: channels, eventChannel, websocket, external-io
---

# Use eventChannel for External Events

## Problem

Sagas only react to Redux actions by default. External event sources (WebSocket, DOM events, timers) need to be bridged into the saga world.

## Incorrect

```javascript
// BUG: callback can't yield effects — this doesn't work
function* watchSocket() {
  socket.on('message', (data) => {
    yield put(messageReceived(data)) // SyntaxError: yield outside generator
  })
}
```

## Correct

```javascript
import { eventChannel, END } from 'redux-saga'
import { take, put, call } from 'redux-saga/effects'

function createSocketChannel(socket) {
  return eventChannel((emit) => {
    // Sanitize the source: never emit null or undefined, and prefer
    // action-shaped data (`{ data }`) over a bare primitive.
    socket.on('message', (data) => {
      if (data != null) emit({ data })
    })
    socket.on('close', () => emit(END)) // closes the channel

    // Return unsubscribe function
    return () => socket.close()
  })
}

function* watchSocket() {
  const socket = yield call(connectSocket)
  const channel = yield call(createSocketChannel, socket)

  try {
    while (true) {
      const { data } = yield take(channel)
      yield put(messageReceived(data))
    }
  } finally {
    channel.close()
  }
}
```

The subscriber contract has two halves. It must return an unsubscribe function — and it must
sanitize what it emits: "You need to sanitize your event sources as to not pass null or
undefined through the event channel. While it's fine to pass numbers through, we'd recommend
structuring your event channel data like your redux actions. `{ number }` over `number`."

## Choosing a Buffer (the default is none — messages drop)

Not an optional enhancement: an `eventChannel` created without a buffer argument does not buffer
at all, and any message emitted while no taker is registered is lost. Decide before shipping
whether that is acceptable.

```javascript
import { buffers } from 'redux-saga'

// Sliding buffer: keeps latest 10 messages, drops older ones
const channel = eventChannel(subscribe, buffers.sliding(10))
```

## Buffer Types

| Buffer | Behavior |
|--------|----------|
| `buffers.none()` (default) | Drops messages when no taker |
| `buffers.fixed(n)` | Fixed queue, errors on overflow |
| `buffers.expanding(n)` | Grows dynamically |
| `buffers.dropping(n)` | Drops new messages on overflow |
| `buffers.sliding(n)` | Drops oldest messages on overflow |
