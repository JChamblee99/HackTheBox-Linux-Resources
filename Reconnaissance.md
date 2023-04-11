# Reconnaissance

 - [Port Scanning](Reconnaissance.md#port-scanning)
 - [Directory Discovery](Reconnaissance.md#directory-discovery)
 - [File Discovery](Reconnaissance.md#file-discovery)
 - [Virtual Host Fuzzing](Reconnaissance.md#virtual-host-fuzzing)
 
## Port Scanning
Nmap is a great tool for performing network recon, so obviously there are a lot of different ways you can use it.  
Personally, I tend to use variations of 3 different scans: TCP SYN scan (-sS), version scan (-sV), and throwing everything at the wall (-A).  
For more details, you can reference the [Nmap man page](https://linux.die.net/man/1/nmap) or the [Nmap SANS Cheat Sheet](https://assets.contentstack.io/v3/assets/blt36c2e63521272fdc/blte37ba962036d487b/5eb08aae26a7212f2db1c1da/NmapCheatSheetv1.1.pdf).

The TCP SYN scan tends to be pretty quick to scan every port. You won't get much information on services, but getting open ports quickly can allow you to start other scanning and enumeration sooner.

```
Command Breakdown:
   nmap: Network exploration tool and security / port scanner
   -sS: TCP SYN scan
   -p-: Scan all ports
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
```

A version scan is a great way to quickly grab service versions.

```
Command Breakdown:
   nmap: Network exploration tool and security / port scanner
   -sV: Probe open ports to determine service/version info
   -p-: Scan all ports
```

```console
user@parrot:~$ sudo nmap -sV -p- 10.10.10.10
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-09 13:30 CDT
Nmap scan report for 10.10.10.10
Host is up (0.12s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 32.54 seconds
```

Want to just get every bit of information? Throw a -A at the wall and see what sticks.  
This scan can take a while sometimes, so keep that in mind.

```
Command Breakdown:
   nmap: Network exploration tool and security / port scanner
   -A: Enable OS detection, version detection, script scanning, and traceroute
   -T<0-5>: Set timing template (higher is faster)
   -p-: Scan every port
```

```console
user@parrot:~$ nmap -A -T5 -p- 10.10.10.10
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
You can find directories and files by searching through the HTML or Javascript, but you can also fuzz for them using wordlists.  
Parrot and Kali are shipped with wordlists in `/usr/share/wordlists/`.

The most common tools used to fuzz for this information on a website are Dirb/Dirbuster and WFuzz.  
Personally, I prefer WFuzz since I have more control of what it's doing, but Dirb is great because you can start a scan and it will automatically traverse into directories to scan deeper.

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
You can specify the normal wordlist, but you can provide a second wordlist to fuzz for the file extension.

```
Command Breakdown:
   wfuzz: Web Fuzzer
   --hc 404: Ignore all 404 responses
   -w /usr/share/wordlists/wfuzz/general/big.txt: The wordlist being used
   -z list,txt-html-php: The list of file extensions to test
   10.10.10.10/FUZZ.FUZ2Z: FUZZ is replaced with the wordlist, and FUZ2Z is replaced with the file extension list
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
When you make a request to a website, the request connects to the IP address, but retains the host name within the `Host` header of the request.  
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

For Hack the Box machines, instead of having `Host: google.com`, we'd have something like `Host: box.htb`, 
where `box.htb` is the domain name of the website. Websites can serve different webpages depending on the domain name using virtual hosts. 
You can fuzz for domains and subdomains with `FUZZ.box.htb` and `FUZZ.htb`.

Among other wordlists, [SecLists](https://github.com/danielmiessler/SecLists) has a really good wordlist dedicated to this task of virtual host fuzzing.  
You'll have to `apt install` or `git clone` the repo to use the wordlists, so the directory will be different.

```
Command Breakdown:
   wfuzz: Web Fuzzer
   --hw 973: Ignore all responses with exactly 973 words (Used to exclude the default host page)
   -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt: The SecLists wordlist being used 
   -H "Host: FUZZ.htb": This is used to fuzz for the virtual host
```

```console
user@parrot:~$ wfuzz --hw 973 -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.htb"  10.10.10.10

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.10/
Total requests: 19966

===================================================================
ID           Response   Lines    Word     Chars       Payload
===================================================================

000000082:   302        0 L      0 W      0 Ch        "cms"

Total time: 0
Processed Requests: 19966
Filtered Requests: 19961
Requests/sec.: 0
```

If you find a special domain/subdomain and you want to view it yourself,  
you can add it to the `/etc/hosts` file in the format `10.10.10.10 cms.htb`.  
By default, Firefox will ignore the hosts file, so you'll have to go to  
`about:config`, change `network.dns.offline-localhost` to false,  
and then restart the browser.
