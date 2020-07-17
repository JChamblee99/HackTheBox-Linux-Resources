# Root

 - [User Permissions](Root.md#user-permissions)
 - [File Permissions](Root.md#file-permissions)

## User Permissions

```
Command Breakdown:
    sudo: Super user do
    -l: List the allowed permissions
```

```console
user@parrot:~$ sudo -l
Matching Defaults entries for user on parrot:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User user may run the following commands on parrot:
    (ALL) NOPASSWD: ALL
```

This is an example of the worst configuration for the sudo permissions;  
with this configuration, you can sudo anything without needing a password.

There are other configurations you're more likely to see,  
like specifying only being allowed to sudo a specific command,  
or using a command in a specific directory. 

### Groups

```console
user@parrot:~$ id
uid=1000(user) gid=1000(user) groups=1000(user)
```

Checking `id`, you can see what groups you're in and the respective ID's.  
If you're given a group membership that you shouldn't have,  
you could be given privileges equivalent of root for a command.

### GTFOBins

```console
user@parrot:~$ sudo find . -exec /bin/sh \; -quit
# id
uid=0(root) gid=0(root) groups=0(root)
```

Given sudo privileges of a random command and don't know how to get root with it?  
[GTFOBins](https://gtfobins.github.io/) is a great list of binaries and their strange security bypasses.

## File Permissions

```console
user@parrot:~$ ls -l /bin/wget
-rwxr-xr-x 1 root root 536128 Oct 26  2019 /bin/wget
```
Each file has permissions broken into three triads  
followed by the user that owns the file and the group the file belongs to.

| Owner | Group | World | Owner | Group |
|:-----:|:-----:|:-----:|:-----:|:-----:|
| r w x | r - x | r - x | Root  | Root  |

Each triad is then broken down into the individual permissions
 - `r` if reading is permitted, `-` if it is not
 - `w` if writing is permitted, `-` if it is not
 - `x` if executing is permitted, `-` if it is not

### Set User ID (SUID) and Set Group ID (SGID)
Whenever you execute a binary with either mode set, you're effectively executing it as the file's owner or a member of the file's group.  
The best example of this SUID permission is the `passwd` binary.

```console
user@parrot:~$ ls -l /bin/passwd
-rwsr-xr-x 1 root root 63960 Feb  7 08:54 /bin/passwd
```

Represented by the `s` in place of the owner execution `x`, we can find that those binaries have SUID enabled;  
this is what allows you to change your password in the `/etc/shadow` without root privileges.  

When you're looking for root privilege escalation, you might want to search for files that have one of these special permissions.  
Luckily, we can specify file permissions in the `find` command.

```
Command Breakdown:
    find: Search for files in a directory
    /bin/: The directory to search through
    -perm /u=s,g=s: Specifies to look for files with either SUID or SGID in that order
```

```console
user@parrot:~$ find /bin/ -perm /u=s,g=s
/bin/wall
/bin/newgrp
/bin/su
/bin/sudo
/bin/mount
/bin/umount
...
```
