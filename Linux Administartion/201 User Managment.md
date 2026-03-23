 Session Notes 4


## 1. Core Concepts

Before touching any commands, the mental model is this:

- A **user** is an individual account on the system
- A **group** is a collection of users, usually representing a team or a role
- A **role** is a position, like developer or admin
- A **permission** is the access level tied to that role, what they can read, write, or execute

So the hierarchy flows like: user belongs to a group, the group maps to a role, the role determines what permissions apply. This is why we manage access at the group level instead of setting permissions per user individually. It scales better and is easier to audit.

---

## 2. Creating a User

```bash
# Create a new user
sudo useradd mohit
```

When you run `useradd`, Linux automatically creates a primary group with the same name as the user. So `useradd mohit` creates both a user called `mohit` and a group called `mohit`. The user is assigned to that group by default.

```bash
# View all users on the system
cat /etc/passwd
```

`/etc/passwd` is the file Linux uses to store user account information. Every user on the system gets an entry here. Each line has fields like username, UID, GID, home directory, and default shell, all colon separated.

```bash
# View all groups on the system
cat /etc/group
```

`/etc/group` stores all group information, including which users belong to each group. When you want to verify that a user was added to a group correctly, this is where you check.

```bash
# Check a specific user's UID, GID, and group memberships
id mohit
```

`id` gives you a quick summary of the user's identity: their user ID, primary group ID, and all supplementary groups they belong to. Useful for confirming group changes took effect.

> **Pitfalls**
> - `/etc/passwd` does not store passwords despite the name. Passwords are stored in `/etc/shadow`, which is only readable by root.
> - Do not confuse `/etc/passwd` with `/etc/users`. There is no `/etc/users` file. Trying to cat it will just give you an error.
> - `useradd` and `adduser` are different. `useradd` is the low level command. `adduser` is a higher level wrapper available on some distros that prompts you interactively. On Amazon Linux, stick to `useradd`.
> - By default `useradd` does not create a home directory on all distros. On Amazon Linux 2023 it does, but worth knowing that on some systems you need the `-m` flag explicitly.

---

## 3. Setting a Password

```bash
# Set or change the password for a user
sudo passwd mohit
```

This prompts you to enter and confirm a new password for the user. As root or with sudo, you can set the password for any user without knowing the current one.

One important thing to know: by default on our EC2 instance, only key-based SSH authentication is enabled. Password-based login is turned off in `/etc/ssh/sshd_config`. So even after you set a password with `passwd`, the user cannot SSH in using that password unless you explicitly enable `PasswordAuthentication yes` in the SSH config. We keep it key-based because it is more secure.

> **Pitfalls**
> - Setting a password with `passwd` does not automatically enable password-based SSH login. These are two separate things.
> - If you are root, `passwd mohit` changes mohit's password. If you just run `passwd` with no argument, it changes your own password.

---

## 4. Managing Groups

```bash
# Create a new group
sudo groupadd devops
```

You create groups to represent teams or roles. Once the group exists, you assign users to it.

```bash
# Add a user to a secondary (supplementary) group
sudo usermod -aG devops mohit
```

`usermod` modifies an existing user's properties. The flags here:
- `-a` means append. Without this, `-G` would replace all existing secondary groups with only the ones you specify. You would lose all previous group memberships silently. Always use `-a` with `-G`.
- `-G` specifies the supplementary group to add the user to.

```bash
# Change a user's primary group
sudo usermod -g devops mohit
```

- `-g` (lowercase) changes the primary group. A user can only have one primary group at a time. When the user creates files, the primary group is what gets stamped as the group owner.

```bash
# Remove a user from a group
sudo gpasswd -d mohit dev
```

`gpasswd` manages group passwords and membership. The `-d` flag removes the specified user from the group. This is the standard way to remove someone from a group without deleting them entirely.

> **Pitfalls**
> - Forgetting `-a` when using `usermod -G` is a common and destructive mistake. `usermod -G devops mohit` without `-a` will remove mohit from every other group and leave them only in devops. No warning, no prompt.
> - `-g` and `-G` look almost identical but do completely different things. Lowercase `-g` changes the primary group. Uppercase `-G` sets supplementary groups.
> - After running `usermod -aG`, the change does not apply to the user's current active session. They need to log out and back in, or run `newgrp <groupname>` for it to take effect immediately.

---

## 5. File Permissions

Every file and directory in Linux has three permission sets:
- `u` = user (owner)
- `g` = group
- `o` = others (everyone else)

Each set has three permission bits:
- `r` = read = 4
- `w` = write = 2
- `x` = execute = 1

So `rwx` = 4+2+1 = 7, `rw-` = 4+2 = 6, `r--` = 4.

When you run `ls -l`, you see something like `-rwxr--r--`. The first character is the file type (`-` for file, `d` for directory). Then three characters each for owner, group, and others.

```bash
# Add execute permission for others using symbolic notation
chmod o+x a.txt

# Remove execute permission for others
chmod o-x a.txt

# Add read and write for owner, group, and others
chmod ugo+rw a.txt

# Set permissions using numeric notation: owner=rwx(7), group=rw(6), others=r(4)
chmod 764 a.txt

# Give full permissions to everyone (use with caution)
chmod 777 a.txt
```

`chmod` changes the permission bits on a file or directory. Either the owner of the file or root can run it. Nobody else.

Symbolic notation is more readable when you want to change one specific bit without touching the rest. Numeric notation is faster when you want to set all permissions at once precisely.

> **Pitfalls**
> - `chmod 777` is almost never the right answer in production. It means anyone on the system can read, write, and execute the file. Use it only when you understand exactly why you need it.
> - For directories, the `x` bit means the ability to enter the directory (cd into it), not to run it like a program. A directory without execute permission cannot be traversed even if read is set.
> - `chmod` changes permissions but not ownership. If the wrong user owns the file, changing permissions alone may not solve the access problem.
> - Only the file owner or root can chmod. If you get "Operation not permitted", check who owns the file with `ls -l`.

---

## 6. Ownership

Every file has an owner (a user) and an owning group. When you create a file, it is owned by you and your primary group by default.

```bash
# Change owner and group of a file
sudo chown ec2-user:devops linux

# Recursively change ownership of a directory and everything inside it
sudo chown -R ec2-user:devops linux
```

`chown` changes file ownership. The format is `chown user:group filename`.
- `-R` means recursive, applies the ownership change to the directory and all files and subdirectories inside it.

An important rule: only root can change ownership. Not even the file owner can reassign their own file to another user. This is why `chown` almost always needs `sudo`.

> **Pitfalls**
> - Forgetting `-R` when fixing ownership on a directory means only the directory itself gets updated. All the files inside still have the old ownership.
> - `chown user filename` without specifying a group changes only the user owner and leaves the group as is. `chown user:group filename` changes both.
> - `chown :group filename` with just a colon and group changes only the group without touching the user owner.
> - Running `chown` without sudo as a non-root user will fail even if you own the file.

---

## 7. Sudo and Root Access

Giving a user full sudo access means they can run any command as root. There are two standard ways to do this.

**Option 1: Add to the wheel group**

```bash
# Add mohit to the wheel group
sudo usermod -aG wheel mohit
```

The `wheel` group is a special group on RHEL-based systems (including Amazon Linux) that is pre-configured to have full sudo access. Adding a user to wheel is the quickest way to give them root-level access. We follow this since it is standard practice on these distros.

**Option 2: Create a drop-in sudoers file**

The main sudoers file is `/etc/sudoers`. You should never edit it directly with vim because a syntax error in that file can lock everyone out of sudo permanently. Instead, we use drop-in files in `/etc/sudoers.d/`.

```bash
# Always edit sudoers files using visudo, never vim directly
sudo visudo /etc/sudoers.d/devops
```

`visudo` is a wrapper that opens the file in an editor but validates the syntax before saving. If there is a syntax error, it warns you instead of writing a broken file.

Inside the file, the rule looks like this:

```
devops ALL=(ALL:ALL) NOPASSWD: ALL
```

Breaking this down:
- `devops` is the group this rule applies to (prefix with `%` for groups, but here we named the file devops and the rule targets the devops group)
- `ALL` (first) means this rule applies on all hosts
- `(ALL:ALL)` means the user can run commands as any user and any group
- `NOPASSWD:` means no password prompt when using sudo
- `ALL` (last) means all commands are permitted

```bash
# Check sudoers file syntax without applying changes
sudo visudo -c
```

`visudo -c` runs a syntax check on the sudoers file and tells you if it is valid. Always run this after making changes.

> **Pitfalls**
> - Never edit `/etc/sudoers` directly with vim. If you introduce a syntax error, sudo stops working and you may not be able to fix it without physical or console access to the machine.
> - `/etc/sudoers.d/` files must not have a `.` in the filename (no extensions). A file called `devops.conf` gets ignored by default.
> - After adding a user to the wheel group, they need to log out and back in for it to take effect.
> - `visudo -c` checks syntax but does not validate logic. A rule can be syntactically correct but still give unintended access.

---

## 8. Scoped Command Access (Partial sudo)

Sometimes you do not want to give a user full root access. You want them to be able to run only one specific command with elevated privileges. This is done with a scoped sudoers rule.

```bash
# Inside /etc/sudoers or a drop-in file
mohit ALL=(ALL) /usr/bin/systemctl
```

This means mohit can run `systemctl` with sudo but nothing else. The path must be the full absolute path to the binary. We will go deeper into `systemctl` itself in a later session, but this is how you scope sudo to a single command when you do not want to give someone full root.

> **Pitfalls**
> - Always use the full absolute path in scoped rules. You can find it with `which systemctl`. Using just `systemctl` without the path will not work correctly.
> - Scoped access can still be dangerous if the command itself allows shell escapes or file writes. But for standard use, this is the safe middle ground between no access and full sudo.

---

## 9. Employee Joins: SSH Key Onboarding

When a new employee joins, we do not create a password for them. Instead, they generate their own SSH key pair on their own machine. They send us only the public key. The private key never leaves their machine.

The workflow as admin:

```bash
# Step 1: Create the user
sudo useradd rahim

# Step 2: Switch to root to do the setup
sudo su -

# Step 3: Navigate to the user's home directory
cd /home/rahim

# Step 4: Create the .ssh directory
mkdir .ssh

# Step 5: Create the authorized_keys file inside it
touch .ssh/authorized_keys

# Step 6: Paste the employee's public key into authorized_keys
vim .ssh/authorized_keys

# Step 7: Fix ownership so rahim owns everything inside .ssh
chown -R rahim:rahim .ssh/

# Step 8: Set correct permissions on the .ssh directory
chmod 700 .ssh

# Step 9: Set correct permissions on authorized_keys
chmod 600 .ssh/authorized_keys
```

Why these specific permissions:
- `.ssh/` needs `700` (only the owner can read, write, enter it). If it is world-readable, SSH refuses to use it as a security measure.
- `authorized_keys` needs `600` (owner can read and write, no one else can do anything). Same reasoning, SSH enforces this.
- Ownership must be the user themselves, not root. Even if permissions look right, if root owns the files, SSH will reject them.

If the employee also needs sudo access:

```bash
# Add to devops group which already has a sudoers rule
sudo usermod -aG devops rahim
```

Or create a scoped rule in `/etc/sudoers.d/` as covered in section 8.

> **Pitfalls**
> - If `.ssh/` or `authorized_keys` has permissions that are too open (like 755 or 644), SSH will silently refuse to use the key. The user will not get a useful error, it will just fall back to password auth or deny them entirely.
> - Ownership matters as much as permissions. `chown -R rahim:rahim .ssh/` must be run. If root owns the directory, it will not work even with correct permission bits.
> - The employee must send only the public key (the `.pub` file). If they accidentally send the private key, treat it as compromised and have them regenerate the pair.
> - `authorized_keys` is the exact filename SSH looks for. `authorized_keys2` or `authorizedkeys` will be ignored.

---

## 10. Employee Leaves: Offboarding

When someone leaves, the steps are: remove their access, back up their data, then delete the account.

```bash
# Step 1: Remove them from all secondary groups
sudo gpasswd -d mohit devops
sudo gpasswd -d mohit dev

# Step 2: Reset their primary group back to their own personal group
sudo usermod -g mohit mohit

# Step 3: Archive their home directory before deleting
sudo tar -czf /root/mohit.tar.gz /home/mohit
```

`tar` flags here:
- `-c` creates a new archive
- `-z` compresses it using gzip (produces a `.tar.gz` file)
- `-f` specifies the output filename, must be immediately followed by the filename

To extract later if needed:

```bash
sudo tar -xf mohit.tar.gz
```

- `-x` extracts the archive
- `-f` specifies the filename to extract from

```bash
# Step 4: Check and clean up SSH config if any access was added there
sudo vim /etc/ssh/sshd_config

# Step 5: Test that the SSH config file is valid before logging out
sudo sshd -t

# Step 6: Delete the user and their home directory
sudo userdel -r mohit
```

`sshd -t` is critical. If you edit `sshd_config` and introduce a syntax error, and then log out, nobody will be able to SSH into the server anymore because sshd will fail to start. Run `sshd -t` first. If it comes back clean with no output, the file is valid. Then log out.

`userdel -r` deletes the user account and their home directory together. Without `-r`, the home directory stays behind orphaned.

> **Pitfalls**
> - Never skip `sshd -t` after editing `sshd_config`. A corrupted config means losing SSH access to the server entirely, which on a cloud instance is a serious problem.
> - `userdel` without `-r` leaves the home directory behind. On a shared server this is a storage and security issue.
> - Always back up with `tar` before `userdel -r`. Once the home directory is deleted, it is gone.
> - After `gpasswd -d`, verify with `id mohit` or `cat /etc/group` that they are actually removed from all groups before proceeding.
> - The `-f` flag in tar must be directly followed by the filename with no other flags in between. `tar -czf backup.tar.gz` is correct. `tar -cfz backup.tar.gz` will break because tar tries to use `z` as the filename.

---

## Common Mistakes Quick Reference

| Mistake | What Actually Happens | Correct Approach |
|---|---|---|
| Using `usermod -G devops mohit` without `-a` | Removes mohit from all other groups silently, only devops remains | Always use `usermod -aG` when adding to a group |
| Editing `/etc/sudoers` directly with vim | A syntax error breaks sudo for everyone with no easy fix | Always use `visudo` or `visudo /etc/sudoers.d/filename` |
| Logging out after editing `sshd_config` without testing | If file is corrupted, SSH daemon fails and nobody can connect | Always run `sshd -t` before logging out |
| Running `userdel -r` without backing up home directory | Home directory and all contents are permanently deleted | Always `tar -czf` the home directory first |
| Setting `.ssh/` permissions to 755 or `authorized_keys` to 644 | SSH silently refuses to use the key, login fails | `.ssh/` must be `700`, `authorized_keys` must be `600` |
| Fixing permissions on `.ssh/` but leaving root as owner | SSH rejects the key even if permission bits are correct | Run `chown -R username:username .ssh/` after setup |
| Running `cat /etc/users` instead of `cat /etc/passwd` | No such file, just an error | The correct file is `/etc/passwd` |
| Using `passwd` and assuming SSH password login now works | Password-based SSH is disabled by default in `sshd_config` | Key-based auth is the default, password auth is separate |
| Putting a file extension on a sudoers drop-in file | File gets ignored by sudo silently | Files in `/etc/sudoers.d/` must have no extension |
| Forgetting `-R` in `chown` on a directory | Only the directory itself is updated, contents keep old ownership | Use `chown -R user:group directory/` |