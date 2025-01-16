---
title: "Wireguard"
date: 2025-01-16T12:14:14+03:00
draft: false
toc: false
images:
tags:
  - vpn
  - wireguard
  - wg
---

# Basic setup

Install software:

```
pacman -S wireguard-tools qrencode
```

Enable forwarding:

```
cat > /etc/sysctl.d/90-local.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
[Ctr+D]
```

Apply:

```
systemctl restart systemd-sysctl
```

# Generate keys

For each peer you should generate pair of keys: public and private in this manner:

```
wg genkey | (umask 0077 && tee alice-laptop.key) | wg pubkey > alice-laptop.pub
```

then we need to setup server and client configs.

# Server configuration

{{< admonition note >}}
Keep in mind: the main point of wireguard is that it is peer to peer secured tunnel,
so you have to specify local peer (private) key in peer config and
the public key of another end of tunnel,
so for server we should specify private key of server and public keys of clients (peers),
for client - private key of client (peer) and public key of server.
{{< /admonition >}}

Create `/etc/wireguard/wg0.conf`:

```
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# alice-laptop
PublicKey = ALICE_PUB_KEY
AllowedIPs = 10.10.10.2/32

[Peer]
# bob-phone
PublicKey = BOB_PUB_KEY
AllowedIPs = 10.10.10.3/32
```

Enable and start wireguard:

```
systemctl enable wg-quick@wg0.service && systemctl start wg-quick@wg0.service
```

# Client configuration

Alice laptop config may looks like:

```
[Interface]
# alice-laptop
Address = 10.10.10.2/32
PrivateKey = ALICE_PRIVATE_KEY

[Peer]
PublicKey = SERVER_PUB_KEY
Endpoint = my.ddns.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
```

(the last line will forward all trafic over the vpn)

share with QR code:

```
qrencode -t ansiutf8 -r alice-laptop.conf
```

Import wireguard vpn to network manager:

```
nmcli connection import type wireguard file alice-laptop.conf
```
