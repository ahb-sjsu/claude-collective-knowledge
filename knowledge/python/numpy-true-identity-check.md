---
title: np.True_ is not True -- use truthiness, not identity
tags: [numpy, boolean, identity, gotcha]
verified: 2026-03-25
platform: any
---

## Problem
Tests fail with `assert result is True` when `result` is a numpy boolean (`np.True_`).

## Solution
Use truthiness checks instead of identity checks:

```python
# Bad -- fails with numpy booleans
assert result is True

# Good
assert result
assert bool(result) is True
```

## What didn't work
`np.True_ is True` evaluates to `False` because `np.True_` is a different object than Python's built-in `True`, even though `np.True_ == True` is `True`.

## Context
- NumPy returns `np.bool_` from comparison operations
- This affects any code that uses `is True` / `is False` checks on values that might come from NumPy
- Python 3.10+, NumPy 1.x and 2.x
