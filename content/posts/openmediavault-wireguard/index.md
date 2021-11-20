---
title: 'Configure Wireguard on Openmediavault 5'
description: 'Just a quick tutorial on how install and configure Wireguard on Debian (OMV5).'
cover:
    image: 'mathew-schwartz-sb7RUrRMaC4-unsplash.jpg'
    alt: 'Wireguard it!'
ShowToc: true
date: 2021-11-20T07:33:51-03:00
draft: false
---

## Introduction
As per my previous [post](https://jessequinn.info/posts/openmediavault-openvpn/) on OMV5 and OpenVPN, I decided to install [Wireguard](https://www.wireguard.com/), a more performant VPN software.

## Configuration
To start, we need to install Wireguard on OMV5. So SSH into your box and run:

```bash
apt update
apt install wireguard
```

Verify that the Wireguard kernel module is functional:

```bash
/sbin/modinfo wireguard
```

Create a server keypair:

```bash
mkdir /etc/wireguard/
chmod 700 /etc/wireguard/
cd /etc/wireguard/
wg genkey | tee vpn-server-private.key | wg pubkey > vpn-server-public.key
```

Install Wireguard on your client machine. With OSX, the keypair is generated for me, and I use the following configuration:

```
[Interface]
PrivateKey = xxx
Address = 10.0.2.2/24
DNS = 8.8.8.8, 8.8.4.4

[Peer]
PublicKey = xxx
AllowedIPs = 0.0.0.0/0
Endpoint = xxx.xxx.xxx.xxx:55820 # use the IP of your Wireguard server
```

On linux you need to generate the key pair:

```bash
mkdir /etc/wireguard/
chmod 700 /etc/wireguard/
cd /etc/wireguard/
wg genkey | tee vpn-client-private.key | wg pubkey > vpn-client-public.key
```

Then back to your Wireguard server. Output the private key of the server and public key of the client and copy into the command below:

```bash
cat vpn-server-private.key
cat > /etc/wireguard/wg0.conf << EOF
[Interface]
Address = 10.0.2.1/24
ListenPort = 55820
PrivateKey = # server private key from above
MTU = 1420

[Peer]
PublicKey = # public client key
AllowedIPs = 10.0.2.2/32
EOF
```

You will need to incorporate some firewall rules and enable ip forwarding:

```bash
# and modify /etc/sysctl.conf as well
sysctl --write net.ipv4.ip_forward=1
# eno1 represents MY internet facing interface
iptables --table nat --append POSTROUTING --jump MASQUERADE --out-interface eno1
# 192.168.122.0/24 is my virtual machines' bridged network
iptables --table filter --insert FORWARD -s 10.0.2.0/24 -d 192.168.122.0/24 -j ACCEPT
```

Start the service:

```bash
systemctl enable wg-quick@wg0.service
systemctl daemon-reload
systemctl start wg-quick@wg0.service
```

## Final Words
Wireguard is super simple to setup. However, you will not have an interface in the OMV5 GUI, but Wireguard is worth it.
