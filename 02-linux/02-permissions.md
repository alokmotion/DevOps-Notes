# File Permissions & Ownership

## Why this matters

"Permission denied" is one of the most common errors you'll hit in DevOps — a script won't run, a container can't write to a volume, SSH refuses your key. All of it is this one topic. Learn it properly once.

## Reading `ls -l`

```
-rwxr-xr--  1  alok  devs  4096  Sep 10 14:22  deploy.sh
│└┬┘└┬┘└┬┘     │     │      │         │            │
│ │  │  │      │     │      │         │            └─ name
│ │  │  │      │     │      │         └─ last modified
│ │  │  │      │     │      └─ size in bytes
│ │  │  │      │     └─ group
│ │  │  │      └─ owner (user)
│ │  │  └─ permissions for OTHERS (everyone else)
│ │  └─ permissions for GROUP
│ └─ permissions for the OWNER (user)
└─ file type
```

**File type (first character):**

| Char | Type |
|---|---|
| `-` | regular file |
| `d` | directory |
| `l` | symbolic link |
| `c` | character device (e.g. `/dev/null`) |
| `b` | block device (e.g. a disk) |

**Three permission sets, three permissions each:**

```
 rwx    r-x    r--
 └┬┘    └┬┘    └┬┘
User   Group  Others
```

| Letter | On a file | On a directory ← *this trips people up* |
|---|---|---|
| `r` (read) | View its contents | List its contents (`ls`) |
| `w` (write) | Modify its contents | Create/delete/rename files inside it |
| `x` (execute) | Run it as a program | **Enter it** (`cd`) / access anything inside |
| `-` | Permission absent | |

**The directory gotcha:** to `cd` into a directory you need `x` on it, not `r`. To `ls` it you need `r`. To delete a file, you need `w` on the *directory* containing it — not on the file itself. That's why you can sometimes delete a file you can't edit.

## Octal (numeric) notation

Each permission is a bit:

| Permission | Value |
|---|---|
| `r` read | **4** |
| `w` write | **2** |
| `x` execute | **1** |
| `-` none | 0 |

Add them per group:

| Digit | Binary | Means | Typical use |
|---|---|---|---|
| 7 | rwx | read + write + execute | full control |
| 6 | rw- | read + write | a normal editable file |
| 5 | r-x | read + execute | a runnable script others shouldn't edit |
| 4 | r-- | read only | |
| 0 | --- | nothing | |

**The ones you'll actually use:**

| Octal | Symbolic | Meaning | Use for |
|---|---|---|---|
| `755` | `rwxr-xr-x` | owner full; everyone else read+run | scripts, directories, binaries |
| `644` | `rw-r--r--` | owner read/write; everyone else read | normal files, configs |
| `600` | `rw-------` | owner only | **secrets, private keys, `.env`** |
| `700` | `rwx------` | owner only, executable | private scripts, `~/.ssh/` |
| `777` | `rwxrwxrwx` | everyone can do anything | ⚠️ almost never correct |

**On `777`:** you will find Stack Overflow answers telling you to `chmod 777` to fix a permissions error. It "works" the way removing your front door "fixes" a stuck lock. It means any user or compromised process on that machine can rewrite your file. Find the actual owner/group problem instead.

## chmod — change permissions

**Octal form (absolute — sets exactly this):**
```bash
chmod 755 deploy.sh       # rwxr-xr-x
chmod 644 config.yml      # rw-r--r--
chmod 600 ~/.ssh/id_rsa   # rw------- (SSH REQUIRES this)
chmod -R 755 /var/www     # recursive, applies to everything inside
```

**Symbolic form (relative — adjusts what's there):**
```bash
chmod +x script.sh        # make executable for everyone
chmod u+x script.sh       # executable for the user (owner) only
chmod g-w file.txt        # remove write from group
chmod o-rwx secret.txt    # remove everything from others
chmod a+r public.txt      # add read for all
chmod u=rw,go=r file.txt  # set exactly: user rw, group+others r
```

Who: `u` = user/owner, `g` = group, `o` = others, `a` = all.
Operator: `+` add, `-` remove, `=` set exactly.

`chmod +x script.sh` is the fix for the extremely common `bash: ./script.sh: Permission denied`.

## chown — change ownership

```bash
chown alok file.txt            # change owner
chown alok:devs file.txt       # change owner AND group
chown :devs file.txt           # change group only
chown -R alok:devs /var/www    # recursive
chgrp devs file.txt            # change group (alternative)
```

Changing ownership almost always needs `sudo` — you can't give your file away to another user without root.

**Where you'll need this:** a web server runs as user `www-data`. If your app's files are owned by `root`, nginx can't read them → 403 errors. Fix: `sudo chown -R www-data:www-data /var/www/html`.

## umask — default permissions for new files

New files don't get `777` by default; `umask` subtracts from the base.

```bash
umask          # show current mask, usually 0022
umask 077      # new files become private to you (useful in a secrets script)
```

Base is `666` for files and `777` for directories. With `umask 022`: files → `644`, directories → `755`. That's why a `touch`ed file isn't executable — a sensible safety default.

## sudo and root

**root** (UID 0) can do anything, ignoring all permission checks. **sudo** = "run this one command as root".

```bash
sudo command              # run one command as root
sudo -u www-data command  # run as a specific other user
sudo -i                   # interactive root shell (be careful)
sudo !!                   # re-run the previous command with sudo
```

**Practice:** never log in as root, and never leave a root shell open. Use `sudo` per command — it gives you an audit trail (`/var/log/auth.log`) and a moment's pause before something destructive. Who may use sudo is defined in `/etc/sudoers`, edited only with `visudo` (which syntax-checks before saving — a broken sudoers file can lock you out of your own machine).

## Special permission bits

You'll see these occasionally; know what they are so they don't confuse you.

| Bit | Octal | Shows as | Effect |
|---|---|---|---|
| **SUID** | 4000 | `s` in user's `x` slot | Run the file with the *owner's* privileges. `passwd` uses this to let you edit `/etc/shadow`. |
| **SGID** | 2000 | `s` in group's `x` slot | On a directory: new files inherit the directory's group. Great for shared team folders. |
| **Sticky** | 1000 | `t` in others' `x` slot | In a shared writable dir, only the file's owner may delete their own files. `/tmp` uses this (`drwxrwxrwt`). |

```bash
chmod 1777 /shared       # sticky, like /tmp
chmod 2775 /team-dir     # SGID — group is inherited
find / -perm -4000 2>/dev/null   # audit: list all SUID binaries (a security check)
```

SUID binaries are a classic privilege-escalation route — that `find` command is a real thing security reviewers run.

## Debugging "Permission denied"

Work down this list:

1. `ls -l file` — what are the permissions and who owns it?
2. `whoami` and `id` — who am I, and what groups am I in?
3. Am I the owner, in the group, or "others"? Read the matching triplet.
4. **Check every parent directory** — you need `x` on *each* directory in the path. `namei -l /path/to/file` shows the whole chain at once. This is the one people miss.
5. For scripts: is `x` set? Is the shebang (`#!/bin/bash`) correct?
6. Still stuck? On RHEL/CentOS check SELinux (`getenforce`), on Ubuntu check AppArmor. They can deny access even when the permission bits look perfect.

```bash
id                    # your uid, gid, and all groups
groups                # just the group names
namei -l /var/www/html/index.html   # permissions of every step in the path
```

## Practice

**1. Read permissions before you change any**
```bash
ls -l /etc/passwd     # who owns it? can you write it?
ls -l /etc/shadow     # why is this one different?
ls -ld /tmp           # what's that final 't'?
ls -l /usr/bin/passwd # spot the 's'
```
Explain to yourself why `/etc/shadow` (password hashes) is locked down but `/etc/passwd` is world-readable.

**2. Make a script executable — the classic**
```bash
mkdir -p ~/perm-practice && cd ~/perm-practice
echo '#!/bin/bash' > hello.sh
echo 'echo "Hello, $(whoami)"' >> hello.sh
./hello.sh            # FAILS: Permission denied — look at ls -l to see why
chmod +x hello.sh
ls -l hello.sh        # the x's appeared
./hello.sh            # works
```

**3. Convert between notations** (write your answers, then verify with `chmod` + `ls -l`)
- `rwxr-xr--` → octal?
- `640` → symbolic?
- `chmod 600` on a file — who can read it?
- Which of `755` / `644` should a directory get, and why does the other one break it?

**4. Prove the directory-`x` rule**
```bash
mkdir testdir && echo "secret" > testdir/file.txt
chmod 644 testdir          # read but no execute
ls testdir                 # works? 
cat testdir/file.txt       # fails — you can list it but not enter it
chmod 755 testdir
cat testdir/file.txt       # works now
```
This is the single most confusing part of Linux permissions. Seeing it happen fixes it permanently.

**5. Lock down a secret**
```bash
echo "DB_PASSWORD=hunter2" > .env
ls -l .env                 # default is probably 644 — everyone can read it
chmod 600 .env
ls -l .env
```
Then answer: why does SSH *refuse to work at all* if your private key is `644`?

**6. Clean up**
```bash
ls ~/perm-practice && rm -rf ~/perm-practice
```

---

**Previous:** [Linux basics](01-basics-and-filesystem.md) · **Next:** [Processes & services](03-processes-and-services.md)
