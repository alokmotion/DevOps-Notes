# Users, Packages & Disks

## Part 1 — Users & Groups

### Concept

Every process runs as some user; every file is owned by some user. Permissions are enforced against **UID** (user ID) and **GID** (group ID) numbers — names are just a display convenience mapped in `/etc/passwd`.

**Groups** let you grant a permission to many users at once. A user has one *primary* group and any number of *secondary* groups.

**Service accounts:** in production, apps run as their own dedicated user (`www-data`, `postgres`, `nodeapp`) with no login shell and only the access they need. If the app is compromised, the attacker gets that user's limited access — not root. This is the principle of least privilege, and it's the whole reason this topic matters in DevOps.

### The files

| File | Contains | Readable by |
|---|---|---|
| `/etc/passwd` | usernames, UIDs, home dirs, shells (**no passwords, despite the name**) | everyone |
| `/etc/shadow` | actual password hashes | root only |
| `/etc/group` | group definitions and members | everyone |
| `/etc/sudoers` | who may run sudo, and what | root only |

A `/etc/passwd` line:
```
alok:x:1001:1001:Alok,,,:/home/alok:/bin/bash
 │   │  │    │      │         │          └─ login shell
 │   │  │    │      │         └─ home directory
 │   │  │    │      └─ comment/full name
 │   │  │    └─ primary GID
 │   │  └─ UID  (0 = root; <1000 = system accounts; ≥1000 = humans)
 │   └─ 'x' = password is in /etc/shadow
 └─ username
```

A shell of `/usr/sbin/nologin` or `/bin/false` means **this account cannot log in** — exactly what you want for a service account.

### Commands

```bash
whoami                  # current username
id                      # uid, gid, and all groups
id alok                 # same for another user
groups                  # my groups
who                     # who is logged in right now
last                    # login history
su - alok               # switch user (full login environment)
sudo -u www-data ls /var/www   # run one command as another user

# Managing users (all need sudo)
sudo useradd -m -s /bin/bash alok    # -m creates the home dir, -s sets the shell
sudo adduser alok                    # friendlier interactive wrapper (Debian/Ubuntu)
sudo passwd alok                     # set a password
sudo usermod -aG docker alok         # ADD to a secondary group
sudo userdel -r alok                 # delete user and their home directory

# Groups
sudo groupadd devs
sudo gpasswd -d alok devs            # remove user from a group
getent group docker                  # who's in the docker group?

# Service account (no login) — the production pattern
sudo useradd -r -s /usr/sbin/nologin appuser
```

**The `-a` in `usermod -aG` is critical.** `-a` means *append*. Without it, `usermod -G docker alok` **replaces** all of the user's secondary groups with just `docker` — silently removing them from `sudo` and everything else. People have locked themselves out of servers this way. Always `-aG`.

**Group changes need a new session.** After `usermod -aG docker alok`, log out and back in. Your current shell still holds the old group list — this is why "I added myself to the docker group but still get permission denied" is such a common complaint.

## Part 2 — Package Management

### Concept

A package manager installs software plus everything it depends on, from trusted repositories, and tracks it so you can update or remove it cleanly. Two major families:

| Family | Distros | Tool | Package format |
|---|---|---|---|
| **Debian** | Debian, Ubuntu | `apt` (`dpkg` underneath) | `.deb` |
| **Red Hat** | RHEL, CentOS, Fedora, Amazon Linux | `dnf` / `yum` (`rpm` underneath) | `.rpm` |

You need to recognise both — Ubuntu dominates cloud VMs and Docker images, RHEL/Amazon Linux dominates enterprise.

### apt (Debian/Ubuntu)

```bash
sudo apt update                  # refresh the package LIST (doesn't install anything)
sudo apt upgrade                 # actually upgrade installed packages
sudo apt update && sudo apt upgrade -y      # the standard combo
sudo apt install nginx           # install
sudo apt install -y nginx curl git          # -y auto-confirms (needed in scripts)
sudo apt remove nginx            # remove the package, keep its config
sudo apt purge nginx             # remove package AND config
sudo apt autoremove              # clean up orphaned dependencies
apt search nginx                 # search
apt show nginx                   # package details
apt list --installed             # everything installed
dpkg -l | grep nginx             # same, lower level
dpkg -L nginx                    # which files did this package install?
```

**`update` ≠ `upgrade`.** `update` only refreshes the catalogue of what's available. `upgrade` installs the newer versions. Running `apt install` without a prior `apt update` on a stale image gives you "package not found" or an old version — which is why nearly every Dockerfile starts with `apt-get update && apt-get install -y ...` **in the same RUN line** (separate layers can leave you with a stale cached list).

### dnf / yum (RHEL family)

```bash
sudo dnf install nginx          # yum on older systems; same syntax
sudo dnf update                 # updates the LIST and the packages (unlike apt!)
sudo dnf remove nginx
dnf search nginx
dnf info nginx
rpm -qa | grep nginx            # list installed
rpm -ql nginx                   # files installed by a package
```

Note the trap: `dnf update` does what apt's `upgrade` does. The word means different things in each family.

### Other package managers you'll meet

```bash
brew install <pkg>       # macOS (Homebrew) — no sudo needed
snap install <pkg>       # Ubuntu, self-contained bundles
npm install -g <pkg>     # Node.js tools
pip install <pkg>        # Python (use a venv, not system-wide)
docker pull <image>      # arguably the package manager that won
```

### Version pinning

```bash
sudo apt install nginx=1.18.0-0ubuntu1     # exact version
apt-mark hold nginx                        # prevent it being upgraded
```

**In production, pin your versions.** `apt install nginx` gives a different version depending on the day you run it — which means your Dockerfile builds differently in January and June, and you get a "works on my machine" bug that's genuinely hard to trace. Pin, and upgrade deliberately.

## Part 3 — Disks & Storage

### Concept

**A full disk is one of the most common production incidents there is.** The app stops writing logs, the database refuses writes, and the failure mode is often confusing — the error message rarely says "disk full". Learn the two commands that diagnose it.

### Commands

```bash
df -h                    # disk free per filesystem, human-readable ← START HERE
df -i                    # INODE usage — the other way to run "out of space"
du -sh /var/log          # total size of a directory
du -sh * | sort -rh | head -10       # 10 biggest items here ← the drill-down
du -h --max-depth=1 /var | sort -rh  # size of each subdirectory of /var

lsblk                    # block devices as a tree — disks and partitions
mount | column -t        # what's mounted where
findmnt                  # nicer mount tree
```

**Inodes:** a filesystem has a fixed number of inodes; each file uses one. Millions of tiny files can exhaust inodes while `df -h` still shows free space — you get "No space left on device" with an apparently empty disk. `df -i` is how you spot it. Classic cause: a session or cache directory nobody cleans up.

### The "disk is full" playbook

```bash
df -h                                  # 1. which filesystem is full?
du -h --max-depth=1 / 2>/dev/null | sort -rh | head    # 2. drill down from the top
du -h --max-depth=1 /var | sort -rh | head             # 3. keep drilling
find / -type f -size +500M 2>/dev/null                 # 4. find huge files
sudo lsof | grep deleted                               # 5. deleted-but-still-open files
```

**Step 5 is the one that catches people out.** If a process still holds a file open, deleting it frees *nothing* until the process closes it — the space stays used but the file is invisible to `du`. `df` says full, `du` says there's plenty. The fix is to restart the process holding it (commonly a log file deleted by hand instead of rotated).

Usual culprits: `/var/log` (logs never rotated), Docker (`docker system df`, then `docker system prune`), old kernels, package caches (`apt clean`), core dumps.

### Log rotation

Never let logs grow unbounded. `logrotate` handles this:

```bash
cat /etc/logrotate.conf
ls /etc/logrotate.d/          # per-application configs
sudo logrotate -d /etc/logrotate.d/nginx   # -d = dry run, test without doing it
```

Example `/etc/logrotate.d/myapp`:
```
/var/log/myapp/*.log {
    daily
    rotate 14          # keep 14 days
    compress           # gzip the old ones
    missingok          # don't error if the file isn't there
    notifempty
    copytruncate       # copy then truncate — safe when the app holds the file open
}
```

### Mounting a new disk (the cloud VM workflow)

```bash
lsblk                                    # find it, e.g. /dev/xvdf, no mountpoint
sudo mkfs -t ext4 /dev/xvdf              # format it (DESTROYS DATA — check twice)
sudo mkdir -p /data
sudo mount /dev/xvdf /data               # mount it (this does NOT survive reboot)
df -h                                    # confirm

# Make it permanent — use the UUID, not the device name
sudo blkid /dev/xvdf                     # get the UUID
echo 'UUID=xxxx-xxxx /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
sudo mount -a                            # TEST IT NOW, before you reboot
```

**Two warnings.** Use the **UUID**, not `/dev/xvdf` — device names can change between boots and the machine will fail to boot looking for a disk that moved. And always run `sudo mount -a` after editing `/etc/fstab`: a typo there means the server won't boot, and on a cloud VM with no console that's a very bad afternoon. `nofail` softens this by letting boot continue if the disk is missing.

## Practice

**1. Users**
```bash
id && groups && whoami
who && last | head -5
grep bash /etc/passwd            # which accounts can actually log in?
grep nologin /etc/passwd | head  # service accounts — what are they?
```
Pick one `nologin` account and work out what it's for.

**2. Create a service account** (Linux VM or container)
```bash
sudo useradd -r -s /usr/sbin/nologin testapp
id testapp
sudo su - testapp        # this should FAIL — that's the point
grep testapp /etc/passwd
sudo userdel testapp
```

**3. Groups — safely**
```bash
sudo groupadd testgroup
sudo usermod -aG testgroup $USER
groups                   # NOT there yet — your shell has the old list
newgrp testgroup         # or log out and back in
groups                   # there now
sudo groupdel testgroup
```
Then explain, in your own words, what `usermod -G testgroup $USER` (no `-a`) would have done and why it's dangerous.

**4. Packages**
```bash
apt list --installed | wc -l     # how many packages? (or: rpm -qa | wc -l)
apt show curl                    # read the description and dependencies
dpkg -L curl                     # every file curl installed — where's the binary?
which curl && curl --version
```

**5. Disk investigation — do this on your own machine**
```bash
df -h                                    # any filesystem over 80%?
df -i                                    # inodes healthy?
du -sh ~/* 2>/dev/null | sort -rh | head -10     # your 10 biggest items
find ~ -type f -size +100M 2>/dev/null   # anything huge lurking?
```

**6. Simulate a disk-full investigation**
```bash
mkdir -p ~/disk-practice && cd ~/disk-practice
fallocate -l 200M bigfile.dat      # macOS: mkfile 200m bigfile.dat
ls -lh bigfile.dat
du -sh ~/disk-practice
du -sh ~/* | sort -rh | head -3    # your practice dir should now be near the top
rm bigfile.dat
```

**7. Reason it out**
- `df -h` says 100% full but `du -sh /` totals far less. What's happening and how do you confirm it?
- Why pin package versions in a Dockerfile?
- Why does `/etc/fstab` use UUIDs rather than `/dev/sdb1`?

---

**Previous:** [Processes & services](03-processes-and-services.md) · **Next:** [Text processing](05-text-processing.md)
