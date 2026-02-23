# Redux-Saga Skill

An AI agent skill for writing, testing, and debugging Redux-Saga code with modern best practices.

## What's Inside

- **22 rules** with incorrect→correct code examples across 7 categories
- **7 deep-dive references** covering the full redux-saga API, fork model, channels, recipes, and troubleshooting
- **Testing coverage** for redux-saga-test-plan (expectSaga, testSaga, providers, matchers) with Jest and Vitest
- **Anti-patterns** reference with 12 common mistakes
- **Redux Toolkit integration** guidance (configureStore, createSlice, when to use saga vs RTK Query)

## Install

```bash
npx skills add anivar/redux-saga-skill -g
```

Or with full URL:

```bash
npx skills add https://github.com/anivar/redux-saga-skill
```

Testing sub-skill only:

```bash
npx skills add https://github.com/anivar/redux-saga-skill --skill redux-saga-testing
```

## Baseline

- redux-saga ^1.4.2
- @reduxjs/toolkit (recommended)
- redux-saga-test-plan ^5.x (for testing skill)
- Jest or Vitest

## Structure

```
├── SKILL.md                          # Entry point for AI agents
├── AGENTS.md                         # Compiled guide for agents
├── rules/                            # Individual rules (Incorrect→Correct)
│   ├── effect-*                      # Effects & yielding (CRITICAL)
│   ├── fork-*                        # Fork model & concurrency (CRITICAL)
│   ├── error-*                       # Error handling (HIGH)
│   ├── testing-*                     # Testing patterns (HIGH)
│   ├── recipe-*                      # Common patterns (MEDIUM)
│   ├── channel-*                     # Channels & external I/O (MEDIUM)
│   ├── rtk-*                         # Redux Toolkit integration (MEDIUM)
│   └── troubleshoot-*               # Debugging (LOW)
├── references/                       # Deep-dive reference docs
│   ├── effects-and-api.md
│   ├── fork-model.md
│   ├── testing.md
│   ├── channels.md
│   ├── recipes.md
│   ├── anti-patterns.md
│   └── troubleshooting.md
└── redux-saga-testing/              # Standalone testing skill
    ├── SKILL.md
    └── references/
        ├── api-reference.md
        └── anti-patterns.md
```

## License

MIT
