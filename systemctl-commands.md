# systemctl & systemd I Actually Use

A proper reference for `systemctl` and `journalctl` — the two commands you end up using constantly once you're managing real Linux servers. The Linux cheatsheet covers the basics; this goes deeper.

Also covers writing your own service files, which is something every backend developer eventually needs to do.

---

## Table of Contents

- [Service Management](#service-management)
- [Checking Status & Info](#checking-status--info)
- [Enable / Disable on Boot](#enable--disable-on-boot)
- [Masking Services](#masking-services)
- [Viewing Logs with journalctl](#viewing-logs-with-journalctl)
- [Writing a Service File](#writing-a-service-file)
- [Targets (Runlevels)](#targets-runlevels)
- [System State & Boot Analysis](#system-state--boot-analysis)
- [Timers — Cron but systemd](#timers--cron-but-systemd)
- [Troubleshooting](#troubleshooting)

---

## Service Management

The commands you'll run most often.

| Command | What it does |
|---|---|
| `sudo systemctl start nginx` | Start a service right now |
| `sudo systemctl stop nginx` | Stop a service right now |
| `sudo systemctl restart nginx` | Stop then start — briefly interrupts connections |
| `sudo systemctl reload nginx` | Reload config without stopping — not all services support this |
| `sudo systemctl try-restart nginx` | Restart only if it's already running. Does nothing if stopped |
| `sudo systemctl condrestart nginx` | Same as try-restart — older alias, still common |
| `sudo systemctl kill nginx` | Send a signal to a service. Defaults to SIGTERM. `--signal=SIGKILL` to force |

> **`restart` vs `reload`** — use `reload` whenever the service supports it. It applies config changes without dropping connections. `restart` is a hard stop + start and will briefly interrupt traffic.

---

## Checking Status & Info

| Command | What it does |
|---|---|
| `systemctl status nginx` | Shows running state, PID, recent log lines, and whether it's enabled on boot |
| `systemctl status nginx -l` | Same but shows full log lines without truncation |
| `systemctl is-active nginx` | Prints `active` or `inactive` — clean output for use in scripts |
| `systemctl is-enabled nginx` | Prints `enabled` or `disabled` — whether it starts on boot |
| `systemctl is-failed nginx` | Prints `failed` if the service has crashed |
| `systemctl list-units` | Lists all currently loaded units (services, mounts, sockets, etc.) |
| `systemctl list-units --type=service` | Lists only services |
| `systemctl list-units --state=failed` | Shows only failed services — first thing to check after a server restart |
| `systemctl list-unit-files` | Lists all installed unit files and their enabled/disabled state |
| `systemctl show nginx` | Shows all properties of a service — timeout values, PID, environment, everything |
| `systemctl cat nginx` | Prints the actual service file content — useful to see what's really configured |
| `systemctl list-dependencies nginx` | Shows what nginx depends on and what depends on nginx |

---

## Enable / Disable on Boot

| Command | What it does |
|---|---|
| `sudo systemctl enable nginx` | Auto-start nginx when the system boots |
| `sudo systemctl disable nginx` | Remove auto-start on boot (doesn't stop it if currently running) |
| `sudo systemctl enable --now nginx` | Enable AND start immediately in one command |
| `sudo systemctl disable --now nginx` | Disable AND stop immediately in one command |

> **Enable ≠ Start.** `enable` only affects boot behaviour. Always use `--now` if you want both at once.

---

## Masking Services

Masking is stronger than disabling — it makes a service completely unlaunchable, even manually.

| Command | What it does |
|---|---|
| `sudo systemctl mask nginx` | Prevents nginx from being started by anything — manual or automatic |
| `sudo systemctl unmask nginx` | Removes the mask, making the service usable again |

Use masking when you want to make absolutely sure a service never starts — for example, masking `apache2` after switching to nginx to prevent it from accidentally starting.

---

## Viewing Logs with journalctl

`journalctl` is the other half of systemd. All your service logs go through it.

| Command | What it does |
|---|---|
| `journalctl -u nginx` | All logs for nginx since boot |
| `journalctl -u nginx -f` | Follow logs in real time — like `tail -f` |
| `journalctl -u nginx -n 100` | Last 100 lines |
| `journalctl -u nginx --since today` | Logs from today only |
| `journalctl -u nginx --since "1 hour ago"` | Logs from the last hour |
| `journalctl -u nginx --since "2025-01-15 10:00" --until "2025-01-15 11:00"` | Logs within a specific time window |
| `journalctl -u nginx -p err` | Only error-level logs and above |
| `journalctl -u nginx -p err..crit` | Logs between error and critical severity |
| `journalctl --disk-usage` | How much space journal logs are taking up |
| `journalctl --vacuum-size=500M` | Trim journal logs to 500MB — frees disk space |
| `journalctl --vacuum-time=7d` | Delete journal entries older than 7 days |
| `journalctl -b` | Logs from the current boot only |
| `journalctl -b -1` | Logs from the previous boot — useful after a crash/reboot |
| `journalctl -k` | Kernel messages only (equivalent to `dmesg`) |

### Log priority levels (low to high)

`debug` → `info` → `notice` → `warning` → `err` → `crit` → `alert` → `emerg`

Using `-p err` shows `err` and everything above it.

---

## Writing a Service File

This is the thing most backend developers need eventually — running your own FastAPI app, Celery worker, or custom script as a proper managed service.

### Where service files live

```
/etc/systemd/system/        ← your custom services go here (takes priority)
/lib/systemd/system/        ← system-installed services (apt/dpkg managed)
/usr/lib/systemd/system/    ← same as above on some distros
```

### A basic service file for a FastAPI app

```ini
# /etc/systemd/system/myapp.service

[Unit]
Description=My FastAPI Application
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/myapp
EnvironmentFile=/home/ubuntu/myapp/.env
ExecStart=/home/ubuntu/myapp/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### After creating or editing a service file

```bash
# reload systemd so it picks up the new file
sudo systemctl daemon-reload

# enable and start it
sudo systemctl enable --now myapp

# check it's running
sudo systemctl status myapp
```

> **Always run `daemon-reload` after editing a `.service` file.** Systemd won't pick up your changes otherwise and you'll be debugging a stale config.

### Key service file options explained

| Option | What it does |
|---|---|
| `After=` | Start this service only after these units are started |
| `Wants=` | Soft dependency — tries to start the listed units, but doesn't fail if they don't start |
| `Requires=` | Hard dependency — if the listed unit fails, this one stops too |
| `Type=simple` | The default. Process started by ExecStart is the main process |
| `Type=forking` | For services that fork into background (older style daemons) |
| `Type=oneshot` | For scripts that run and exit — systemd waits for them to finish |
| `User=` | Run the service as this user (never run production services as root) |
| `EnvironmentFile=` | Load environment variables from a file — great for secrets |
| `Restart=always` | Always restart if the process exits for any reason |
| `Restart=on-failure` | Only restart on non-zero exit codes (not on clean stops) |
| `RestartSec=5` | Wait 5 seconds before restarting — prevents rapid crash loops |
| `WantedBy=multi-user.target` | Start this service in normal multi-user mode (what you almost always want) |

### Celery worker service example

```ini
[Unit]
Description=Celery Worker
After=network.target redis.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
EnvironmentFile=/home/ubuntu/myapp/.env
ExecStart=/home/ubuntu/myapp/venv/bin/celery -A tasks worker --loglevel=info
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## Targets (Runlevels)

Targets are systemd's equivalent of old-school runlevels. You rarely need to touch these but useful to know.

| Command | What it does |
|---|---|
| `systemctl get-default` | Shows the default target the system boots into |
| `sudo systemctl set-default multi-user.target` | Boot into multi-user mode (no GUI) — good for servers |
| `sudo systemctl set-default graphical.target` | Boot into graphical mode |
| `sudo systemctl isolate rescue.target` | Drop into rescue/single-user mode right now |
| `sudo systemctl reboot` | Reboot the system cleanly |
| `sudo systemctl poweroff` | Shut down the system |
| `sudo systemctl suspend` | Suspend to RAM |
| `sudo systemctl hibernate` | Hibernate to disk |

---

## System State & Boot Analysis

| Command | What it does |
|---|---|
| `systemctl is-system-running` | Overall system health — `running`, `degraded`, `maintenance`, etc. |
| `systemd-analyze` | Total boot time — kernel + userspace |
| `systemd-analyze blame` | Lists every service and how long it took to start. Great for finding what's slowing down boot |
| `systemd-analyze critical-chain` | Shows the chain of units that determined total boot time |
| `systemd-analyze plot > boot.svg` | Generates a visual SVG timeline of the boot process |
| `systemctl --failed` | Quick list of everything that failed — run this after a reboot to check server health |

---

## Timers — Cron but systemd

Systemd timers are the modern replacement for cron jobs. More logging, better dependency handling.

### Create a timer

You need two files — a service and a timer.

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Run database backup

[Service]
Type=oneshot
User=ubuntu
ExecStart=/home/ubuntu/scripts/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily at 2am

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true  # run immediately if the last scheduled run was missed

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer

# check timers and when they'll next run
systemctl list-timers
```

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Service won't start | `systemctl status myapp` — read the last few log lines |
| Exit code in logs | `journalctl -u myapp -n 50` — find the actual error before the crash |
| Changes not applying | Did you run `sudo systemctl daemon-reload` after editing the service file? |
| Service starts then immediately stops | Check `Restart=` and look at exit codes in journal. Often a config or permission issue |
| Can't find where logs went | `systemctl cat myapp` — check `StandardOutput` and `StandardError` settings |
| Service runs fine manually but not as a service | Usually an environment variable or path issue. Check `EnvironmentFile=` and use absolute paths in `ExecStart` |
| `Failed to connect to bus` error | You're running `systemctl` without `sudo` for a system service |

### Quick debug sequence

```bash
# 1. what's the current state?
systemctl status myapp

# 2. see full logs including the crash
journalctl -u myapp -n 100

# 3. check the service file is correct
systemctl cat myapp

# 4. after fixing — reload daemon, restart service
sudo systemctl daemon-reload
sudo systemctl restart myapp

# 5. verify it's running
systemctl is-active myapp
```

---

## Contributing

If something's wrong or missing — PRs welcome.

---

*Last updated: 2025*
