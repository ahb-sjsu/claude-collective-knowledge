---
title: gh secret set via PowerShell 5.1 pipe corrupts secret with UTF-16 BOM
tags: [github-actions, gh-cli, powershell, secrets, pypi, windows, encoding]
verified: 2026-07-13
platform: windows
---

## Problem

Setting a GitHub Actions secret from Windows PowerShell 5.1 by piping the value:

```powershell
$tok | gh secret set PYPI_API_TOKEN --repo owner/repo
```

silently prepends a UTF-16 BOM (`﻿`) to the secret value. `gh` reports
success and the secret looks fine in the repo settings (values are never
displayed), so nothing seems wrong until a workflow consumes it.

For a PyPI publish workflow (`pypa/gh-action-pypi-publish` with
`password: ${{ secrets.PYPI_API_TOKEN }}`), the failure shows up deep in
twine/requests as:

```
UnicodeEncodeError: 'latin-1' codec can't encode character '﻿' in position 0: ordinal not in range(256)
```

"position 0" is the tell: the secret's first character is the BOM, not the
token's `p` in `pypi-...`. Other consumers may fail with generic 401s instead,
which is even harder to trace back to the secret.

## Solution

Pass the value as an argument instead of via stdin:

```powershell
$tok = "...secret value..."
gh secret set PYPI_API_TOKEN --repo owner/repo --body $tok
```

Re-setting the secret with `--body` and re-running the failed workflow fixed it
immediately. (Caveat: the value briefly appears in the local process command
line — fine on your own machine, avoid on shared hosts.)

## What didn't work

- `$tok | gh secret set ...` — PowerShell 5.1 encodes the pipeline to a native
  process using `$OutputEncoding`, and the string handed to `gh` arrives with a
  BOM prefix. `gh` stores exactly the bytes it reads, BOM included.
- Diagnosing from the workflow's side first: the twine traceback points at
  requests' latin-1 auth-header encoding, which looks like a twine/requests bug
  or a malformed token, not a secret-storage problem. Checking whether the token
  works locally (it did, via `~/.pypirc`) is what isolated the secret as the
  broken link.

## Context

- Windows 10, Windows PowerShell 5.1 (powershell.exe), gh CLI authenticated via
  keyring
- Consumer: GitHub Actions `pypa/gh-action-pypi-publish@release/v1` →
  twine 6.x → requests basic-auth (latin-1 header encoding surfaces the BOM)
- PowerShell 7 (pwsh) defaults to BOM-less UTF-8 for native pipes, so this is
  specific to 5.1; `cmd /c "type token.txt | gh secret set ..."` would also
  avoid it, but `--body` is simplest
