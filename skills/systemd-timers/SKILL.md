> **Part of [Consola Skills](https://github.com/oberon-logistics/consola-skills) — battle-tested Hermes agent skills**

---
name: systemd-user-timers
version: 1.0.0
description: >
  Create and manage systemd user timers for scheduled tasks that must run
  independently of Hermes (e.g., Hermes self-updates, gateway restarts).
  Prefer systemd user timers over Hermes internal cron when the scheduled
  task could restart or kill the Hermes gateway itself.
tags: [cron, systemd, timer, scheduling, hermes-update]
---

# Systemd User Timers

Use systemd user timers when a scheduled task needs to survive Hermes gateway
restarts or when it must run entirely outside the Hermes process tree.

## When to Use This vs Hermes Internal Cron

| Scenario | Use |
|---|---|
| Task could restart/kill the Hermes gateway | **Systemd timer** |
| Task depends on Hermes being running | Hermes internal cron |
| Task is independent (scripts, updates) | **Systemd timer** |
| Task needs Hermes tools (delegate_task, etc.) | Hermes internal cron |
| Task needs AI context (session history, memory) | **Hermes cron** (see `hermes-cron` skill) |

## Directory Structure

```
~/.config/systemd/user/
├── <name>.service    # oneshot service definition
└── <name>.timer      # timer schedule
```

Logs go to: `~/.hermes/cron/output/<name>.log`

## Creating a New Timer

### 1. Create the service unit

```ini
# ~/.config/systemd/user/<name>.service
[Unit]
Description=<What it does>

[Service]
Type=oneshot
ExecStart=<absolute path to command>
StandardOutput=append:~/.hermes/cron/output/<name>.log
StandardError=append:~/.hermes/cron/output/<name>.log
```

**Always use absolute paths** — systemd doesn't inherit PATH or shell aliases.

### 2. Create the timer unit

```ini
# ~/.config/systemd/user/<name>.timer
[Unit]
Description=<Name> timer

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

`Persistent=true` means if the machine was off at the scheduled time, it'll
catch up on next boot.

### 3. Enable and start

```bash
systemctl --user daemon-reload
systemctl --user enable --now <name>.timer
systemctl --user status <name>.timer
```

### 4. Verify

```bash
systemctl --user list-timers
```

## Existing Timers

| Timer | Schedule | Purpose |
|---|---|---|
| `gmail-triage-auto.timer` | 30 min | Email triage via opencode agents |
| `hermes-nightly-update.timer` | 03:00 daily | Pull latest Hermes Agent release |

## Removing a Timer

```bash
systemctl --user stop <name>.timer
systemctl --user disable <name>.timer
rm ~/.config/systemd/user/<name>.service
rm ~/.config/systemd/user/<name>.timer
systemctl --user daemon-reload
```

## Pitfalls

- **No crontab on this system** — `crontab` command not available. Use systemd timers instead.
- **Always use absolute paths** in ExecStart — no shell expansion, no aliases.
- **Hermes internal cron + gateway restart = job killed** — any task that runs `hermes update` or otherwise restarts the gateway MUST use an external scheduler.
- **Create log directory first** — `mkdir -p ~/.hermes/cron/output/` before enabling a timer that writes logs there.
