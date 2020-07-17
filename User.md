# User

 - [Basic Enumeration](User.md#basic-enumeration)
 - [Secure Shell (SSH)](User.md#secure-shell-ssh)
 - [Password Cracking](User.md#password-cracking)

## Basic Enumeration
The ability to detect anomalies and find assets in the filesystem is very important at this stage  
since majority of the vectors to user are likely to be backup files, plain text passwords in code,  
and services that are open only on localhost.

```
Command Breakdown:
    ls: List directory contents
    -a: Do not ignore entries starting with `.`
    -l: Use a long listing format (shows permissions and owner)
```

```console
www-data@example~$ ls -al /home
total 16
drwxr-xr-x 1 root root   8 Feb  8 03:18 .
drwxr-xr-x 1 root root 302 Jul  4 13:10 ..
drwxr-xr-x 1 user user 966 Jul 12 16:30 user
```

### Finding a file by its name
```
Command Breakdown:
    find: Search for files in a directory hierarchy
    /var/: The directory to search in
    -iname "*credentials*": Show files that contain the case insensitive word "credentials" anywhere in the name
    2>/dev/null: Redirect standard error away from the screen so that it doesn't display "Permission denied" 
```

```console
www-data@example~$ find /var/ -iname "*credentials*" 2>/dev/null
/var/log/sneakydir/Credentials.tar.gz
```

### Finding a file by its contents
```
Command Breakdown:
    grep: Print lines matching a pattern
    -w: Word regexp, match only if the entire word matches the search
    -i: Ignore case distinction
    -n: Include line number
    -r: Read all files under each directory, recursively.
    /var/www/: Directory to start search
    -e "password": Search pattern
```

```console
www-data@example~$ grep -winr /var/www/ -e "password"
/var/www/html/login.php:8:        if(Password != 'P@5sw0Rd!'){
```

### Internal Network 
```
Command Breakdown:
    netstat: Print network connections and interface information
    -t: Show TCP connections
    -u: Show UDP connections
    -l: Show only listening sockets
    -n: Numeric addresses and ports instead of names
```

```console
www-data@example:~$ netstat -tuln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:9126          0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp6       0      0 :::80                   :::*                    LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN
udp        0      0 127.0.0.53:53           0.0.0.0:*                     
```
Whenever a port is bound to the local address `0.0.0.0` or `:::`, it means that it's bound on every network interface.  
Whenever a port is bound to the local address `127.0.0.1`, it means that it's only accessible internally.  
If I found a normal port open internally like 3306(MySQL), I'd try to find credentials for it and access the database.  
If I found a strange port open internally like 9126, I'd connect to it with `telnet` and see if I can give any commands.  

```
Command Breakdown:
    telnet: User interface to the TELNET protocol
```

```console
www-data@example:~$ telnet localhost 9126
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

version
Example v1.5.3
```

## Secure Shell (SSH)
When you connect to a server over SSH, you'll need some form of authentication;  
this authentication can come in the form of an account password or private key.  
If you can't use an account password, then your option would be to either find the user's private key,  
or to generate your own key and add the public key to the user's authorized keys.

### Generating a private key
```
Command Breakdown:
    ssh-keygen: Authentication key generation
    -t rsa: Specifies the type of key to create
    -b 4096: Generate a 4096 bit length key
```

```console
user@parrot:~$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/home/user/.ssh/id_rsa): 
/home/user/.ssh/id_rsa already exists.
Enter passphrase (empty for no passphrase): !august1993$
Enter same passphrase again: !august1993$
Your identification has been saved in /home/user/.ssh/id_rsa
Your public key has been saved in /home/user/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:7x5Ct7B4LymJHTisHlRT/f12J4ausyesor+md3kOLL8 user@parrot
The key's randomart image is:
+---[RSA 4096]----+
|      ..         |
|     .  .        |
|    o    . .     |
|   . .    . .    |
|  .. .  S .  o   |
| .  + .+ = .. = o|
|  .. +oo*++. o o.|
|  ... B+===.o    |
| .. o*o=EBB*     |
+----[SHA256]-----+

user@parrot:~$ ls -al .ssh
total 12
drwx------ 1 user user   54 Jul 12 22:58 .
drwxr-xr-x 1 user user  928 Jul 12 19:37 ..
-rw------- 1 user user 3381 Jul 13 00:21 id_rsa
-rw-r--r-- 1 user user  737 Jul 13 00:21 id_rsa.pub
```

### Authorized keys
Appending the contents of your public key (id_rsa.pub) to the user's authorized keys  
allows you to use your own key to connect to the target without a password.

```console
user@example:~$ echo "ssh-rsa AAAA ... w== user@parrot" >> /home/user/.ssh/authorized_keys
```

### Using an acquired private key
Once you find a key, you need to set it to read and write for only the user with chmod,  
then you can use it with SSH to log directly into the target.

```console
user@parrot~$ sudo chmod 600 acquired_id_rsa
user@parrot~$ ssh -i acquired_id_rsa user@10.10.10.10
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 4.15.0-109-generic x86_64)
Last login: Mon Jul 13 06:09:46 2020 from 10.10.15.2
user@example:~$ 
```

## Password Cracking
Since you can take rsa private keys and use them against the target, they're a prime asset to look for,  
but sometimes they can be password protected. To kill two birds with one stone, I'm going to also cover SSH  
private keys with the password cracking section. It's easy enough to google other needs and look at `john --list=subformats`.  
This could just as easily have been a random md5 hash in a database, a very strangely salted hash, or a shadow file entry.

```
Command Breakdown:
    ssh2john.py: Conversion script to take an ssh key and convert it to a workable file for John The Ripper
    > hash.txt: Direct standard out into the file hash.txt
    
    john: A tool to find weak passwords of your users
    --wordlist=: Set the wordlist to rockyou.txt, a large standard list of passwords
    hash.txt: The location of the hash you're targeting.
```

```console
user@parrot~$ /./usr/share/john/ssh2john.py acquired_id_rsa > hash.txt
user@parrot~$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 5 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
!august1993$     (acquired_id_rsa)
Session aborted
```
Note: On HTB keys, this should takes seconds to minutes. Running this on a real RSA key would take days in a VM.
