# dev-cheatsheets

Commands I kept looking up. Written the way I'd explain them to a teammate — plain English, no jargon, just the stuff that actually comes up.

Built this as a personal reference while working as a backend developer. If you find it useful, feel free to fork it.

---

## Cheatsheets

| File | What's inside |
|---|---|
| [linux-commands.md](./linux-commands.md) | Navigation, permissions, processes, networking, text processing, system admin, performance, security |
| [docker-commands.md](./docker-commands.md) | Images, containers, volumes, networks, compose, Dockerfile patterns, cleanup |
| [nginx-commands.md](./nginx-commands.md) | CLI commands, server blocks, reverse proxy, SSL, logs, performance tuning, security headers |
| [systemctl-commands.md](./systemctl-commands.md) | Service management, journalctl, writing service files, timers, boot analysis |

---

## Who this is for

Developers with a year or two of backend experience who are comfortable with the basics but still google the same flags over and over. Everything here is written for that level — not for absolute beginners, but also not assuming you have the entire man page memorised.

---

## Stack context

Most of this comes from working with:

- Ubuntu servers on AWS (EC2, ECS)
- FastAPI / Python backends
- PostgreSQL, Redis
- Docker + Docker Compose for local and production
- Nginx as a reverse proxy
- Celery workers managed via systemctl

If your stack is similar, this should map pretty directly to your day-to-day.

---

## Coming soon

- `git-commands.md` — beyond the basics (rebase, bisect, reflog, worktrees)
- `aws-cli.md` — EC2, ECS, RDS, S3, IAM from the terminal
- `postgres-commands.md` — psql, explain analyze, common queries, maintenance
- `redis-commands.md` — data types, persistence, debugging memory usage

---

## Contributing

If something's wrong, outdated, or missing — PRs are welcome. Keeping it practical is the goal, so nothing too academic or edge-case-y.

---

*Last updated: 2026*
