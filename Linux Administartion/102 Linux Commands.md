Session Notes 2

## 1. Setup — Getting Into The Server

### Generating SSH Keys
I didn't use AWS-generated keys. My sir said this is standard practice because in office environments you often can't generate keys on the spot — so you generate your own and import them.

I opened Git Bash on my Windows machine and ran:
```bash
ssh-keygen -f keys
```
- `ssh-keygen` — the tool that generates SSH key pairs
- `-f keys` — `-f` means "filename". This tells it to name the output files `keys`. Without `-f` it defaults to `id_rsa`

This generated two files in whatever directory I ran it from:
- `keys` — the **private key**. This stays on MY machine. Never share this with anyone, never upload it anywhere. Think of it as your house key.
- `keys.pub` — the **public key**. This is what you give to servers. Think of it as the lock on your door — you can share it freely.

My keys are sitting at:
```
C:\Users\mohit.kallam\Documents\daws\keys
```

### Importing to AWS
- Went to AWS Console → EC2 → Key Pairs → **Import Key Pair**
- Pasted the contents of `keys.pub`
- Named it and saved it
- When launching the EC2 instance, selected this keypair

### Connecting to the Instance
I opened Git Bash, navigated to my keys folder first:
```bash
cd C:\Users\mohit.kallam\Documents\daws\keys
```
Why did I cd here first? Because the SSH command needs the private key file and it's easiest to run it from the same directory the key is in. Otherwise you'd have to give the full path to the key.

Then connected:
```bash
ssh -i keys ec2-user@<your-ip-address>
```
- `ssh` — Secure Shell. Encrypted remote login protocol
- `-i keys` — `-i` means "identity file". Tells SSH which private key to use for authentication. Without this SSH doesn't know how to prove your identity to the server
- `ec2-user` — default user AWS creates on Amazon Linux instances. Not root, not admin — specifically `ec2-user`
- `@<ip-address>` — the public IP of your EC2 instance, grabbed from the AWS console

My instance details:
- **AMI:** Amazon Linux 2023, kernel 6.1
- **Architecture:** x86 64-bit, UEFI
- **Virtualization:** HVM
- **Root device:** EBS

### Switching to Root
Once inside, I'm `ec2-user`. To get full root access:
```bash
sudo su -
```
- `sudo` — "Superuser Do". Lets `ec2-user` run commands as root. Works because AWS pre-adds `ec2-user` to the sudoers list
- `su` — "Switch User". Switches to another user, defaults to root when no username given
- `-` — the hyphen is critical. It's a **login shell** flag. It loads root's full environment — the correct `$PATH`, `$HOME=/root`, and runs root's profile scripts

Without the `-`:
```bash
sudo su   # you're root but environment is still ec2-user's
```
You're technically root but `$HOME` still points to `/home/ec2-user`, `$PATH` might be wrong. This causes subtle bugs where commands aren't found or behave unexpectedly. Always use `sudo su -`.

**How you know which user you are:**
- `$` at the end of prompt = normal user
- `#` at the end of prompt = root
- `[ec2-user@ip ~]$` = normal user in home directory
- `[root@ip ~]#` = root in `/root`

**Pitfalls:**
- Forgetting to `cd` to your keys folder before SSH — you'll get "No such file or directory" for the key
- Using `sudo su` instead of `sudo su -` — environment issues will bite you later
- Closing the terminal without `exit`-ing root first — bad habit, always exit cleanly

---

## 2. Linux Filesystem Hierarchy (FHS)

Everything in Linux lives under `/` — the root of the entire filesystem. There's no `C:\` or `D:\` like Windows. One tree, everything hangs off it. This structure is called the **Filesystem Hierarchy Standard (FHS)** and every Linux distro follows it.

Knowing this is not optional in DevOps — when something breaks at 2am you need to know exactly where to look.

---

### `/etc` — Configuration Headquarters
Every service, every tool, every system setting has a config file here. The name historically comes from "et cetera" but practically think of it as **Editable Text Configuration**.

Important files inside:
```
/etc/passwd          — every user account on the system (explained in detail below)
/etc/shadow          — actual hashed passwords, only root can read this
/etc/hosts           — manually map IP addresses to hostnames, local DNS
/etc/hostname        — your machine's name
/etc/ssh/sshd_config — SSH server configuration, you'll edit this a lot
/etc/sudoers         — who gets sudo access and what they can do
/etc/os-release      — OS and version info (cat this when you forget which Linux you're on)
/etc/resolv.conf     — which DNS servers your machine uses to resolve domain names
/etc/fstab           — filesystems to automatically mount at boot
/etc/crontab         — system-wide scheduled jobs
/etc/profile         — environment variables loaded for all users at login
```

**`/etc/passwd` in detail:**
```bash
cat /etc/passwd
```
Each line has 7 fields separated by `:`:
```
ec2-user:x:1000:1000:EC2 Default User:/home/ec2-user:/bin/bash
```
| Field | Example | What it means |
|-------|---------|---------------|
| 1 | `ec2-user` | Username |
| 2 | `x` | Password placeholder — `x` means actual password is in `/etc/shadow` |
| 3 | `1000` | User ID (UID) |
| 4 | `1000` | Group ID (GID) |
| 5 | `EC2 Default User` | Description/comment |
| 6 | `/home/ec2-user` | Home directory |
| 7 | `/bin/bash` | Default shell — what shell opens when this user logs in |

**UID ranges matter:**
- UID `0` = root, the superuser
- UID `1-999` = system/service users — created by the OS for internal processes like `nginx`, `mysql`, `sshd`. These aren't real humans, they exist so services can run with limited permissions
- UID `1000+` = real human users

This is why in `awk` I used `$3>=1000` to filter only real users.

**DevOps relevance:** Ansible uses `/etc/passwd` to manage users across servers. You'll create application-specific users (like a `jenkins` user) and they'll appear here.

---

### `/var` — Variable Data (Always Changing)
Everything that grows, changes, or accumulates while the system runs lives here.

```
/var/log/            — ALL logs. This is where you live when debugging
/var/www/            — default web root for Apache
/var/lib/            — application state and data
/var/spool/          — queued data (print jobs, mail)
/var/tmp/            — temp files that survive reboots
```

**`/var/log` in detail — you'll use this every single day:**
```
/var/log/messages       — general system messages, catch-all log
/var/log/secure         — SSH login attempts, authentication, sudo usage
/var/log/dnf.log        — package installs/updates via dnf/yum (you tail -f'd this)
/var/log/boot.log       — what happened during system boot
/var/log/cron           — scheduled job execution logs
/var/log/nginx/         — nginx access and error logs (once you install nginx)
/var/log/httpd/         — apache access and error logs
/var/log/audit/         — security audit logs
```

**`/var/lib` in detail:**
```
/var/lib/mysql/         — MySQL/MariaDB database files live here
/var/lib/docker/        — Docker stores ALL container data, images, volumes here
/var/lib/jenkins/       — Jenkins jobs, builds, configs
```

**DevOps relevance:** When you get into Prometheus and Grafana, they write their data to `/var/lib`. When Docker runs out of space, you check `/var/lib/docker`. When an app crashes, you check `/var/log`.

---

### `/home` — User Home Directories
Every normal user gets their own folder here:
```
/home/ec2-user/     — ec2-user's personal space
/home/mohit/        — if you create a user called mohit
```
Root is special — root's home is `/root`, NOT `/home/root`.

What lives inside a user's home:
```
~/.bash_history     — command history
~/.bashrc           — shell configuration loaded on every new terminal
~/.bash_profile     — loaded on login
~/.ssh/             — SSH keys and authorized_keys
```

**DevOps relevance:** When you set up Ansible, you create an `ansible` user. Its home directory and SSH keys live here. Kubernetes service accounts, Jenkins users — same story.

---

### `/root` — Root's Home
Root doesn't share `/home` with normal users. When you run `sudo su -` and your prompt shows `~`, that `~` resolves to `/root`. Keep this tidy — don't dump random files here.

---

### `/bin` — Essential User Binaries
The most basic commands that every user needs, available even in recovery/emergency mode when other filesystems aren't mounted yet.
```
ls, cat, cp, mv, rm, mkdir, pwd, echo, grep, cut — all live here
```
These are available to ALL users.

---

### `/sbin` — System Binaries (Admin Commands)
Like `/bin` but for system administration — mostly used by root:
```
fdisk        — partition disks
ip, ifconfig — network configuration
reboot       — restart the system
shutdown     — shut down
fsck         — filesystem check and repair
```

---

### `/usr` — User System Resources
One of the biggest directories. Think of it as a secondary hierarchy — mirrors root's structure but for installed software.

```
/usr/bin/           — commands installed via package manager (git, wget, curl, tree, python)
/usr/sbin/          — admin commands from installed packages
/usr/lib/           — libraries that /usr/bin programs need
/usr/lib64/         — 64-bit libraries (your Amazon Linux 2023 uses this heavily)
/usr/local/         — software you install MANUALLY (not through yum/dnf)
/usr/local/bin/     — manually installed binaries go here
/usr/share/         — documentation, man pages, shared data
/usr/share/doc/     — docs for installed packages
```

**`/usr/local` is important:** When you download Tomcat manually and install it, its binaries go in `/usr/local/bin`. When you compile something from source, it goes here. Package manager installs go to `/usr/bin`, manual installs go to `/usr/local/bin`.

**DevOps relevance:** When you get "command not found" errors, the first thing to check is whether the binary is in `/usr/bin` or `/usr/local/bin` and whether that path is in your `$PATH` variable.

---

### `/opt` — Optional / Third Party Software
Large third party software that doesn't follow standard Linux conventions installs its own self-contained directory here.
```
/opt/tomcat/        — manually installed Tomcat
/opt/jenkins/       — Jenkins if installed manually
/opt/aws/           — AWS CLI tools
/opt/splunk/        — Splunk (monitoring tool)
```

Each application gets its own subdirectory with everything it needs inside it — binaries, configs, logs — self-contained.

**DevOps relevance:** A huge number of DevOps tools land in `/opt`. When your sir says "install this tool", check `/opt` first to see if it's already there.

---

### `/proc` — Process & Kernel Information (Virtual Filesystem)
This is fascinating — `/proc` doesn't actually exist on disk. The kernel **generates it in memory** and presents it as files. It exposes real-time information about running processes and the kernel itself.

```bash
cat /proc/cpuinfo       # CPU model, cores, speed
cat /proc/meminfo       # RAM — total, used, free, cached
cat /proc/uptime        # how long the system has been running
cat /proc/version       # kernel version
ls /proc/               # you'll see numbered folders — those are process IDs (PIDs)
cat /proc/1/status      # info about process with PID 1 (systemd)
```

Every running process has a folder `/proc/<PID>/` with files describing it — open files, memory maps, command that started it.

**DevOps relevance:** Prometheus reads directly from `/proc` to collect system metrics — CPU, memory, process info. When you set up the Node Exporter (Prometheus agent), it's literally reading `/proc` files and exposing them as metrics. Understanding this makes Prometheus make much more sense.

---

### `/sys` — System & Hardware Information (Virtual Filesystem)
Similar to `/proc` but focused on hardware and kernel subsystems. Also virtual, lives in memory, generated by kernel.
```
/sys/class/net/         — network interfaces
/sys/block/             — block devices (your disks)
```
You can actually tune kernel behavior by writing values to files here. Advanced stuff — you'll encounter this in performance tuning.

---

### `/dev` — Device Files
In Linux, **hardware devices are represented as files**. Everything is a file — including your hard disk, keyboard, terminal, and a black hole.

```
/dev/sda            — your first hard disk (SCSI/SATA)
/dev/sda1           — first partition of that disk
/dev/xvda           — disk in AWS EC2 (Xen virtual disk)
/dev/xvdf           — second EBS volume you attach to EC2
/dev/null           — the black hole. Anything written here disappears forever
/dev/zero           — produces infinite stream of zeros, used for wiping disks
/dev/random         — generates random data, used for crypto operations
/dev/tty            — your current terminal
/dev/pts/           — pseudo terminals (SSH sessions show up here)
```

`/dev/null` usage you'll use constantly:
```bash
command > /dev/null          # discard stdout (normal output)
command 2> /dev/null         # discard stderr (error output)
command > /dev/null 2>&1     # discard ALL output
```

**DevOps relevance:** When you attach a new EBS volume to your EC2 instance in AWS, it appears as `/dev/xvdf` or `/dev/nvme1n1`. You format it, create a filesystem on it, and mount it. You'll do this when setting up servers that need extra storage.

---

### `/tmp` — Temporary Files
Any program can write temp files here. Gets **wiped on every reboot**. Never store anything important here.

```bash
ls /tmp             # you'll often see build artifacts, socket files, lock files
```

**DevOps relevance:** CI/CD pipelines often write build artifacts here temporarily. Also a common attack vector — malicious scripts get written to `/tmp` because any user can write here. Something to watch when doing security hardening.

---

### `/boot` — Boot Files
Everything needed to start the system lives here:
```
/boot/vmlinuz       — the actual Linux kernel binary
/boot/initramfs     — initial RAM filesystem used during boot
/boot/grub/         — GRUB bootloader configuration
```
**Don't touch `/boot` unless you know exactly what you're doing.** One wrong edit here and your system won't boot. You'll deal with this during kernel updates — be careful.

---

### `/lib` and `/lib64` — Shared Libraries
Shared libraries that `/bin` and `/sbin` programs need to run. Like `.dll` files in Windows.
- `/lib64` is for 64-bit libraries — your Amazon Linux 2023 uses this
- You rarely touch these directly — package manager handles them

---

### `/mnt` and `/media` — Mount Points
```
/mnt        — manual mount point. When you attach an EBS volume, you mount it here
/media      — auto-mounted removable media (USB drives, CDs)
```

```bash
mount /dev/xvdf /mnt        # attach a disk to /mnt so you can use it
```

**DevOps relevance:** Every time you add storage to a server — new EBS volume, NFS share, S3 mount — you'll be creating mount points and mounting them here or under custom paths.

---

### `/run` — Runtime Data
Stores runtime information since the last boot — PID files, Unix sockets, lock files. Programs write here to say "I'm running, here's my process ID."
```
/run/sshd.pid       — SSH daemon's process ID
/run/nginx.pid      — nginx's process ID
```
You'll rarely interact with this directly but knowing it exists helps when debugging "is this service actually running" questions.

---

### Quick Reference — What You'll Actually Use in DevOps

| Directory | When you'll use it |
|-----------|-------------------|
| `/etc` | Editing configs — SSH, Nginx, services, users |
| `/var/log` | Debugging, troubleshooting, monitoring |
| `/var/lib` | Database files, Docker data, app state |
| `/opt` | Installing and finding DevOps tools |
| `/usr/local/bin` | Manually installed tool binaries |
| `/proc` | System metrics, Prometheus data source |
| `/dev` | Disk management, attaching EBS volumes |
| `/tmp` | Build artifacts, temp files, watch for attacks |
| `/home` | User management, SSH keys |
| `/boot` | Kernel updates — touch carefully |
| `/mnt` | Mounting extra storage volumes |
| `/run` | Checking if services are running |

---

## 3. Navigation & Basic Exploration

These are the commands I'll use every single day without thinking about it.

```bash
pwd                 # Print Working Directory — tells me exactly where I am
```
Always run this first when you're confused about where you are. Never assume.

```bash
ls                  # list contents of current directory
ls -l               # long format — permissions, owner, size, timestamp, alphabetical
ls -lt              # long format, sorted by newest modified first (latest on top)
ls -ltr             # long format, sorted by oldest modified first (latest at bottom)
ls -la              # long format + hidden files
ls -lart            # long format + hidden files + sorted oldest first (most complete view)
ls -R               # recursive — shows contents of all subdirectories too
```

**Understanding `ls -l` output:**
```
drwxr-xr-x  2 ec2-user ec2-user 4096 Mar 11 10:30 adir
-rw-r--r--  1 ec2-user ec2-user   23 Mar 11 10:31 a.txt
```
- First character: `d` = directory, `-` = file
- Next 9 characters: permissions (rwx for owner, group, others)
- Number: hard link count
- `ec2-user ec2-user`: owner and group
- `4096`: size in bytes
- `Mar 11 10:30`: last modified timestamp
- `adir` / `a.txt`: name

**Hidden files:**
Anything starting with `.` is hidden — `.bashrc`, `.bash_history`, `.ssh/`. Normal `ls` won't show them. Use `ls -la` or `ls -lart`.

```bash
cd                  # go to your home directory from wherever you are
cd ..               # go one level up
cd /var/log         # go to absolute path
cd adir             # go into adir (relative — only works if adir is in current directory)
clear               # clears terminal screen, history is still there
history             # shows every command you ran this session
```

**Pitfalls:**
- Running `ls` and missing hidden files — use `ls -la` when something seems missing
- `ls -lt` shows newest on top, `ls -ltr` shows newest at bottom — easy to mix up, remember `r` = reverse
- Not running `pwd` when confused and running commands in the wrong directory
- `clear` feels like it deleted history — it didn't. Run `history` to confirm

---

## 4. File & Directory CRUD

### Creating

```bash
touch a.txt                         # creates empty file. If file exists, just updates timestamp
touch adir/a.txt adir/aa.txt        # create multiple files at once
mkdir adir                          # create a directory
mkdir -p adir/subdir/deepdir        # create nested directories in one shot
                                    # -p means "parents" — create all intermediate dirs too
mkdir bdir/{b,bb,bbb}               # brace expansion — creates bdir/b, bdir/bb, bdir/bbb at once
```

### Reading
```bash
ls -R                               # see all files recursively
tree                                # visual tree structure (needs: yum install tree -y)
```

### Updating / Moving / Copying
```bash
cp adir/a.txt bdir                  # copy a.txt from adir into bdir
cp adir/a.txt bdir/newname.txt      # copy and rename at destination
mv oldname.txt newname.txt          # rename a file
mv a.txt /tmp/                      # move a file to /tmp
```

### Deleting
```bash
rm a.txt                            # delete a file
rm -r adir                          # delete directory and everything inside it (-r = recursive)
rmdir adir                          # delete directory ONLY if it's empty
```

**Pitfalls:**
- `rm -r` has no undo, no confirmation by default. One wrong path and data is gone forever. Be slow and deliberate with this command
- `rmdir` silently fails if directory isn't empty — use `rm -r` instead
- `mkdir -p adir/a.txt` — the `-p` flag creates `a.txt` as a **folder**, not a file. Don't mix up `touch` and `mkdir`
- Your mistake: `mdkir` and `mkdkir` — Linux will say "command not found", it won't guess. Spell it `mkdir`
- Your mistake: `cp adir\a.txt bdir` — backslash `\` is Windows. Linux only uses forward slash `/`. Always `cp adir/a.txt bdir`

---

## 5. Reading & Writing Files

```bash
cat > a.txt         # create or OVERWRITE a.txt. Type content, press Enter, then Ctrl+D to save
cat >> a.txt        # APPEND to a.txt. Adds to bottom, doesn't touch existing content
cat a.txt           # read and print contents of a.txt to screen
```

**`>` and `>>` — Redirection operators:**

`>` is the redirection operator. It takes the output of something and throws it into a file:
```bash
echo "hello" > a.txt        # writes "hello" into a.txt, overwrites if exists
ls -l > output.txt          # saves ls output to a file instead of screen
```

`>>` does the same but appends:
```bash
echo "world" >> a.txt       # adds "world" to bottom of a.txt, keeps existing content
```

**Pitfalls:**
- `>` is destructive and silent. `cat > a.txt` on a file with 500 lines of work = gone. No warning. No undo
- Forgetting `Ctrl+D` after typing into `cat >` — your terminal just sits there waiting for more input. It hasn't saved anything yet
- Using `cat` to read a 10,000 line log file — it dumps everything at once and floods your terminal. Use `head`, `tail`, or `less` instead
- `>>` to a file that doesn't exist — it creates the file. That's fine, just know it does that

---

## 6. wget & curl

Both fetch things from the internet but they behave very differently.

### wget — Downloads Files to Disk
```bash
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.113/bin/apache-tomcat-9.0.113.tar.gz
```
- Connects to the URL
- **Downloads and saves the file** to your current directory
- Shows progress bar, file size, speed
- After it finishes, the file physically exists — `ls` and you'll see it

Think of `wget` as a download manager. You run it, file appears on disk.

Useful flags:
```bash
wget -O myfile.tar.gz <url>     # save with a custom filename
wget -c <url>                   # resume interrupted download (-c = continue)
wget -q <url>                   # quiet mode, no progress output (useful in scripts)
wget -b <url>                   # download in background
```

### curl — Transfers Data, Prints to Screen
```bash
curl https://raw.githubusercontent.com/daws-88s/notes/main/session-03.txt
```
- Fetches content from URL
- **Prints it directly to your terminal**
- Nothing is saved unless you tell it to

To save with curl:
```bash
curl -O <url>                   # save with original filename (capital O)
curl -o myfile.txt <url>        # save with custom filename (lowercase o)
curl -I <url>                   # show only response headers (is the server up?)
curl -L <url>                   # follow redirects
```

### The GitHub Raw URL Mistake
This is important. When you copy a GitHub file URL it looks like:
```
https://github.com/daws-88s/notes/blob/main/session-03.txt
```
That URL is the **GitHub webpage** — HTML, navbar, buttons, styling. When you `wget` or `curl` it, you get that HTML garbage, not the file content.

The **raw URL** is:
```
https://raw.githubusercontent.com/daws-88s/notes/main/session-03.txt
```
Notice: `github.com` → `raw.githubusercontent.com` and remove the `/blob` part.

### When to use which in DevOps:
```bash
# wget — downloading binaries, packages, tarballs
wget https://dlcdn.apache.org/tomcat/.../apache-tomcat-9.0.113.tar.gz

# curl — checking if a service/API is responding
curl http://localhost:8080                   # is my app running?
curl -I http://google.com                    # check response headers
curl -X POST http://api.example.com/data     # hit a REST API endpoint
```

**Pitfalls:**
- Your mistake: using `https://github.com/.../blob/...` URL — always use raw URL for file content
- `curl` without `-O` or `-o` saves nothing — easy to think it downloaded when nothing is on disk
- `wget` on a webpage URL downloads an `.html` file — it worked but it's useless, check what you actually downloaded
- Both tools need the exact URL — a typo in the URL gives you a 404 page downloaded silently

---

## 7. grep — Searching Inside Files

`grep` searches inside files for a pattern and returns matching lines.

```bash
grep word filename                  # basic search
grep case textfile                  # find lines containing "case"
```

### Flags:
```bash
grep -i "word" file         # case insensitive — finds Word, WORD, word
grep -v "word" file         # invert — shows lines that do NOT match
grep -n "word" file         # show line numbers with results
grep -c "word" file         # count — just tells you how many lines matched
grep -in "word" file        # combine flags — case insensitive + line numbers
grep -inc "word" file       # case insensitive + line numbers + count
```

### Anchors:
```bash
grep '^word' file           # ^ = start of line — lines that START with "word"
grep 'word$' file           # $ = end of line — lines that END with "word"
grep '^grep' textfile       # find lines starting with the word grep
```

### Combining with pipe:
```bash
cat /etc/passwd | grep ec2-user     # find ec2-user's entry
grep -i "error" /var/log/messages   # find errors in system log
```

**Pitfalls:**
- Linux is case sensitive — `grep Error file` won't find "error" or "ERROR". Use `-i` when unsure
- `-v` inverts results — `grep -v f file` shows everything WITHOUT "f". Easy to confuse — it's exclusion not inclusion
- Grepping a huge file for a vague term returns too much noise — use more specific patterns or combine with other commands
- `grep` returns exit code 0 if found, 1 if not found — useful in scripts to check if something exists

---

## 8. cut & awk — Text Processing

### cut — Slice Columns from Structured Text
```bash
cut -d ":" -f1,3 /etc/passwd
```
- `-d ":"` — delimiter. Tells cut what character separates columns. Here it's `:`
- `-f1,3` — fields. Pick field 1 and field 3
- `/etc/passwd` — file to process

More examples:
```bash
cut -d "/" -f1-5 linktext       # fields 1 through 5, split by /
cut -d ":" -f1 /etc/passwd      # just usernames
```

### awk — Powerful Pattern Processing
`awk` is like `cut` but smarter. It can filter AND format in the same command.

```bash
awk -F "/" '{print $NF}' linktext
```
- `-F "/"` — field separator (like `-d` in cut)
- `'{print $NF}'` — print the last field. `$NF` = Number of Fields = last one
- `$1` = first field, `$2` = second, `$NF` = last

```bash
awk -F ":" '{print $NF}' /etc/passwd            # print last field (shell) of every user
awk -F ":" '$3>=1000{print $1,$3}' /etc/passwd  # only print users with UID >= 1000
```
That last one reads as: "split by `:`, if field 3 is >= 1000, print field 1 and field 3." This filters out system users (UID < 1000) and shows only real human users.

### Pipe `|` — Chaining Commands
The pipe sends the output of one command as input to the next:
```bash
cat /etc/passwd | cut -d ":" -f1,3         # cat output goes into cut
cat linktext | awk -F "/" '{print $NF}'    # cat output goes into awk
head -n 13 textfile | tail -n 10           # get lines 4-13 (head first 13, tail last 10 of those)
```
You can chain as many commands as you want with pipes.

**Pitfalls:**
- Your mistake: `cut -d ":" -f1,f3` — it's just `-f1,3`. No `f` before field numbers
- Your mistake: `cut d ":" -f1,3` — missing the `-` before `d`. It's `-d`, not `d`
- Your mistake: `/etc/passed` — it's `/etc/passwd`. Linux won't autocorrect
- `cut` only works well on consistently delimited text — messy/inconsistent files need `awk`
- `$NF` in awk is the last field, `$1` is the first — they're not the same thing

---

## 9. Log Viewing

### head and tail
```bash
head textfile               # first 10 lines by default
tail textfile               # last 10 lines by default
head -n 20 textfile         # first 20 lines
tail -n 20 textfile         # last 20 lines
```

### Specific Line Range Trick
```bash
head -n 13 textfile | tail -n 10
```
This gets lines 4 through 13. How:
- `head -n 13` — gives you first 13 lines
- `| tail -n 10` — from those 13, gives you last 10 = lines 4 to 13

### tail -f — Live Log Following
```bash
tail -f /var/log/dnf.log
```
- `-f` means "follow" — keeps the file open and prints new lines as they're written
- Press `Ctrl+C` to stop following

This is the most important log command in DevOps. When you're deploying an app and want to watch for errors in real time — `tail -f` is your tool.

### Which logs to watch in DevOps:
```bash
tail -f /var/log/messages           # general system activity
tail -f /var/log/secure             # SSH logins, failed auth attempts
tail -f /var/log/dnf.log            # package installation progress
tail -f /var/log/nginx/error.log    # nginx errors (once installed)
tail -f /var/log/httpd/error_log    # apache errors (once installed)
```

**Pitfalls:**
- Your mistake: `tail -n dnf.log` — `-n` expects a **number** before the filename. `tail -n dnf.log` treats `dnf.log` as the number and errors out. Correct: `tail -n 20 dnf.log`
- Forgetting `-f` and wondering why logs aren't updating — `tail` without `-f` is a one-time snapshot, not live
- `tail -f` keeps running forever until `Ctrl+C` — if you forget it's running in a session it keeps the file handle open
- Catting huge log files instead of tailing them — use `tail -n 100` to see recent entries

---

## 10. Miscellaneous

### tree — Visual Directory Structure
```bash
yum install tree -y         # install it first — not included by default
tree                        # show full tree from current directory
tree -L 2                   # limit depth to 2 levels
```
- `yum` is the package manager for Amazon Linux / RHEL-based systems (newer versions use `dnf`, same syntax)
- `install tree` — the package to install
- `-y` — automatically say yes to the "do you want to install?" prompt. Without it you'd have to type `y` manually — useful when scripting

### uname — OS and Kernel Info
```bash
uname           # just prints "Linux"
uname -a        # all info — kernel version, hostname, architecture, date
uname -r        # just the kernel version
```

### man — The Manual
```bash
man ls          # full manual for ls command
man grep        # manual for grep
man cut         # manual for cut
```
- Opens in a pager — use arrow keys or Page Up/Down to scroll
- Press `q` to quit — NOT Ctrl+C
- `/` to search within the manual page

**Pitfalls:**
- `tree` on a directory with thousands of files floods your terminal — always use `tree -L 2` to limit depth first
- `man` quits with `q`, not `Ctrl+C`. Ctrl+C sends an interrupt signal, `q` closes the pager cleanly
- `yum` vs `dnf` — on Amazon Linux 2023 the actual package manager is `dnf` but `yum` still works as it's aliased. You'll use both interchangeably

---

## 11. Common Mistakes — Quick Reference

Every mistake I made this session, compiled in one place so future me doesn't repeat them:

| Mistake | What I did | What it should be |
|---------|-----------|-------------------|
| Typo | `mdkir adir` | `mkdir adir` |
| Typo | `mkdkir adir` | `mkdir adir` |
| Wrong separator | `cp adir\a.txt bdir` | `cp adir/a.txt bdir` |
| Wrong field format in cut | `cut -d ":" -f1,f3` | `cut -d ":" -f1,3` |
| Missing `-` in cut | `cut d ":" -f1,3` | `cut -d ":" -f1,3` |
| Typo in filename | `/etc/passed` | `/etc/passwd` |
| tail -n without number | `tail -n dnf.log` | `tail -n 20 dnf.log` |
| Wrong GitHub URL | `github.com/.../blob/...` | `raw.githubusercontent.com/.../...` |
| cat without Ctrl+D | Typed into `cat >` and terminal hung | Type content then press Ctrl+D to save |
| wget on webpage | Downloaded HTML instead of file content | Use raw/direct file URL |

---

*These notes cover Linux Administration Session 1 — filesystem, navigation, file operations, text processing, log viewing, and downloading. Next up: more Linux → Ansible → Terraform → CI/CD → Kubernetes → Grafana → Prometheus.*
