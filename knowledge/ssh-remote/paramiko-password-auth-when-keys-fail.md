---
title: Use paramiko with explicit password when SSH keys have passphrase issues
tags: [paramiko, ssh, password-auth, python]
verified: 2026-03-28
platform: any
---

## Problem
Need to SSH to a remote machine from a Python script, but the SSH key has an unknown passphrase and `ssh` CLI password auth fails in non-interactive contexts.

## Solution
Use `paramiko` with explicit password authentication:

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('hostname', username='user', password='password')

stdin, stdout, stderr = ssh.exec_command('ls -la')
print(stdout.read().decode())

ssh.close()
```

For file transfers, use SFTP:

```python
sftp = ssh.open_sftp()
sftp.put('local_file', '/remote/path')
sftp.get('/remote/path', 'local_file')
sftp.close()
```

## What didn't work
- `ssh` CLI with password via `-o PasswordAuthentication=yes` -- prompts for password interactively
- `sshpass` -- not always available, unreliable
- `ssh-agent` -- key has unknown passphrase
- `subprocess.Popen` with stdin pipe -- unreliable password input timing

## Context
- Works from Windows, Linux, or macOS
- `pip install paramiko`
- For Windows paths in output, use `sys.stdout.reconfigure(encoding='utf-8', errors='replace')` to handle unicode
