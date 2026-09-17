# Linux Basics & The Filesystem

## Why Linux matters for DevOps

Roughly every server you'll ever touch runs Linux. Every Docker container is a Linux userland. Every CI runner, every Kubernetes node, every cloud VM by default. You don't need to be a kernel developer, but you need to be genuinely comfortable at a shell with no GUI to fall back on.

**Note for macOS users:** macOS is Unix-like, so ~90% of this works identically in Terminal. The differences that will bite you are flagged as `[macOS]` below. When you need real Linux, use Docker (`docker run -it ubuntu bash`) or a cloud VM.

## The filesystem: one tree

Windows has `C:\`, `D:\`. Linux has exactly one tree starting at `/` (root). Everything — including USB drives and other disks — is *mounted* somewhere inside that one tree.

```
/
├── bin      → essential commands (ls, cp, cat)
├── sbin     → system commands, usually root-only (fdisk, iptables)
├── etc      → configuration files. ALL of them. Text files.
├── home     → user home directories (/home/alok)
├── root     → the root user's home (note: NOT /home/root)
├── var      → variable data: logs (/var/log), caches, spool
├── tmp      → temporary files, wiped on reboot, anyone can write
├── usr      → user programs and their data (/usr/bin, /usr/local/bin)
├── opt      → optional / third-party software
├── dev      → device files (disks, terminals) — everything is a file
├── proc     → virtual: live kernel & process info, not on disk
├── sys      → virtual: kernel/hardware interface
├── boot     → kernel and bootloader
├── lib      → shared libraries
└── mnt,/media → mount points for external filesystems
```

**The four you'll actually live in:**

| Directory | Why you'll be there | Example |
|---|---|---|
| `/etc` | Every config file | `/etc/nginx/nginx.conf`, `/etc/hosts` |
| `/var/log` | Every log file — first stop when debugging | `/var/log/syslog`, `/var/log/nginx/error.log` |
| `/home/<user>` | Your files, your SSH keys | `~/.ssh/`, `~/.bashrc` |
| `/usr/local/bin` | Where you install your own tools/scripts | `/usr/local/bin/deploy.sh` |

**Key ideas:**
- **Everything is a file.** Disks, keyboards, network sockets, running processes — all represented as files. This is why the same handful of tools (`cat`, `grep`, `>`) works on everything.
- **Case sensitive.** `File.txt` and `file.txt` are different files. (`[macOS]` — the default macOS filesystem is case-*insensitive*, which hides bugs that then appear in Linux CI. A real gotcha.)
- **No file extensions required.** A file called `script` can be a bash script. Extensions are a convention for humans; Linux looks at the content and the permission bits.
- **Files starting with `.` are hidden.** `ls -a` shows them. Almost all config lives in dotfiles.

## Absolute vs relative paths

```bash
/home/alok/notes/file.txt   # absolute — starts at /, always works from anywhere
notes/file.txt              # relative — depends on where you currently are
```

| Shortcut | Means |
|---|---|
| `.` | current directory |
| `..` | parent directory |
| `~` | your home directory (`/home/alok`) |
| `-` | previous directory (`cd -` toggles back) |
| `/` | root of the filesystem |

**Rule:** in scripts and cron jobs, always use absolute paths. Relative paths break the moment the script runs from a different working directory — a classic source of "it works when I run it but fails in cron".

## Navigation

```bash
pwd                  # print working directory — where am I?
ls                   # list files
ls -l                # long format: permissions, owner, size, date
ls -a                # include hidden (dot) files
ls -lh               # human-readable sizes (4.0K not 4096)
ls -lt               # sort by modification time, newest first
ls -ltr              # ...reversed, so newest is at the BOTTOM (near your cursor)
ls -la /etc          # list another directory without going there

cd /var/log          # change directory (absolute)
cd ../..             # up two levels
cd ~                 # home
cd                   # also home (no argument)
cd -                 # back to the previous directory
```

`ls -ltr` is the one you'll type most in real life: when you're hunting for the newest log file, you want it printed last, right above your prompt.

## Reading files

```bash
cat file.txt              # dump the whole file
cat -n file.txt           # with line numbers
less file.txt             # page through it (q=quit, /=search, G=end, g=start)
head file.txt             # first 10 lines
head -n 50 file.txt       # first 50 lines
tail file.txt             # last 10 lines
tail -n 100 file.txt      # last 100 lines
tail -f /var/log/app.log  # FOLLOW — stream new lines as they're written
tail -f app.log | grep ERROR   # follow, showing only errors
wc -l file.txt            # count lines
```

**`tail -f` is the single most-used debugging command in ops.** You run it, then trigger the bug in another window, and watch the error appear live. Ctrl+C to stop.

Use `less` over `cat` for anything large — `cat` on a 2GB log will flood your terminal and you'll be scrolling for a while.

## Creating & manipulating files

```bash
touch file.txt              # create empty file (or update its timestamp)
mkdir mydir                 # make a directory
mkdir -p a/b/c              # make nested dirs, no error if they exist  ← use this in scripts
cp file.txt backup.txt      # copy
cp -r dir1/ dir2/           # copy a directory (recursive)
cp -a src/ dst/             # archive copy: preserves permissions, timestamps, links
mv old.txt new.txt          # rename
mv file.txt /tmp/           # move
rm file.txt                 # delete a file
rm -r mydir/                # delete a directory and its contents
rm -f file.txt              # force, no prompt, no error if missing
ln -s /path/to/real /path/link   # symbolic link (a shortcut)
```

**`rm -rf` warning:** there is no recycle bin. `rm -rf /` with root privileges destroys the machine. Before running any `rm -rf`, run the same path through `ls` first to confirm you're pointing at what you think you are. Beware a stray space: `rm -rf /tmp /mydata` deletes two things, not one.

## Finding things

```bash
find /var/log -name "*.log"            # by name, recursively
find . -name "*.conf" -type f          # files only (-type d for directories)
find . -mtime -1                       # modified in the last 1 day
find . -size +100M                     # larger than 100MB
find /tmp -name "*.tmp" -delete        # find and delete
find . -name "*.sh" -exec chmod +x {} \;   # run a command on each result

which python3        # where is this command's binary?
whereis nginx        # binary, source, and man page locations
locate filename      # fast, but uses a database (run updatedb first)
```

`find . -size +100M` is your first move when a disk fills up.

## Pipes and redirection — the core Unix idea

Small tools, each doing one thing, chained together. This is *the* concept that makes the shell powerful.

```bash
command > file       # send output to file (OVERWRITES it)
command >> file      # append to file
command < file       # read input from file
command 2> err.log   # redirect errors (stderr) only
command > out.log 2>&1   # redirect both output and errors to one file
command &> out.log       # same thing, shorter (bash)
command > /dev/null 2>&1 # throw everything away (silence it)

command1 | command2  # PIPE: send command1's output into command2's input
```

**Streams:** every command has three: `0` = stdin (input), `1` = stdout (normal output), `2` = stderr (errors). `2>&1` means "send stream 2 to wherever stream 1 is going". `/dev/null` is the black hole — write to it to discard.

Real chains:

```bash
cat access.log | grep "500" | wc -l              # how many 500 errors?
ps aux | grep nginx                              # is nginx running?
history | grep docker                            # what docker command did I run?
cat users.txt | sort | uniq                      # sorted, deduplicated
du -sh * | sort -rh | head -5                    # 5 biggest things here
grep ERROR app.log | tail -20                    # last 20 errors
```

## Disk & system info

```bash
df -h                # disk free, human-readable — is the disk full?
du -sh /var/log      # how big is this directory?
du -sh * | sort -rh  # size of everything here, biggest first
free -h              # RAM usage                    [macOS: use `vm_stat`]
uptime               # how long up, plus load average
uname -a             # kernel and architecture
whoami               # which user am I?
hostname             # machine name
date                 # current date/time
top                  # live process/resource view (q to quit)
```

`df -h` and `du -sh * | sort -rh` are the two-command combo for "the disk is full" — the most common ops incident there is.

## Getting help

```bash
man ls               # full manual (q to quit, / to search)
ls --help            # quick option summary
tldr ls              # practical examples (install separately — genuinely worth it)
type ls              # is it a binary, alias, or shell builtin?
```

## Shortcuts that will save you hours

| Keys | Does |
|---|---|
| `Tab` | Autocomplete. Press constantly. Twice = show all options. |
| `Ctrl+C` | Kill the running command |
| `Ctrl+D` | End of input / logout |
| `Ctrl+L` | Clear screen (same as `clear`) |
| `Ctrl+R` | **Search command history** — type a few letters, it finds the command |
| `Ctrl+A` / `Ctrl+E` | Jump to start / end of line |
| `Ctrl+U` / `Ctrl+K` | Delete to start / to end of line |
| `Ctrl+W` | Delete the word before the cursor |
| `↑` / `↓` | Previous / next command |
| `!!` | Repeat last command — `sudo !!` reruns it with sudo |
| `!$` | Last argument of the previous command |

`Ctrl+R` and `sudo !!` are the two that make you look like you've been doing this for years.

## Practice

Work through these in order. Type them — don't paste.

**1. Explore without changing anything**
```bash
pwd; ls -la ~; ls /etc | head -20; ls -ltr /var/log
```
Look at `/var/log`. Which file was modified most recently? Open it with `less` and quit with `q`.

**2. Build and destroy a tree**
```bash
mkdir -p ~/practice/app/{logs,config}    # brace expansion makes both at once
cd ~/practice/app
echo "server_port=8080" > config/app.conf
echo "started" > logs/app.log
echo "ERROR: db timeout" >> logs/app.log
cat logs/app.log
ls -R ~/practice
```
Now: copy `config/` to `config-backup/`, rename the backup to `config-old/`, then delete it. Verify each step with `ls`.

**3. Redirection**
- Write the output of `ls -la /etc` into `~/practice/etc-list.txt`, then count its lines with `wc -l`.
- Run `ls /this-does-not-exist` and redirect *only* the error into `~/practice/err.log`. Check the file has the error and nothing printed to your screen.
- Append today's `date` to `etc-list.txt` without overwriting it. Confirm with `tail -1`.

**4. Pipes**
- Count how many lines in `/etc/passwd` contain `bash`.
- Find the 5 largest directories inside `/var` (`sudo du -sh /var/* | sort -rh | head -5`).
- Use `history | grep` to find a command you ran earlier today.

**5. Find**
- Find every `.conf` file under `/etc` (expect permission errors — send them away with `2>/dev/null`).
- Find files in your home directory modified in the last day.

**6. Follow a log live**
Open two terminals. In one: `tail -f ~/practice/app/logs/app.log`. In the other: `echo "new line" >> ~/practice/app/logs/app.log`. Watch it appear instantly. This is how you debug a live service.

**7. Clean up**
```bash
ls ~/practice        # LOOK before you delete — always
rm -rf ~/practice
```

---

**Next:** [File permissions & ownership](02-permissions.md)
