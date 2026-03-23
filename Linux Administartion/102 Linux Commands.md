Notes 2

## 1. Setup - Getting Into The Server

### Generating SSH Keys

We follow this since its standard practice. In office environments you often can't generate keys on the spot so you generate your own and import them.

Open Git Bash and run:
```bash
ssh-keygen -f keys
```
- `ssh-keygen` is the tool that generates SSH key pairs
- `-f keys` means filename. This tells it to name the output files `keys`. Without `-f` it defaults to `id_rsa`

This generates two files in whatever directory you ran it from:
- `keys` is the private key. This stays on your machine. Never share this with anyone, never upload it anywhere. Think of it as your house key.
- `keys.pub` is the public key. This is what you give to servers. Think of it as the lock on your door, you can share it freely.

Keys are sitting at:
```
/c/Users/mohit.kallam/Documents/daws/keys
```

### Importing to AWS
- Go to AWS Console, EC2, Key Pairs, Import Key Pair
- Paste the contents of `keys.pub`
- Name it and save it
- When launching the EC2 instance, select this keypair

### Connecting to the Instance

Open Git Bash and navigate to the keys folder first:
```bash
cd /c/Users/mohit.kallam/Documents/daws/keys
```
The reason to cd here first is because the SSH command needs the private key file and its easiest to run it from the same directory the key is in. Otherwise you have to give the full path to the key.

Then connect:
```bash
ssh -i keys ec2-user@<your-ip-address>
```
- `ssh` is Secure Shell, an encrypted remote login protocol
- `-i keys` means identity file. Tells SSH which private key to use for authentication. Without this SSH has no way to prove your identity to the server
- `ec2-user` is the default user AWS creates on Amazon Linux instances
- `@<ip-address>` is the public IP of the EC2 instance grabbed from the AWS console

Instance details used:
- AMI: Amazon Linux 2023, kernel 6.1
- Architecture: x86 64-bit, UEFI
- Virtualization: HVM
- Root device: EBS

### Switching to Root

Once inside, you land as `ec2-user`. To get full root access:
```bash
sudo su -
```
- `sudo` means Superuser Do. Lets `ec2-user` run commands as root. Works because AWS pre-adds `ec2-user` to the sudoers list
- `su` means Switch User. Switches to another user, defaults to root when no username is given
- `-` is the login shell flag and it is critical. It loads root's full environment, the correct `$PATH`, `$HOME=/root`, and runs root's profile scripts

Without the `-`:
```bash
sudo su
```
You are technically root but `$HOME` still points to `/home/ec2-user` and `$PATH` might be wrong. This causes subtle bugs where commands are not found or behave unexpectedly. Always use `sudo su -`.

How to know which user you are:
- `$` at end of prompt means normal user
- `#` at end of prompt means root
- `[ec2-user@ip ~]$` means normal user in home directory
- `[root@ip ~]#` means root in `/root`

**Pitfalls:**
- Forgetting to `cd` to the keys folder before SSH gives you "No such file or directory" for the key
- Using `sudo su` instead of `sudo su -` causes environment issues that bite you later
- Closing the terminal without `exit`-ing root first is a bad habit, always exit cleanly

---

## 2. Linux Filesystem Hierarchy (FHS)

Everything in Linux lives under `/` which is the root of the entire filesystem. There is no `C:\` or `D:\` like Windows. One tree, everything hangs off it. This structure is called the Filesystem Hierarchy Standard (FHS) and every Linux distro follows it.

Knowing this is not optional in DevOps. When something breaks you need to know exactly where to look.

---

### `/etc` - Configuration Headquarters

Every service, every tool, every system setting has a config file here. Think of it as Editable Text Configuration.

Important files inside:
```
/etc/passwd          - every user account on the system
/etc/shadow          - actual hashed passwords, only root can read this
/etc/hosts           - manually map IP addresses to hostnames, local DNS
/etc/hostname        - your machine's name
/etc/ssh/sshd_config - SSH server configuration, you will edit this a lot
/etc/sudoers         - who gets sudo access and what they can do
/etc/os-release      - OS and version info, cat this when you forget which Linux you are on
/etc/resolv.conf     - which DNS servers your machine uses to resolve domain names
/etc/fstab           - filesystems to automatically mount at boot
/etc/crontab         - system-wide scheduled jobs
/etc/profile         - environment variables loaded for all users at login
```

`/etc/passwd` in detail:
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
| 2 | `x` | Password placeholder, x means actual password is in `/etc/shadow` |
| 3 | `1000` | User ID (UID) |
| 4 | `1000` | Group ID (GID) |
| 5 | `EC2 Default User` | Description/comment |
| 6 | `/home/ec2-user` | Home directory |
| 7 | `/bin/bash` | Default shell, what shell opens when this user logs in |

UID ranges matter:
- UID `0` is root, the superuser
- UID `1-999` are system/service users created by the OS for internal processes like `nginx`, `mysql`, `sshd`. These are not real humans, they exist so services can run with limited permissions
- UID `1000+` are real human users

This is why in `awk` the filter `$3>=1000` was used to get only real users.

DevOps relevance: Ansible uses `/etc/passwd` to manage users across servers. When you create application-specific users like a `jenkins` user they appear here.

---

### `/var` - Variable Data (Always Changing)

Everything that grows, changes, or accumulates while the system runs lives here.

```
/var/log/   - ALL logs, this is where you live when debugging
/var/www/   - default web root for Apache
/var/lib/   - application state and data
/var/spool/ - queued data like print jobs and mail
/var/tmp/   - temp files that survive reboots
```

`/var/log` in detail, you will use this every single day:
```
/var/log/messages    - general system messages, catch-all log
/var/log/secure      - SSH login attempts, authentication, sudo usage
/var/log/dnf.log     - package installs/updates via dnf/yum
/var/log/boot.log    - what happened during system boot
/var/log/cron        - scheduled job execution logs
/var/log/nginx/      - nginx access and error logs once you install nginx
/var/log/httpd/      - apache access and error logs
/var/log/audit/      - security audit logs
```

`/var/lib` in detail:
```
/var/lib/mysql/   - MySQL/MariaDB database files live here
/var/lib/docker/  - Docker stores ALL container data, images, volumes here
/var/lib/jenkins/ - Jenkins jobs, builds, configs
```

DevOps relevance: When you get into Prometheus and Grafana they write their data to `/var/lib`. When Docker runs out of space you check `/var/lib/docker`. When an app crashes you check `/var/log`.

---

### `/home` - User Home Directories

Every normal user gets their own folder here:
```
/home/ec2-user/ - ec2-user's personal space
/home/mohit/    - if you create a user called mohit
```
Root is special. Root's home is `/root`, not `/home/root`.

What lives inside a user's home:
```
~/.bash_history  - command history
~/.bashrc        - shell configuration loaded on every new terminal
~/.bash_profile  - loaded on login
~/.ssh/          - SSH keys and authorized_keys
```

DevOps relevance: When you set up Ansible you create an `ansible` user. Its home directory and SSH keys live here. Same for Jenkins users and Kubernetes service accounts.

---

### `/root` - Root's Home

Root does not share `/home` with normal users. When you run `sudo su -` and your prompt shows `~`, that `~` resolves to `/root`. Keep this tidy, do not dump random files here.

---

### `/bin` - Essential User Binaries

The most basic commands that every user needs, available even in recovery mode when other filesystems are not mounted yet.
```
ls, cat, cp, mv, rm, mkdir, pwd, echo, grep, cut - all live here
```
These are available to all users.

---

### `/sbin` - System Binaries (Admin Commands)

Like `/bin` but for system administration, mostly used by root:
```
fdisk        - partition disks
ip, ifconfig - network configuration
reboot       - restart the system
shutdown     - shut down
fsck         - filesystem check and repair
```

---

### `/usr` - User System Resources

One of the biggest directories. Think of it as a secondary hierarchy that mirrors root's structure but for installed software.

```
/usr/bin/       - commands installed via package manager like git, wget, curl, tree, python
/usr/sbin/      - admin commands from installed packages
/usr/lib/       - libraries that /usr/bin programs need
/usr/lib64/     - 64-bit libraries, Amazon Linux 2023 uses this heavily
/usr/local/     - software you install manually, not through yum/dnf
/usr/local/bin/ - manually installed binaries go here
/usr/share/     - documentation, man pages, shared data
/usr/share/doc/ - docs for installed packages
```

`/usr/local` is important. When you download Tomcat manually and install it its binaries go in `/usr/local/bin`. Package manager installs go to `/usr/bin`, manual installs go to `/usr/local/bin`.

DevOps relevance: When you get "command not found" errors the first thing to check is whether the binary is in `/usr/bin` or `/usr/local/bin` and whether that path is in your `$PATH` variable.

---

### `/opt` - Optional / Third Party Software

Large third party software that does not follow standard Linux conventions installs its own self-contained directory here.
```
/opt/tomcat/  - manually installed Tomcat
/opt/jenkins/ - Jenkins if installed manually
/opt/aws/     - AWS CLI tools
/opt/splunk/  - Splunk monitoring tool
```

Each application gets its own subdirectory with everything it needs inside, binaries, configs, logs, all self-contained.

DevOps relevance: A huge number of DevOps tools land in `/opt`. When your sir says install this tool, check `/opt` first to see if it is already there.

---

### `/proc` - Process and Kernel Information (Virtual Filesystem)

`/proc` does not actually exist on disk. The kernel generates it in memory and presents it as files. It exposes real-time information about running processes and the kernel itself.

```bash
cat /proc/cpuinfo    # CPU model, cores, speed
cat /proc/meminfo    # RAM total, used, free, cached
cat /proc/uptime     # how long the system has been running
cat /proc/version    # kernel version
ls /proc/            # numbered folders here are process IDs (PIDs)
cat /proc/1/status   # info about process with PID 1 which is systemd
```

Every running process has a folder `/proc/<PID>/` with files describing it, open files, memory maps, the command that started it.

DevOps relevance: Prometheus reads directly from `/proc` to collect system metrics like CPU and memory. When you set up the Node Exporter it is literally reading `/proc` files and exposing them as metrics. Understanding this makes Prometheus make much more sense later.

---

### `/sys` - System and Hardware Information (Virtual Filesystem)

Similar to `/proc` but focused on hardware and kernel subsystems. Also virtual, lives in memory, generated by the kernel.
```
/sys/class/net/ - network interfaces
/sys/block/     - block devices, your disks
```
You can tune kernel behavior by writing values to files here. Advanced stuff you will encounter in performance tuning later.

---

### `/dev` - Device Files

Hardware devices are represented as files in Linux. Your hard disk, keyboard, terminal, all of them are files here.

```
/dev/sda         - your first hard disk
/dev/sda1        - first partition of that disk
/dev/xvda        - disk in AWS EC2 (Xen virtual disk)
/dev/xvdf        - second EBS volume you attach to EC2
/dev/null        - the black hole. Anything written here disappears forever
/dev/zero        - produces infinite stream of zeros, used for wiping disks
/dev/random      - generates random data, used for crypto operations
/dev/tty         - your current terminal
/dev/pts/        - pseudo terminals, SSH sessions show up here
```

`/dev/null` usage you will use constantly:
```bash
command > /dev/null        # discard stdout (normal output)
command 2> /dev/null       # discard stderr (error output)
command > /dev/null 2>&1   # discard ALL output
```

DevOps relevance: When you attach a new EBS volume to your EC2 instance in AWS it appears as `/dev/xvdf` or `/dev/nvme1n1`. You format it, create a filesystem on it, and mount it when setting up servers that need extra storage.

---

### `/tmp` - Temporary Files

Any program can write temp files here. Gets wiped on every reboot. Never store anything important here.

```bash
ls /tmp    # you will often see build artifacts, socket files, lock files
```

DevOps relevance: CI/CD pipelines often write build artifacts here temporarily. Also a common attack vector since malicious scripts get written to `/tmp` because any user can write here.

---

### `/boot` - Boot Files

Everything needed to start the system lives here:
```
/boot/vmlinuz    - the actual Linux kernel binary
/boot/initramfs  - initial RAM filesystem used during boot
/boot/grub/      - GRUB bootloader configuration
```
Do not touch `/boot` unless you know exactly what you are doing. One wrong edit here and your system will not boot. You will deal with this during kernel updates so be careful.

---

### `/lib` and `/lib64` - Shared Libraries

Shared libraries that `/bin` and `/sbin` programs need to run. Like `.dll` files in Windows.
- `/lib64` is for 64-bit libraries, Amazon Linux 2023 uses this
- You rarely touch these directly, the package manager handles them

---

### `/mnt` and `/media` - Mount Points

```
/mnt   - manual mount point. When you attach an EBS volume you mount it here
/media - auto-mounted removable media like USB drives
```

```bash
mount /dev/xvdf /mnt    # attach a disk to /mnt so you can use it
```

DevOps relevance: Every time you add storage to a server, new EBS volume, NFS share, S3 mount, you will be creating mount points and mounting them here or under custom paths.

---

### `/run` - Runtime Data

Stores runtime information since the last boot. PID files, Unix sockets, lock files. Programs write here to say "I am running, here is my process ID."
```
/run/sshd.pid   - SSH daemon's process ID
/run/nginx.pid  - nginx's process ID
```
You will rarely interact with this directly but knowing it exists helps when debugging whether a service is actually running.

---

### Quick Reference - What You Will Actually Use in DevOps

| Directory | When you will use it |
|-----------|----------------------|
| `/etc` | Editing configs for SSH, Nginx, services, users |
| `/var/log` | Debugging, troubleshooting, monitoring |
| `/var/lib` | Database files, Docker data, app state |
| `/opt` | Installing and finding DevOps tools |
| `/usr/local/bin` | Manually installed tool binaries |
| `/proc` | System metrics, Prometheus data source |
| `/dev` | Disk management, attaching EBS volumes |
| `/tmp` | Build artifacts, temp files |
| `/home` | User management, SSH keys |
| `/boot` | Kernel updates, touch carefully |
| `/mnt` | Mounting extra storage volumes |
| `/run` | Checking if services are running |

---

## 3. Navigation and Basic Exploration

These are the commands you will use every single day without thinking about it.

```bash
pwd             # Print Working Directory, tells you exactly where you are
```
Always run this first when confused about where you are. Never assume.

```bash
ls              # list contents of current directory
ls -l           # long format, permissions, owner, size, timestamp, alphabetical
ls -lh          # long format with human readable sizes, shows MB and GB instead of bytes
ls -lt          # long format sorted by newest modified first, latest on top
ls -ltr         # long format sorted by oldest modified first, latest at bottom
ls -la          # long format plus hidden files
ls -lart        # long format plus hidden files sorted oldest first, most complete view
ls -R           # recursive, shows contents of all subdirectories too
ls -ld dirname  # info about the directory itself, not its contents
```

Understanding `ls -l` output:
```
drwxr-xr-x  2 ec2-user ec2-user 4096 Mar 11 10:30 adir
-rw-r--r--  1 ec2-user ec2-user   23 Mar 11 10:31 a.txt
```
- First character: `d` means directory, `-` means file
- Next 9 characters: permissions, rwx for owner, group, others
- `ec2-user ec2-user` is owner and group
- `4096` is size in bytes
- `Mar 11 10:30` is last modified timestamp

Hidden files: Anything starting with `.` is hidden like `.bashrc`, `.bash_history`, `.ssh/`. Normal `ls` will not show them. Use `ls -la` or `ls -lart`.

```bash
cd              # go to your home directory from wherever you are
cd ..           # go one level up
cd /var/log     # go to absolute path
cd adir         # go into adir using relative path, only works if adir is in current directory
clear           # clears terminal screen, history is still there
history         # shows every command you ran this session
history | grep ssh      # search your history for a specific command
!52             # re-run command number 52 from history
```

Ctrl+R is reverse search through history. Press Ctrl+R, start typing, and it shows the most recent matching command you ran. Very useful when you remember running something but not the exact command.

**Pitfalls:**
- Running `ls` and missing hidden files, use `ls -la` when something seems missing
- `ls -lt` shows newest on top and `ls -ltr` shows newest at bottom, easy to mix up, remember `r` means reverse
- Not running `pwd` when confused and running commands in the wrong directory
- `clear` feels like it deleted history but it did not. Run `history` to confirm

---

## 4. File and Directory CRUD

### Creating

```bash
touch a.txt                     # creates empty file, if file exists it just updates timestamp
touch adir/a.txt adir/aa.txt    # create multiple files at once
mkdir adir                      # create a directory
mkdir -p adir/subdir/deepdir    # create nested directories in one shot
                                # -p means parents, creates all intermediate dirs too
mkdir bdir/{b,bb,bbb}           # brace expansion, creates bdir/b bdir/bb bdir/bbb at once
```

### Reading
```bash
ls -R                           # see all files recursively
tree                            # visual tree structure, needs yum install tree -y
```

### Updating, Moving, Copying
```bash
cp adir/a.txt bdir              # copy a.txt from adir into bdir
cp adir/a.txt bdir/newname.txt  # copy and rename at destination
cp -r sourcedir destdir         # copy entire directory and everything inside it
cp -p file dest                 # copy and preserve timestamps and permissions
mv oldname.txt newname.txt      # rename a file
mv a.txt /tmp/                  # move a file to /tmp
```

### Deleting
```bash
rm a.txt                        # delete a file
rm -r adir                      # delete directory and everything inside it, r means recursive
rmdir adir                      # delete directory ONLY if it is empty
```

**Pitfalls:**
- `rm -r` has no undo and no confirmation by default. One wrong path and data is gone forever. Be slow and deliberate with this command
- `rmdir` silently fails if directory is not empty, use `rm -r` instead
- `mkdir -p adir/a.txt` creates `a.txt` as a folder not a file. Do not mix up `touch` and `mkdir`
- Your mistake: `mdkir` and `mkdkir`, Linux will say command not found and will not guess. Spell it `mkdir`
- Your mistake: `cp adir\a.txt bdir`, backslash is Windows. Linux only uses forward slash `/`. Always `cp adir/a.txt bdir`

---

## 5. Reading and Writing Files

```bash
cat > a.txt     # create or OVERWRITE a.txt. Type content, press Enter, then Ctrl+D to save
cat >> a.txt    # APPEND to a.txt. Adds to bottom, does not touch existing content
cat a.txt       # read and print contents of a.txt to screen
cat -n a.txt    # read and print with line numbers
cat file1 file2 # read and print multiple files one after another
```

`>` and `>>` are redirection operators.

`>` takes the output of something and throws it into a file:
```bash
echo "hello" > a.txt    # writes "hello" into a.txt, overwrites if exists
ls -l > output.txt      # saves ls output to a file instead of screen
```

`>>` does the same but appends:
```bash
echo "world" >> a.txt   # adds "world" to bottom of a.txt, keeps existing content
```

**Pitfalls:**
- `>` is destructive and silent. `cat > a.txt` on a file with 500 lines of work means it is gone with no warning and no undo
- Forgetting Ctrl+D after typing into `cat >` leaves your terminal sitting there waiting for more input and nothing is saved yet
- Using `cat` to read a 10,000 line log file dumps everything at once and floods your terminal. Use `head`, `tail`, or `less` instead
- `>>` to a file that does not exist creates the file, that is fine just know it does that

---

## 6. wget and curl

Both fetch things from the internet but they behave very differently.

### wget - Downloads Files to Disk
```bash
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.113/bin/apache-tomcat-9.0.113.tar.gz
```
- Connects to the URL
- Downloads and saves the file to your current directory
- Shows progress bar, file size, speed
- After it finishes the file physically exists on disk, `ls` and you will see it

Think of `wget` as a download manager. You run it, file appears on disk.

Useful flags:
```bash
wget -O myfile.tar.gz <url>     # save with a custom filename
wget -c <url>                   # resume an interrupted download, c means continue
wget -q <url>                   # quiet mode, no progress output, useful in scripts
wget -b <url>                   # download in background
```

### curl - Transfers Data, Prints to Screen
```bash
curl https://raw.githubusercontent.com/daws-88s/notes/main/session-03.txt
```
- Fetches content from URL
- Prints it directly to your terminal
- Nothing is saved unless you tell it to

To save with curl:
```bash
curl -O <url>               # save with original filename, capital O
curl -o myfile.txt <url>    # save with custom filename, lowercase o
curl -I <url>               # show only response headers, is the server up?
curl -L <url>               # follow redirects
```

### The GitHub Raw URL Mistake

When you copy a GitHub file URL it looks like:
```
https://github.com/daws-88s/notes/blob/main/session-03.txt
```
That URL is the GitHub webpage with HTML, navbar, buttons, styling. When you `wget` or `curl` it you get that HTML garbage, not the file content.

The raw URL is:
```
https://raw.githubusercontent.com/daws-88s/notes/main/session-03.txt
```
The difference is `github.com` becomes `raw.githubusercontent.com` and you remove the `/blob` part.

### When to use which in DevOps:
```bash
# wget for downloading binaries, packages, tarballs
wget https://dlcdn.apache.org/tomcat/.../apache-tomcat-9.0.113.tar.gz

# curl for checking if a service or API is responding
curl http://localhost:8080              # is my app running?
curl -I http://google.com              # check response headers
curl -X POST http://api.example.com    # hit a REST API endpoint
```

**Pitfalls:**
- Your mistake: using the `github.com/.../blob/...` URL, always use raw URL for file content
- `curl` without `-O` or `-o` saves nothing to disk, easy to think it downloaded when nothing is there
- `wget` on a webpage URL downloads an HTML file, it worked but the file is useless, always check what you actually downloaded
- Both tools need the exact URL, a typo gives you a 404 page downloaded silently

---

## 7. grep - Searching Inside Files

`grep` searches inside files for a pattern and returns matching lines.

```bash
grep word filename      # basic search
grep case textfile      # find lines containing "case"
```

### Flags:
```bash
grep -i "word" file         # case insensitive, finds Word, WORD, word
grep -v "word" file         # invert, shows lines that do NOT match
grep -n "word" file         # show line numbers with results
grep -c "word" file         # count, just tells you how many lines matched
grep -w "word" file         # whole word match only, grep -w "log" wont match "login" or "syslog"
grep -in "word" file        # combine flags, case insensitive plus line numbers
grep -inc "word" file       # case insensitive plus line numbers plus count
```

### Searching Across Directories:
```bash
grep -r "word" /etc/        # recursive search through entire directory
                            # use this when you dont know which config file has a setting
grep -l "word" /etc/        # only show filenames that match, not the content itself
```

### Context Around Matches (very useful for logs):
```bash
grep -A 2 "error" file      # show 2 lines AFTER each match
grep -B 2 "error" file      # show 2 lines BEFORE each match
grep -C 2 "error" file      # show 2 lines before AND after each match
```
In real work when you find an error line, just that one line is usually not enough. You need the lines around it to understand what caused it. `-C 2` is the one you will reach for most.

### Multiple Patterns:
```bash
grep -E "error|warning" file    # extended regex, match either "error" OR "warning"
                                # very natural when watching logs for multiple things at once
```

### Anchors:
```bash
grep '^word' file       # ^ means start of line, finds lines that START with "word"
grep 'word$' file       # $ means end of line, finds lines that END with "word"
grep '^grep' textfile   # find lines starting with the word grep
```

### Combining with pipe:
```bash
cat /etc/passwd | grep ec2-user     # find ec2-user's entry
grep -i "error" /var/log/messages   # find errors in system log
history | grep wget                 # find all wget commands you ran
```

**Pitfalls:**
- Linux is case sensitive. `grep Error file` will not find "error" or "ERROR". Use `-i` when unsure
- `-v` inverts results. `grep -v f file` shows everything WITHOUT "f". It is exclusion not inclusion, easy to confuse
- Grepping a huge file for a vague term returns too much noise. Use more specific patterns or combine with other commands
- `-r` on `/` searches the entire filesystem which will take forever. Always narrow it down like `grep -r "word" /etc/`

---

## 8. cut and awk - Text Processing

### cut - Slice Columns from Structured Text
```bash
cut -d ":" -f1,3 /etc/passwd
```
- `-d ":"` is the delimiter. Tells cut what character separates columns, here it is `:`
- `-f1,3` means fields. Pick field 1 and field 3
- `/etc/passwd` is the file to process

More examples:
```bash
cut -d "/" -f1-5 linktext       # fields 1 through 5 split by /
cut -d ":" -f1 /etc/passwd      # just usernames, first field only
```

### awk - Powerful Pattern Processing

`awk` is like `cut` but smarter. It can filter AND format in the same command.

```bash
awk -F "/" '{print $NF}' linktext
```
- `-F "/"` is the field separator, like `-d` in cut
- `'{print $NF}'` prints the last field. `$NF` means Number of Fields which equals the last one
- `$1` is first field, `$2` is second, `$NF` is last

```bash
awk -F ":" '{print $NF}' /etc/passwd                # print last field which is the shell for every user
awk -F ":" '$3>=1000{print $1,$3}' /etc/passwd      # only print users with UID >= 1000
awk '{print NR, $0}' file                           # NR is line number, $0 is the entire line
```

The `$3>=1000` one reads as: split by `:`, if field 3 is greater than or equal to 1000, print field 1 and field 3. This filters out system users with UID below 1000 and shows only real human users.

### Pipe - Chaining Commands

The pipe `|` sends the output of one command as input to the next:
```bash
cat /etc/passwd | cut -d ":" -f1,3         # cat output goes into cut
cat linktext | awk -F "/" '{print $NF}'    # cat output goes into awk
head -n 13 textfile | tail -n 10           # get lines 4 to 13
history | grep ssh                         # search history for ssh commands
```
You can chain as many commands as you want with pipes.

**Pitfalls:**
- Your mistake: `cut -d ":" -f1,f3`, it is just `-f1,3`, no `f` before field numbers
- Your mistake: `cut d ":" -f1,3`, missing the `-` before `d`. It is `-d` not `d`
- Your mistake: `/etc/passed`, it is `/etc/passwd`. Linux will not autocorrect
- `cut` only works well on consistently delimited text. Messy or inconsistent files need `awk`
- `$NF` in awk is the last field and `$1` is the first. They are not the same thing

---

## 9. Log Viewing

### head and tail
```bash
head textfile           # first 10 lines by default
tail textfile           # last 10 lines by default
head -n 20 textfile     # first 20 lines
tail -n 20 textfile     # last 20 lines
```

### Specific Line Range Trick
```bash
head -n 13 textfile | tail -n 10
```
This gets lines 4 through 13. How it works:
- `head -n 13` gives you first 13 lines
- `| tail -n 10` from those 13 gives you last 10 which equals lines 4 to 13

### tail -f - Live Log Following
```bash
tail -f /var/log/dnf.log
```
- `-f` means follow. Keeps the file open and prints new lines as they are written
- Press Ctrl+C to stop following

This is the most important log command in DevOps. When you are deploying something and want to watch for errors in real time, `tail -f` is the tool.

### Which logs to watch in DevOps:
```bash
tail -f /var/log/messages           # general system activity
tail -f /var/log/secure             # SSH logins, failed auth attempts
tail -f /var/log/dnf.log            # package installation progress
tail -f /var/log/nginx/error.log    # nginx errors once installed
tail -f /var/log/httpd/error_log    # apache errors once installed
```

**Pitfalls:**
- Your mistake: `tail -n dnf.log`, `-n` expects a number before the filename. `dnf.log` gets treated as the number and it errors out. Correct is `tail -n 20 dnf.log`
- Forgetting `-f` and wondering why logs are not updating. `tail` without `-f` is a one-time snapshot not live
- `tail -f` keeps running forever until Ctrl+C. If you forget it is running it keeps the file handle open
- Catting huge log files instead of tailing them. Use `tail -n 100` to see recent entries

---

## 10. Miscellaneous

### tree - Visual Directory Structure
```bash
yum install tree -y     # install it first, not included by default
tree                    # show full tree from current directory
tree -L 2               # limit depth to 2 levels
```
- `yum` is the package manager for Amazon Linux and RHEL-based systems, newer versions use `dnf` with the same syntax
- `-y` automatically says yes to the install prompt, without it you have to type `y` manually which matters in scripts

### uname - OS and Kernel Info
```bash
uname           # just prints "Linux"
uname -a        # all info, kernel version, hostname, architecture, date
uname -r        # just the kernel version
```

### man - The Manual
```bash
man ls      # full manual for ls command
man grep    # manual for grep
man cut     # manual for cut
```
- Opens in a pager, use arrow keys or Page Up/Down to scroll
- Press `q` to quit, not Ctrl+C
- Press `/` to search within the manual page

**Pitfalls:**
- `tree` on a directory with thousands of files floods your terminal. Always use `tree -L 2` to limit depth first
- `man` quits with `q` not Ctrl+C. Ctrl+C sends an interrupt signal, `q` closes the pager cleanly
- On Amazon Linux 2023 the actual package manager is `dnf` but `yum` still works as it is aliased. You will use both interchangeably

---

## 11. Common Mistakes - Quick Reference

Every mistake from this session in one place so future you does not repeat them:

| Mistake | What was typed | What it should be |
|---------|---------------|-------------------|
| Typo | `mdkir adir` | `mkdir adir` |
| Typo | `mkdkir adir` | `mkdir adir` |
| Wrong path separator | `cp adir\a.txt bdir` | `cp adir/a.txt bdir` |
| Wrong field format in cut | `cut -d ":" -f1,f3` | `cut -d ":" -f1,3` |
| Missing flag dash in cut | `cut d ":" -f1,3` | `cut -d ":" -f1,3` |
| Typo in filename | `/etc/passed` | `/etc/passwd` |
| tail -n without number | `tail -n dnf.log` | `tail -n 20 dnf.log` |
| Wrong GitHub URL | `github.com/.../blob/...` | `raw.githubusercontent.com/.../...` |
| cat without Ctrl+D | Typed into `cat >` and terminal hung | Type content then press Ctrl+D to save |
| wget on webpage | Downloaded HTML instead of actual file | Use the raw or direct file URL |

---

*These notes cover Linux Administration Session 1. Next up: more Linux, then Ansible, Terraform, CI/CD, Kubernetes, Grafana, Prometheus.*
