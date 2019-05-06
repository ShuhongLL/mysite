---
title: How to Check Open TCP/IP Ports in Mac OS X
date: 2019-05-06 17:48:23
tags: [CLI, Mac OS, port]
photos: ["../images/cli.JPG"]
---
The core of Mac OS is Darwin and we can use most of the CLI tools in Mac OS just like how it feels like in Linux. If we want to check out the current ports in usage, the command **`netstat`** is useful:
>netstat -ap tcp | grep -i "listen"

That will print out something like this in the console:
```
Achive Internet connections (including servers)
Proto       Recv-Q      Send-Q      Local Address       Foreign Address     (state)   
tcp4         0                0                ocalhost.25035      *.*                          LISTEN
```
That works but the problem is that it doesn't show up the names of the procedures which occupy the ports. Sometimes we want to know precisely which program is exposing the port. 

Then found out that there is another command **`lsof`**:
>sudo lsof -nP -iTCP:PortNumber -sTCP:LISTEN

```
COMMAND     PID     USER   FD   TYPE    DEVICE      SIZE/OFF    NODE          NAME
syslogd           350      root     5w     VREG  222,5          0                 440818        /var/adm/messages
syslogd           350      root     6w     VREG  222,5          339098       6248            /var/log/syslog
cron                353      root     cwd   VDIR    222,5          512             254550        /var -- atjobs
```

**`-n`** : No dns (no host name)
**`-P`** : List port number instead of its name
**`-i `** : Lists IP sockets

To view the port associated with a daemon:
>lsof -i -n -P | grep python

</br>

If we just want to see the name:
>sudo lsof -i :PortNumber | grep LISTEN

</br>

Get all running **PID** in a specific port:
>sudo lsof -i :PortNumber| grep LISTEN | awk '{ print $2; }' | head -n 2 | grep -v PID

</br>

And then we can kill all the processes:
>sudo kill -9 $(sudo lsof -i :PortNumber| grep LISTEN | awk '{ print $2; }' | head -n 2 | grep -v PID)

</br>

list all commands:
>lsof -h

