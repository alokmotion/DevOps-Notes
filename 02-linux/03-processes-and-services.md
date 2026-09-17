# Processes & Services

## Concept

A **process** is a running program. Every process has:

- **PID** — Process ID, a unique number
- **PPID** — Parent PID, the process that started it
- **Owner** — the user it runs as (this determines what it's allowed to touch)
- **State** — running, sleeping, stopped, zombie

Processes form a tree. `PID 1` is the first process the kernel starts (`systemd` on modern Linux) and it's the ancestor of everything else. If a process's parent dies, PID 1 adopts it.

**Why DevOps cares:** "the app is down" almost always means a process died, is stuck, or is eating all the CPU/RAM. Diagnosing that is this page.

## Viewing processes

```bash
ps aux                    # ALL processes, detailed — the one to memorise
ps aux | grep nginx       # is nginx running?
ps -ef                    # same idea, System V style; shows PPID clearly
ps -ef --forest           # as a tree, showing parent/child
pstree -p                 # tree view with PIDs
pgrep nginx               # just the PIDs matching a name
pgrep -a nginx            # PIDs plus the full command
```

Reading `ps aux` output:

```
USER   PID  %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
www     892  0.4  1.2 728104 49232 ?    Ssl  09:14   0:23 nginx: worker process
```

| Column | Meaning |
|---|---|
| `USER` | who it runs as |
| `PID` | process id — what you pass to `kill` |
| `%CPU` / `%MEM` | percentage of each in use |
| `RSS` | **actual physical RAM used, in KB** — the memory number that matters |
| `VSZ` | virtual memory reserved; usually much larger and less meaningful |
| `STAT` | state (see below) |
| `TIME` | total CPU time consumed |
| `COMMAND` | the command line |

**STAT codes:** `R` running · `S` sleeping (normal, waiting for work) · `D` uninterruptible sleep (usually stuck on disk/network I/O — a bad sign if it persists) · `Z` zombie · `T` stopped · `s` session leader · `+` foreground.

**Zombies (`Z`)** are finished processes whose parent hasn't collected their exit status. They use no resources but consume a PID slot. You can't kill a zombie — it's already dead. Kill or fix the *parent*. A pile of zombies means a buggy parent process.

## Live monitoring

```bash
top                # live view. Press: P=sort by CPU, M=sort by memory, k=kill, q=quit
htop               # much better; install it (sudo apt install htop). Arrow keys, F9 to kill.
watch -n 2 'ps aux --sort=-%mem | head -10'   # rerun every 2s: top 10 memory hogs
uptime             # load average
free -h            # memory usage
```

**Load average** (from `uptime`): three numbers = average number of processes wanting CPU over 1, 5, and 15 minutes.

> Compare against your CPU core count (`nproc`). On a 4-core box: load 4.0 = fully busy. Load 8.0 = twice the work as capacity, things are queuing. Load 1.0 on a 4-core box is fine, but load 1.0 on a 1-core box is fully saturated.

The trend across the three numbers tells you the story: `8.0 2.0 1.0` means a spike just started; `1.0 2.0 8.0` means it's recovering.

## Killing processes

```bash
kill 1234              # polite: sends SIGTERM (15), asks it to shut down cleanly
kill -9 1234           # forceful: SIGKILL, the kernel destroys it immediately
kill -15 1234          # explicit SIGTERM (same as plain kill)
kill -HUP 1234         # SIGHUP (1): many daemons reload their config on this
killall nginx          # kill all processes by name
pkill -f "python app"  # kill by matching the full command line
```

**Signals worth knowing:**

| Signal | Number | Effect |
|---|---|---|
| `SIGTERM` | 15 | "Please stop." Process can catch it, flush data, close connections, exit cleanly. **Default — always try this first.** |
| `SIGKILL` | 9 | "Stop now." Cannot be caught or ignored. No cleanup, possible data corruption. **Last resort.** |
| `SIGHUP` | 1 | Traditionally "terminal closed"; daemons repurpose it as "reload config". |
| `SIGINT` | 2 | What Ctrl+C sends. |
| `SIGSTOP` / `SIGCONT` | 19/18 | Pause / resume a process. |

**Why `kill -9` isn't the default answer:** a database killed with `-9` never flushes its write buffer — you can lose or corrupt data. A web server killed with `-9` drops in-flight requests. Always `kill`, wait a few seconds, and only then `kill -9` if it's genuinely hung.

This is exactly how containers behave too: `docker stop` sends SIGTERM, waits 10 seconds, then SIGKILL. Kubernetes does the same. Your app should handle SIGTERM.

## Foreground, background, jobs

```bash
./long-script.sh &     # start in the background
jobs                   # list this shell's background jobs
fg %1                  # bring job 1 to the foreground
bg %1                  # resume a stopped job in the background
Ctrl+Z                 # suspend the foreground job (then bg or fg it)
Ctrl+C                 # kill the foreground job

nohup ./script.sh &            # keep running after you log out (output → nohup.out)
nohup ./script.sh > out.log 2>&1 &   # ...with output where you want it
```

**The logout problem:** a background job started with `&` dies when your SSH session closes. `nohup` prevents that. For anything interactive or long-running, use `tmux` or `screen` instead — you can detach, log out, come back tomorrow and reattach with the session exactly as you left it.

```bash
tmux new -s deploy     # start a named session
# Ctrl+B then D        → detach (it keeps running)
tmux ls                # list sessions
tmux attach -t deploy  # come back to it
```

Learning tmux is worth an hour of your life. Losing a 3-hour migration because your laptop slept is a lesson you only need once.

## systemd — managing services

A **service** (or daemon) is a long-running background process managed by the system: nginx, docker, sshd, your app. `systemd` is the standard manager on modern Linux (Ubuntu 16+, RHEL 7+, Debian 8+).

`systemctl` controls services. **This is the most important command on this page.**

```bash
sudo systemctl start nginx      # start now
sudo systemctl stop nginx       # stop now
sudo systemctl restart nginx    # stop then start (brief downtime)
sudo systemctl reload nginx     # reload config WITHOUT dropping connections ← prefer this
sudo systemctl status nginx     # is it running? recent logs? ← your first debugging step
sudo systemctl enable nginx     # start automatically at boot
sudo systemctl disable nginx    # don't start at boot
sudo systemctl enable --now nginx   # enable AND start, in one command

systemctl is-active nginx       # prints "active" — useful in scripts
systemctl is-enabled nginx      # will it start at boot?
systemctl list-units --type=service --state=running   # everything currently running
systemctl list-unit-files --state=enabled             # everything set to start at boot
sudo systemctl daemon-reload    # reload systemd after EDITING a unit file
```

**`start` vs `enable` is a classic gotcha:** `start` runs it right now but doesn't survive a reboot. `enable` makes it start at boot but doesn't start it now. You almost always want both — hence `enable --now`. The "service disappeared after the server rebooted" incident is always a missing `enable`.

**`restart` vs `reload`:** `reload` re-reads the config with zero downtime where supported. Use `reload` in production if the service supports it.

### Reading `systemctl status`

```
● nginx.service - A high performance web server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; ...)
   Active: active (running) since Wed 2026-09-10 09:14:02 UTC; 2h 3min ago
 Main PID: 892 (nginx)
    Tasks: 3 (limit: 4915)
   CGroup: /system.slice/nginx.service
           ├─892 nginx: master process
           └─893 nginx: worker process
```

- `Loaded: ... enabled` → will start at boot ✓
- `Active: active (running)` → it's up. Other states: `inactive (dead)`, `failed`, `activating`.
- If it says **`failed`**, the last few log lines are printed right below — read them, the answer is usually there.

### Writing a unit file for your own app

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Application
After=network.target          # start after networking is up

[Service]
Type=simple
User=appuser                  # never run as root
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py    # MUST be an absolute path
Restart=always                # restart it if it crashes
RestartSec=5                  # wait 5s between restart attempts
Environment="PORT=8080"
EnvironmentFile=/opt/myapp/.env    # or load vars from a file

[Install]
WantedBy=multi-user.target    # what "enable" hooks it into
```

Then:
```bash
sudo systemctl daemon-reload         # required after creating/editing a unit file
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

`Restart=always` is why systemd is worth using over `nohup` — your app comes back automatically after a crash or a reboot.

## journalctl — reading service logs

systemd captures each service's stdout/stderr into the journal.

```bash
journalctl -u nginx              # all logs for this service
journalctl -u nginx -f           # FOLLOW live (like tail -f) ← use constantly
journalctl -u nginx -n 50        # last 50 lines
journalctl -u nginx --since "10 minutes ago"
journalctl -u nginx --since today
journalctl -u nginx --since "2026-09-10 09:00" --until "2026-09-10 10:00"
journalctl -p err -b             # errors only, this boot
journalctl -u myapp -f | grep -i error
journalctl --disk-usage          # how much space the journal is using
sudo journalctl --vacuum-time=7d # delete journal entries older than 7 days
```

**Debugging a failed service, in order:**
1. `systemctl status myapp` — what does it say and what are those last log lines?
2. `journalctl -u myapp -n 100 --no-pager` — the fuller story
3. Common causes: wrong absolute path in `ExecStart`, the `User` can't read the files, a port already in use, a missing environment variable.

## Practice

**1. Explore what's running**
```bash
ps aux | head -20
ps aux --sort=-%mem | head -5      # 5 biggest memory users — what are they?
ps aux --sort=-%cpu | head -5
pstree -p | head -30               # see the tree from PID 1
nproc && uptime                    # cores vs load average — is this machine busy?
```

**2. Create, find, and kill a process**
```bash
sleep 300 &                  # background job
jobs                         # see it
ps aux | grep sleep          # find its PID
kill %1                      # kill by job number
jobs                         # gone

sleep 300 &
pkill -f "sleep 300"         # kill by command match instead
```

**3. Foreground/background juggling**
```bash
sleep 200          # runs in foreground, blocks your terminal
# press Ctrl+Z     → suspended
jobs               # shows it as "Stopped"
bg                 # resume it in the background
jobs               # now "Running"
fg                 # pull it back to the foreground
# Ctrl+C           → kill it
```

**4. Services** (needs Linux — use `docker run -it ubuntu bash` or a VM if you're on macOS)
```bash
systemctl list-units --type=service --state=running
systemctl status ssh          # or sshd
systemctl is-enabled ssh
journalctl -u ssh -n 30 --no-pager
```
Answer from the output: is SSH running? Will it start at boot? When did it last start?

**5. Write your first unit file** (on a Linux VM)
- Write a script at `/opt/hello/loop.sh` that prints the date every 5 seconds in an infinite loop.
- Make it executable.
- Write `/etc/systemd/system/hello.service` with `Restart=always`.
- `daemon-reload`, `enable --now`, then `journalctl -u hello -f` and watch the output stream.
- Now find its PID and `kill -9` it. Watch systemd bring it straight back. *That's* the value of `Restart=always`.
- Clean up: `sudo systemctl disable --now hello`.

**6. Think it through**
- Why might `kill -9` on a database be a genuinely bad idea?
- A service is `active (running)` but `disabled`. What happens after a reboot?
- Load average is `12.0` — is that a problem? What's the one thing you need to know before you can answer?

---

**Previous:** [Permissions](02-permissions.md) · **Next:** [Users, packages & disks](04-users-packages-disks.md)
