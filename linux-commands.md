# Linux Commands I Actually Use

A living document of Linux commands I've picked up over the years working as a backend developer. Nothing fancy — just the stuff that comes up day to day, with notes written the way I'd explain it to a teammate.

Covers everything from basic navigation to the kind of system-level stuff you start needing once you're dealing with real servers, ECS deployments, and debugging things at 2am.

---

## Table of Contents

- [Navigation & File System](#navigation--file-system)
- [Permissions & Ownership](#permissions--ownership)
- [Text Search & Viewing](#text-search--viewing)
- [Process Management](#process-management)
- [Networking](#networking)
- [Disk & Storage](#disk--storage)
- [Compression & Archives](#compression--archives)
- [Text Processing — Pipes & Filters](#text-processing--pipes--filters)
- [System Administration](#system-administration)
- [Performance & Monitoring](#performance--monitoring)
- [Security & SSH](#security--ssh)
- [Developer Tools](#developer-tools)
- [Shortcuts & Power Tricks](#shortcuts--power-tricks)

---

## Navigation & File System

The basics. You use these every single day.

| Command | What it does |
|---|---|
| `pwd` | Print Working Directory — tells you exactly where you are in the file system. Think of it as "You Are Here" on a map |
| `ls` | Lists all files and folders in the current directory |
| `ls -la` | Lists everything including hidden files (the ones starting with `.`), along with permissions, owner, size, and date |
| `cd /path` | Move to a different directory. `cd ..` goes one level up, `cd ~` takes you home |
| `mkdir myfolder` | Creates a new folder. `mkdir -p a/b/c` creates nested folders in one shot |
| `rm file.txt` | Deletes a file permanently — no recycle bin. `rm -rf folder/` nukes a folder and everything inside it |
| `cp a.txt b.txt` | Copies a file. `cp -r src/ dest/` copies an entire folder recursively |
| `mv old.txt new.txt` | Moves or renames a file — same command for both |
| `touch file.txt` | Creates an empty file, or updates the "last modified" timestamp if the file already exists |
| `cat file.txt` | Prints the full content of a file to the screen. Fine for small files |
| `less file.txt` | Like `cat` but paginated — scroll with arrow keys, `q` to quit. Use this for anything bigger than a screen |
| `head -n 20 file` | Shows only the first 20 lines |
| `tail -n 20 file` | Shows only the last 20 lines. `tail -f file` follows in real-time — I use this constantly for watching logs |

---

## Permissions & Ownership

Trips up almost everyone at some point.

| Command | What it does |
|---|---|
| `chmod 755 file` | Sets permissions. `7` = owner can read/write/execute, `5` = group and others can read/execute |
| `chmod +x script.sh` | Makes a script executable — you run this before running any `.sh` file |
| `chown user:group file` | Changes who owns a file. Handy when something is accidentally owned by root |
| `sudo command` | Run a command as administrator. You'll be asked for your password |
| `whoami` | Shows which user you're logged in as — useful when hopping between users |

---

## Text Search & Viewing

| Command | What it does |
|---|---|
| `grep 'text' file` | Search for a pattern in a file. `-r` searches recursively in folders, `-i` ignores case |
| `find . -name '*.py'` | Find files by name from the current directory. Wildcards work: `find . -name '*.log'` |
| `wc -l file.txt` | Counts lines (`-l`), words (`-w`), or characters (`-c`) in a file |
| `diff file1 file2` | Shows line-by-line differences between two files — like a manual `git diff` |
| `echo 'hello'` | Prints text to the screen. `echo $VAR` prints a variable's value. Used a lot in scripts |

---

## Process Management

Stuff you need once you're running real services.

| Command | What it does |
|---|---|
| `ps aux` | Lists all running processes with CPU and memory usage. Pipe it: `ps aux \| grep python` |
| `top` / `htop` | Live process monitor — like Task Manager. `htop` is more visual (install separately) |
| `kill -9 <PID>` | Force-kills a process by its ID. Get the PID from `ps aux`. `-9` means no mercy |
| `pkill nginx` | Kill by process name — easier than hunting for a PID first |
| `bg` / `fg` | `Ctrl+Z` pauses a job, `bg` resumes it in the background, `fg` brings it back to front |
| `jobs` | Lists background or paused jobs in your current terminal session |
| `nohup cmd &` | Runs a command in the background that keeps running after you close the terminal. Output goes to `nohup.out` |
| `screen` / `tmux` | Terminal multiplexers — create persistent sessions that survive disconnections. `tmux` is more modern |

---

## Networking

| Command | What it does |
|---|---|
| `curl -X GET url` | Makes HTTP requests from the terminal. `-H` for headers, `-d` for body data. My go-to for quick API tests |
| `wget url` | Downloads a file from the internet. `wget -O name.zip url` saves it with a specific name |
| `ping google.com` | Tests if a server is reachable and measures response time |
| `ss -tulnp` | Shows which ports are open and which process is listening. Modern replacement for `netstat -tulnp` |
| `ip a` | Shows your network interfaces and IP addresses. Modern version of `ifconfig` |
| `ssh user@host` | Connect to a remote server securely. Add `-i key.pem` to use a private key file |
| `scp file user@host:/path` | Transfer files over SSH — like `cp` but between machines |
| `rsync -avz src dst` | Sync files between locations, only transfers what changed. Much faster than `scp` for large transfers |
| `nslookup domain` | Looks up DNS records — useful for debugging DNS issues |
| `traceroute host` | Shows the network path packets take to reach a server |

---

## Disk & Storage

| Command | What it does |
|---|---|
| `df -h` | Shows disk space usage across all drives in a human-readable format (GB/MB) |
| `du -sh folder/` | Shows how much space a folder is using. `du -sh *` shows all items in the current directory |
| `lsblk` | Lists all disks and partitions on the system in a tree layout |
| `mount` / `umount` | Attach or detach a filesystem (like a USB drive or network share) to a directory |

---

## Compression & Archives

| Command | What it does |
|---|---|
| `tar -czf out.tar.gz dir/` | Creates a compressed archive. `-c` = create, `-z` = gzip, `-f` = filename |
| `tar -xzf file.tar.gz` | Extracts a `.tar.gz` archive. Add `-C /path` to extract to a specific directory |
| `zip -r out.zip dir/` | Creates a `.zip` archive recursively |
| `unzip file.zip` | Extracts a `.zip`. `unzip file.zip -d /dest` to extract to a specific folder |

---

## Text Processing — Pipes & Filters

This is where Linux starts to feel like a superpower.

| Command | What it does |
|---|---|
| `cmd1 \| cmd2` | Pipe — sends output of `cmd1` as input to `cmd2`. Chain as many as you want |
| `cmd > file.txt` | Redirects output to a file (overwrites). `>>` appends. `2>` redirects errors |
| `sort file.txt` | Sorts lines alphabetically. `-n` for numeric, `-r` for reverse |
| `uniq` | Removes consecutive duplicate lines. Usually paired with `sort`: `sort file \| uniq -c` counts occurrences |
| `cut -d',' -f1` | Cuts specific columns. `-d` sets the delimiter, `-f` sets the field number. Great for CSVs |
| `awk '{print $2}'` | Powerful text processor — extracts columns, does calculations. `$2` = second column |
| `sed 's/old/new/g'` | Find and replace in a stream or file. `g` flag replaces all occurrences |
| `tr 'a-z' 'A-Z'` | Translates characters — this example converts lowercase to uppercase |
| `tee file.txt` | Writes to both screen AND a file simultaneously. Great for logging output while watching it live |
| `xargs` | Converts input into arguments for another command. E.g. `find . -name '*.log' \| xargs rm` |

---

## System Administration

The stuff you start needing once you're managing actual servers.

| Command | What it does |
|---|---|
| `systemctl status nginx` | Check the status of a service. `start`, `stop`, `restart`, `enable`, `disable` all follow the same pattern |
| `journalctl -u nginx` | View logs for a specific systemd service. `-f` follows live, `--since today` filters by date |
| `crontab -e` | Edit scheduled tasks. Format: `min hour day month weekday command` |
| `env` / `printenv` | Shows all environment variables. `printenv VAR_NAME` shows a specific one |
| `export VAR=value` | Sets an environment variable for the current session. Add to `~/.bashrc` to make it permanent |
| `source ~/.bashrc` | Reloads your bash config without restarting the terminal — run this after editing `.bashrc` |
| `ulimit -n 65536` | Sets resource limits. `-n` = max open file descriptors. You'll hit this on high-traffic servers |
| `lsof -i :8000` | Shows what process is using port 8000. `lsof -p PID` shows all files a process has open |
| `strace -p <PID>` | Traces system calls of a running process — advanced debugging to see exactly what a process is doing |
| `dmesg \| tail` | Shows kernel messages. Useful for diagnosing OOM kills, hardware issues, driver errors |

---

## Performance & Monitoring

| Command | What it does |
|---|---|
| `vmstat 1` | Shows CPU, memory, and I/O stats every 1 second. Good starting point for performance issues |
| `iostat -x 1` | I/O stats per disk device. `%util` shows how saturated a disk is |
| `sar -u 1 10` | Collects CPU stats over time. Good for historical analysis |
| `free -h` | Shows RAM usage. The `available` column is what actually matters — not `free` |
| `nice -n 10 cmd` | Run a command with lower CPU priority so it doesn't starve other processes |
| `renice 10 -p PID` | Change the priority of an already-running process |
| `perf stat cmd` | Shows CPU cycles, cache misses, branch mispredictions. For when you really need to dig in |

---

## Security & SSH

| Command | What it does |
|---|---|
| `ssh-keygen -t ed25519` | Generates a new SSH key pair. Ed25519 is modern and secure — prefer this over RSA |
| `ssh-copy-id user@host` | Copies your public key to a remote server. Enables passwordless login after this |
| `ufw allow 80/tcp` | UFW (Uncomplicated Firewall) — easy firewall management. `ufw status` shows current rules |
| `iptables -L` | Lists raw firewall rules. More granular than UFW |
| `fail2ban-client status` | Shows fail2ban status — auto-bans IPs that fail too many login attempts |
| `last` / `lastlog` | `last` shows recent logins, `lastlog` shows last login time per user |
| `auditctl -l` | Shows active Linux audit rules — used for tracking file access and user commands |

---

## Developer Tools

| Command | What it does |
|---|---|
| `docker ps -a` | Lists all containers (running and stopped). `docker logs -f name` streams logs |
| `docker exec -it name sh` | Opens a shell inside a running container |
| `docker stats` | Live CPU and RAM usage for all running containers |
| `git log --oneline` | Compact git commit history. `--graph --all` shows branch structure |
| `git stash` / `git stash pop` | Stash saves uncommitted changes temporarily, pop restores them |
| `ldd /path/to/binary` | Lists shared library dependencies — helps debug "library not found" errors |
| `xxd file \| head` | Hex dump of a file — for inspecting binary files or protocol debugging |
| `base64 -d file` | Decodes base64 content. `base64 file` encodes. Shows up everywhere: JWTs, certs, env vars |

---

## Shortcuts & Power Tricks

These save a surprising amount of time.

| Shortcut / Command | What it does |
|---|---|
| `Ctrl + C` | Kill the currently running command — your most-used shortcut |
| `Ctrl + Z` | Pause (suspend) the current process. Use `fg` to bring it back |
| `Ctrl + R` | Reverse search through command history — start typing to find a previous command |
| `!!` | Repeat the last command. `sudo !!` re-runs the previous command as sudo |
| `history \| grep cmd` | Search your history for a specific command you ran before |
| `alias ll='ls -la'` | Create a shortcut. Add to `~/.bashrc` to make permanent |
| `cmd1 && cmd2` | Run `cmd2` only if `cmd1` succeeds |
| `cmd1 \|\| cmd2` | Run `cmd2` only if `cmd1` fails — good for fallbacks |
| `cp file.txt{,.bak}` | Brace expansion — creates `file.txt.bak`. Quick backup without retyping the filename |
| `man command` | Built-in documentation for any command. Press `q` to exit |
| `command --help` | Faster than `man` for a quick usage reference |

---

## Contributing

If something's wrong, missing, or could be explained better — PRs are welcome. This is meant to be useful, not exhaustive.

---

*Last updated: 2026*
