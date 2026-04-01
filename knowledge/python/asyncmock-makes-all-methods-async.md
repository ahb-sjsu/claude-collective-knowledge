---
title: AsyncMock makes ALL methods async -- use MagicMock for sync methods
tags: [asyncmock, magicmock, unittest, testing, async]
verified: 2026-03-20
platform: any
---

## Problem
Tests break when using `AsyncMock` for an object that has both sync and async methods. The sync methods unexpectedly return coroutines.

## Solution
Use `MagicMock` for the object, and only use `AsyncMock` for the specific async methods:

```python
# Bad -- nc.jetstream() now returns a coroutine instead of a value
nc = AsyncMock()

# Good -- sync methods work normally, set async ones explicitly
nc = MagicMock()
nc.jetstream = MagicMock(return_value=js)
nc.subscribe = AsyncMock(return_value=sub)
```

## What didn't work
Using `AsyncMock` for the entire mock object. `AsyncMock` makes every attribute access return an `AsyncMock`, so calling `nc.jetstream()` returns a coroutine that must be awaited, even when the real method is synchronous.

## Context
- Python 3.10+, `unittest.mock`
- Common with NATS client mocking (`nats.connect()` is async but `nc.jetstream()` is sync)
- Applies to any library mixing sync and async interfaces
