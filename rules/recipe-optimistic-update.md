---
title: Optimistic Update with Undo
impact: MEDIUM
description: Apply UI changes immediately and race between undo action and commit timeout.
tags: recipes, optimistic, undo, race
---

# Optimistic Update with Undo

## Pattern

```javascript
function* watchDelete() {
  while (true) {
    const action = yield take('DELETE_ITEM')
    // spawn, not fork/takeLatest: the optimistic update has already been applied, so the
    // task must survive parent cancellation to reach either the revert or the API commit.
    // Otherwise client and server diverge.
    yield spawn(deleteItemSaga, action)
  }
}

function* deleteItemSaga(action) {
  const { itemId } = action.payload
  const item = yield select(getItemById, itemId)

  // 1. Optimistic: update UI immediately
  yield put(removeItem(itemId))
  yield put(showUndoToast({ itemId, message: 'Item deleted' }))

  // 2. Race: undo vs timeout
  const { undo } = yield race({
    undo: take((a) => a.type === 'UNDO_DELETE' && a.payload.itemId === itemId),
    commit: delay(5000),
  })

  // 3. Revert or commit
  if (undo) {
    yield put(restoreItem(item))
  } else {
    try {
      yield call(api.deleteItem, itemId)
    } catch (e) {
      // API failed — revert the optimistic update
      yield put(restoreItem(item))
      yield put(showError('Delete failed'))
    }
  }

  yield put(hideUndoToast(itemId))
}
```

## Key Points

- Start the saga with `spawn`, not `takeLatest`/`fork` — once the optimistic `put` has
  dispatched, cancelling the task mid-`race` strands the UI in the optimistic state with neither
  the revert nor the commit ever running
- Save the current state before the optimistic update so you can restore it
- Use `race` with a filtered `take` to match the specific item's undo action
- Handle API failure by reverting even after the undo window expires
- Clear the undo toast regardless of outcome
