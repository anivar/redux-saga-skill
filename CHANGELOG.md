# Changelog

All notable changes to this skill are documented here.

## 1.1.1 — 2026-11-22

Audited against **redux-saga 1.5.1**, as in 1.1.0. Frontmatter and navigation only; no rule changes.

### Changed

- The description says when to load the skill: any file that imports redux-saga, and a saga to write, debug or refactor, even when the user only describes a race, a leak, a missed action or a request that never cancels.
- The frontmatter carries only the fields of the Agent Skills specification. `user-invocable: false` and `agentic: false` are removed: `agentic` is not a field of the specification, and claude.ai and the Skills API refuse a skill whose frontmatter has a key outside the specification. In Claude Code, `/redux-saga` now loads the skill by hand.
- Each reference file over 100 lines opens with a contents list, so a partial read still shows what the file covers.

## 1.1.0 — 2026-08-08

Audited against **redux-saga 1.5.1** (npm `latest`, released 2026-07-26) and the official
documentation at <https://redux-saga.js.org>. Every code example added or changed in this
release was executed against redux-saga 1.5.1 before being written down.

### Baseline

- Declared baseline moved from redux-saga 1.4.2 to 1.5.1 (`SKILL.md`, `AGENTS.md`, `README.md`).
  No effect creator was added, removed, renamed or deprecated in that range, so no teaching
  content changed as a consequence of the bump.
- `redux-saga-test-plan` prerequisite corrected from `^5.x` — a major version that has never been
  published — to `^4.0.6`, the current npm `latest`.

### Fixed

- **`select`, `flush` and `all` were listed as blocking.** `select` and `flush` resolve
  synchronously; `all` blocks only if a child effect blocks. The old table licensed
  `race({ data: select(sel), timeout: delay(5000) })`, which settles immediately and leaves the
  timeout branch unreachable. (`SKILL.md`, `AGENTS.md`, `references/effects-and-api.md`,
  `rules/effect-race-patterns.md`)
- **Attached-fork cancellation described as automatic.** A parent that finishes its own body
  *waits* for its attached forks; it does not cancel them. With the example's `while (true)`
  refresh loop, the explicit `cancel` is the only thing that lets the parent ever return.
  (`rules/fork-attached-vs-detached.md`)
- **`throttle` described as discarding intra-window actions.** It buffers the most recent one and
  fires a trailing invocation, so a burst produces a leading *and* a trailing call.
  (`references/effects-and-api.md`)
- **Pattern argument presented as optional for six helpers.** Only `take`/`takeMaybe` may omit it.
  `takeEvery`, `takeLatest`, `takeLeading`, `throttle` and `debounce` all take `pattern` as a
  required parameter; use `'*'` for a catch-all watcher. (`SKILL.md`, `AGENTS.md`)
- **Troubleshooting entry quoted a warning redux-saga does not emit.** Replaced with the
  documented rule it was reaching for: event sources must be sanitized so that `null`/`undefined`
  never enters an event channel. (`references/troubleshooting.md`)
- **Pre-1.0 effect object shapes.** `{ PUT: { action } }` and `{ CALL: { fn, args } }` are
  redux-saga 0.x and no longer describe anything the library produces, which broke the
  `deepEqual` assertions documented alongside them. The shape is an undocumented internal, so the
  skill now describes effects as the docs do — plain objects the middleware executes — and keeps
  assertions pointed at the effect creator itself. (`references/troubleshooting.md`,
  `rules/effect-use-call.md`, `AGENTS.md`)

### Added

- **`take` on a `multicastChannel` requires a `pattern` argument.** The one-argument form
  `take(chan)` — the form the generic-`channel` example teaches — fails at runtime on a multicast
  channel. Documented with a two-worker example. (`references/channels.md`,
  `references/effects-and-api.md`, `SKILL.md`, `AGENTS.md`)
- **`eventChannel` is unbuffered by default.** Messages emitted while no taker is registered are
  lost; `channel()` queues up to 10 and `actionChannel` buffers without limit, so the three do not
  behave alike. Buffer choice is now framed as a decision, not an enhancement.
  (`references/channels.md`, `rules/channel-event-channel.md`, `AGENTS.md`)
- **The `eventChannel` subscriber's sanitization contract.** Never emit `null`/`undefined`;
  prefer action-shaped payloads (`{ data }` over a bare primitive). Examples now guard the emit.
  (`rules/channel-event-channel.md`, `references/channels.md`, `references/troubleshooting.md`,
  `AGENTS.md`)
- **A watcher for the optimistic-undo recipes, started with `spawn`.** Under `takeLatest` a second
  action cancels the first worker mid-`race` after the optimistic `put` has dispatched, so
  neither the revert nor the API commit runs and client and server diverge permanently.
  (`references/recipes.md`, `rules/recipe-optimistic-update.md`,
  `rules/recipe-throttle-debounce.md`)
- **Batching dispatched actions.** Wrapping `stdChannel().put` to unwrap a `BATCH` envelope, and
  `createSagaMiddleware({ channel })`. Without it, every `take`/`takeEvery` watcher silently stops
  firing once `redux-batched-actions` is introduced. The existing analytics recipe was renamed to
  "Batching Outgoing Side Effects" so the two no longer collide. (`references/recipes.md`,
  `AGENTS.md`)

## 1.0.0

- Initial release: 22 rules across 7 categories, 7 deep-dive references, compiled `AGENTS.md`.
