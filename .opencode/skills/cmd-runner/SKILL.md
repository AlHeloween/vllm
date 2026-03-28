---
name: cmd-runner
description: Run interactive Windows commands safely via cmd_runner (ConPTY-only) with per-run logs and an inbox bridge.
---

# cmd-runner

Use this skill when a command may be:
- long/noisy,
- interactive (prompts, TUIs),
- crash-prone or likely to destabilize the agent when run directly.

## What cmd_runner is (current)

- Windows-only, ConPTY-only, serverless (no background server, no TCP control plane).
- Root-only policy:
  - Repo checkout: must be launched from the repo root (cwd contains `cmd_runner.py` and `cmd_runner_pkg/`).
  - Release bundle: run from the bundle root (cwd contains `cmd_runner.exe`).
- Logs are written to: `logs/cmd_runner/<run_id>/`
- Programmatic input bridge: append JSONL messages to `logs/cmd_runner/<run_id>/inbox.jsonl`.

## How to run it (recommended)

- Repo/dev (any shell; deterministic file entrypoint):
  - `python cmd_runner.py ...` or `uv run cmd_runner.py ...`
- Release bundle:
  - `cmd_runner.exe ...` (preferred; no `uv` required)

## Core workflow

1) Start an interactive run (spawns a new window):
- `python cmd_runner.py start -- <command ...>`
  - Prints `run_id` and `inbox=` path in the *current* terminal.
  - Do NOT use `--terminal conhost` - use default terminal.
  - Use PowerShell for Windows commands: `powershell -c "..."`
  - For .cmd/.bat scripts: `python cmd_runner.py start -- powershell -c "& { cd D:/path; .\_run_asm.cmd }"`

2) Check status first (from the current terminal):
- `python cmd_runner.py list`
- `python cmd_runner.py status <run_id>`
  - Use `status` first to confirm the program is still alive and did not crash before reading output.

3) Tail output (from the current terminal):
- `python cmd_runner.py tail <run_id>` (repo root)
  - Start with non-follow `tail` for a compact snapshot.
  - Add `--follow` only when live streaming is needed.
  - Prefer repeated `status`/non-follow `tail` checks over ad hoc shell sleeps; keep delay/wait handling inside the cmd_runner workflow.

4) Inject input programmatically (bridge):
- Append JSONL to: `logs/cmd_runner/<run_id>/inbox.jsonl`
- Built-in (preferred):
  - `python cmd_runner.py send <run_id> --keys "TEXT:/exit,ENTER"`

5) Stop (serverless terminate):
- `python cmd_runner.py stop <run_id> --reason "done"`
  - Writes `logs/cmd_runner/<run_id>/stop_request.json`; the hosting cmd_runner watches for it and terminates the Job Object.

Notes:
- `add_crlf` defaults to `false` (no implicit Enter). Use `ENTER` in `keys` or `--crlf` in the helper.
- Use `powershell -c "..."` for Windows commands, NOT `cmd /c`.
- For `.cmd` scripts, use PowerShell: `powershell -c "& { cd D:/path; .\_run_asm.cmd }"`
