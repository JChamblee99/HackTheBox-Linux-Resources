# Reconnaissance

 - [Port Scanning](Reconnaissance.md#port-scanning)
 - [Directory discovery](Reconnaissance.md#directory-discovery)
 - [File discovery](Reconnaissance.md#file-discovery)
 - [Virtual host fuzzing](Reconnaissance.md#virtual-host-fuzzing)
 
## Port Scanning
Starting off with a straight forward option, we can do a TCP Null scan across every port  
and then follow it up with a service scan of the specific ports that came up as open.

```
Command Breakdown:
   nmap: Network exploration tool and security / port scanner
   -sS: TCP SYN scan
   -sV: Probe open ports to determine service/version info
   -p-: Scan all ports
   -p 21,22,80: Scan only ports 21, 22, and 80
```

```console
user@parrot:~$ sudo nmap -sS -p- 10.10.10.10
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-09 13:30 CDT
Nmap scan report for 10.10.10.10
Host is up (0.12s latency).
Not shown: 65533 closed ports
PORT   STATE         SERVICE
21/tcp open          ftp
22/tcp open          ssh
80/tcp open          http

Nmap done: 1 IP address (1 host up) scanned in 8.33 seconds

user@parrot:~$ sudo nmap -sV -p 21,22,80 10.10.10.10
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-09 13:30 CDT
Nmap scan report for 10.10.10.10
Host is up (0.12s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 9.44 seconds
```

Here's an alternative option that reveals a lot more information, but can take a lot longer.

```
Command Breakdown:
   nmap: Network exploration tool and security / port scanner
   -A: Enable OS detection, version detection, script scanning, and traceroute
   -T<0-5>: Set timing template (higher is faster)
   -p-: Scan every port
```

```console
user@parrot:~$ nmap -A -T4 -p- 10.10.10.10
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-10 17:06 CDT
Nmap scan report for 10.10.10.10
Host is up (0.12s latency).
Not shown: 65490 closed ports, 42 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey: 
|   2048 8a:47:e9:21:68:12:9d:28:54:24:02:1a:39:35:4h:b9 (RSA)
|   256 32:f8:54:21:4d:46:a4:25:55:7a:87:3e:2f:a8:e7:02 (ECDSA)
|_  256 g3:2d:rk:d0:5c:42:f8:54:31:5a:f8:54:c4:a9:a7:37 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Example-Site
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 904.62 seconds
```

## Directory Discovery
There are quite a few tools that will help you find directories and files,  
and a few that are definitely more specialized for their tasks, but my favorite tool is Web Fuzzer (WFuzz).  
It's a very versatile tool that allows you to do a lot of heavy lifting.  
Another good tool for this task is DirB or DirBuster since it automatically spiders into the directores.

Your main limitation in these latter phases are often your wordlist;  
there are quite a few wordlists built into Parrot that can be found  
in `/usr/share/wordlists/`, but you can also develop a wordlist from the contents of the website if they don't work.

```
Command Breakdown:
   cewl: Custom Wordlist Generator
   -w keywords.txt: The file name to save the wordlist as
   -m 2: Lower the minimum character count
   -d 10: Increase the max directories to crawl
```

```console
user@parrot:~$ cewl -w keywords.txt -m 2 -d 10 10.10.10.10
CeWL 5.4.8 (Inclusion) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
via
SMS
chatting
Tests
CMS
...
```

Once you have your wordlist of choice, you can start fuzzing for the directories.
```
Command Breakdown:
   wfuzz: Web Fuzzer
   --hc 404: Ignore all 404 responses
   -w /usr/share/wordlists/dirb/big.txt: The wordlist being used
   10.10.10.10/FUZZ: Wfuzz replaces occurrences of the word FUZZ with the payload
```

```console
user@parrot:~$ wfuzz --hc 404 -w /usr/share/wordlists/dirb/big.txt 10.10.10.10/FUZZ

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.10/FUZZ
Total requests: 20469

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000015:   403        9 L      28 W     277 Ch      ".htaccess"
000000016:   403        9 L      28 W     277 Ch      ".htpasswd"
000002716:   301        9 L      28 W     313 Ch      "assets"
000009378:   301        9 L      28 W     313 Ch      "images"
000016215:   403        9 L      28 W     277 Ch      "server-status"

Total time: 301.9181
Processed Requests: 20469
Filtered Requests: 20463
Requests/sec.: 67.79651
```

## File Discovery
Very similar to the directory fuzzing, we can do the same thing to find files within the directories but add an extension.  
One way to do this is just putting a file extension on the end like `FUZZ.txt` but there's a better option.

```
Command Breakdown:
   wfuzz: Web Fuzzer
   --hc 404: Ignore all 404 responses
   -w /usr/share/wordlists/wfuzz/general/big.txt: The wordlist being used
   -z list,txt-html-php: The list of extensions to test
   10.10.10.10/FUZZ.FUZ2Z: FUZZ is replaced with the wordlist, and FUZ2Z is replaced with the extension list
```

```console
user@parrot:~$ wfuzz --hc 404 -w /usr/share/wordlists/wfuzz/general/big.txt -z list,txt-html-php 10.10.10.10/FUZZ.FUZ2Z

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.10/FUZZ.FUZ2Z
Total requests: 9072

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000004122:   200        0 L      5 W      30 Ch       "login - php"
000008263:   200        4 L      23 W     118 Ch      "creds - txt"

Total time: 269.6662
Processed Requests: 9072
Filtered Requests: 9070
Requests/sec.: 33.64158
```

## Virtual Host Fuzzing
When you make a request to a website, the request connects to the IP address,  
but retains the host name within the `Host` header of the request.  
This allows the website to serve different domain names and subdomains.
```console
user@parrot:~$ curl -v google.com 
*   Trying 172.217.13.174:80...
* TCP_NODELAY set
* Connected to google.com (172.217.13.174) port 80 (#0)
> GET / HTTP/1.1
> Host: google.com
> User-Agent: curl/7.68.0
> Accept: */*
```

```
Command Breakdown:
   wfuzz: Web Fuzzer
   --hw 973: Ignore all responses with exactly 973 words (Used to exclude the default host page)
   -w keywords.txt: The wordlist being used
   -H "Host: FUZZ.htb": The hostname format that's fuzzed
```

```console
user@parrot:~$ wfuzz --hh 8193 -w keywords.txt -H "Host: FUZZ.htb"  10.10.10.10

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.10/
Total requests: 609

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000005:   302        0 L      0 W      0 Ch        "CMS"

Total time: 9.234456
Processed Requests: 609
Filtered Requests: 608
Requests/sec.: 65.94865

```

If you find a special domain/subdomain and you want to view it yourself,  
you can add it to the `/etc/hosts` file in the format `10.10.10.10 cms.htb`.  
By default, Firefox will ignore the hosts file, so you'll have to go to  
`about:config`, change `network.dns.offline-localhost` to false,  
and then restart the browser.
