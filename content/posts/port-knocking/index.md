---
title: 'Port knocking'
description: 'A very short article on port knocking.'
date: "2021-09-11"
tags: ["Linux", "Firewall"]
ShowToc: true
cover:
  image: 'anthony-rampersad-gXVCjlTyNtE-unsplash.jpg'
  alt: 'Port knocking'
---

## Introduction
This very short article is about port knocking and iptables. Port knocking allows a specific port to be opened when a sequence of connection attempts on predefined ports
are made. The correct sequence of "knocks" will dynamically open the desired port temporarily so that a connection can be made on the desired port.

In this article we will use port 22 as the port to hide with four (4) UDP ports to knock. 

## Code
Using iptables, the following commands will create several chains, `INTO-PHASE2`, `INTO-PHASE3`, and `INTO-PHASE4`. 
Each chain will have several rules appended. Specifically 
a match is used to make up the condition under which the next step is invoked. 
If the given sequence of UDP ports are knocked in sequence port 22 on eth0 interface will open for five (5) seconds. 

```bash
## Define port knocking
sudo iptables -N INTO-PHASE2
sudo iptables -A INTO-PHASE2 -m recent --name PHASE1 --remove
sudo iptables -A INTO-PHASE2 -m recent --name PHASE2 --set
sudo iptables -A INTO-PHASE2 -j LOG --log-prefix "INTO PHASE2: "
sudo iptables -A INTO-PHASE2 -j DROP

sudo iptables -N INTO-PHASE3
sudo iptables -A INTO-PHASE3 -m recent --name PHASE2 --remove
sudo iptables -A INTO-PHASE3 -m recent --name PHASE3 --set
sudo iptables -A INTO-PHASE3 -j LOG --log-prefix "INTO PHASE3: "
sudo iptables -A INTO-PHASE3 -j DROP

sudo iptables -N INTO-PHASE4
sudo iptables -A INTO-PHASE4 -m recent --name PHASE3 --remove
sudo iptables -A INTO-PHASE4 -m recent --name PHASE4 --set
sudo iptables -A INTO-PHASE4 -j LOG --log-prefix "INTO PHASE4: "
sudo iptables -A INTO-PHASE4 -m recent --rcheck --name PHASE4 -j LOG --log-prefix "(OPEN PORT 22) - "
sudo iptables -A INTO-PHASE4 -j DROP

## if you want to knock using tcp packets uncomment this line:
#sudo iptables -A INPUT -m recent --update --name PHASE1

## Define knocking sequence: knock on ports and then open SSH port for 5 seconds
sudo iptables -A INPUT -p udp --dport 23456 -m recent --set --name PHASE1
sudo iptables -A INPUT -p udp --dport 34567 -m recent --rcheck --name PHASE1 -j INTO-PHASE2
sudo iptables -A INPUT -p udp --dport 45678 -m recent --rcheck --name PHASE2 -j INTO-PHASE3
sudo iptables -A INPUT -p udp --dport 56789 -m recent --rcheck --name PHASE3 -j INTO-PHASE4

sudo iptables -A INPUT -p tcp --dport 22 -i eth0 -m recent --rcheck --seconds 5 --name PHASE4 -j ACCEPT
```

The following script can be used to open port 22 where `HOST` is the ip address of the sshd:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
HOST='xxx.xxx.xxx.xxx'

PORT1=23456
PORT2=34567
PORT3=45678
PORT4=56789

echo "KNOCK1"
echo -n "*" | nc -w1 -u $HOST $PORT1

echo "KNOCK2"
echo -n "*" | nc -w1 -u $HOST $PORT2

echo "KNOCK3"
echo -n "*" | nc -w1 -u $HOST $PORT3

echo "KNOCK4"
echo -n "*" | nc -w1 -u $HOST $PORT4

echo "CONNECT"
```

## Final words
Port knocking is a great way to protect your ports. Specifically for sshd, which is usually kept open. An additional enhancement for security purposes would be to change the port
that sshd uses.