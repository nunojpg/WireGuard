[![Actions Status](https://github.com/nunojpg/WireGuard-BASICS/workflows/main/badge.svg)](https://github.com/nunojpg/WireGuard-BASICS/actions)

# WireGuard BASICS &mdash; WireGuard spiked for easy control of a large fleet of devices
WireGuard was merged into the Linux kernel for 5.6. This repository contains a backport of WireGuard for kernels 3.10 to 5.5, as an out of tree module.

WireGuard BASICS is a small patch on top of mainline [WireGuard](https://www.wireguard.com/) that simplifies a arguably typical scenario that WireGuard mainline does not support and likely has not intention to support.

With WireGuard BASICS IP address management becomes automatic.

WireGuard BASICS runs on a Server, where a small or large number of clients connect. Clients use mainline WireGuard (IMPORTANT: clients should have regular WireGuard installed, not WireGuard BASICS).


## Clients

All clients use the same configuration to connect to the Server, except for the individual private keys:


#### WireGuard configuration
```
[Interface]
PrivateKey = <client unique private key>

[Peer]
Endpoint = tunnel.example.com:9876
PublicKey = <server public key>
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```
#### IP configuration
```
ip address add dev wg0 10.73.0.1 peer 10.73.0.0
```


## Server

On the server the (Basic) Magic happens:


#### WireGuard BASICS configuration

```
[Interface]
ListenPort = 9876
PrivateKey = <server private key>

[Peer]
PublicKey = <client 1 public key>
AllowedIPs = 10.73.0.1/32

[Peer]
PublicKey = <client 2 public key>
AllowedIPs = 10.73.0.2/32
```

#### IP configuration
```
ip address add dev wg0 10.73.0.0/16
```

## And now?

Now from the Server you can connect to client \<n> with ssh 10.73.0.\<n>.

Under the hood, when you connect to 10.73.0.\<n>, the server rewrites the destination address to 10.73.0.1 as the client expects.

When the client sends a packet the reverse happens.

The second additional feature of WireGuard BASICS is that if a unauthorized peer attempts to connect its Public Key and IP address is printed to the kernel message buffer (dmesg/journalctl -k). We use this to remotely accept new clients after confirming the public key out of band (as they are easy to confirm over the phone, but not so easy to pass verbatim):

```
Oct 25 10:47:10 instance-2 kernel: wireguard: wg0: denied peer 48.3.1.7:46248 YcMIghE09gVaZJLdgmgQlNRuISYjBZ/9adYgdQe460g=
```


## What else?

Clients might be under lighter security enforcement, so you might want to disable inbound connections from the clients so that a client cannot connect to services on the server or other clients. Consider this iptables rules:

```
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i wg0 -j REJECT --reject-with icmp-port-unreachable
```

On the clients you can also automate the configuration of the Server public key. A neat solution would be using a DNSSEC record, which at the moment is probably not reliable for about 100% of the world, so I personally use TLS:

```
wget https://example.com/basics-pub-key -O /etc/wgbasics/pubkey
pubkey="$(cat /etc/wgbasics/pubkey)"
wg set wg0 peer "${pubkey}"
```

## Limitations

The Server can only reach every Client immediate endpoint. It's not possible to address other destinations on the Client network.

The subnet 10.73.0.0/16 is hardcoded. It's trivial to change it. 10.73.0.0 belongs to the Server, so without any modification you can connect to 65534 clients.

## Can I run WireGuard BASICS and WireGuard simultaneously on the same Server?

Omg, don't try it. WireGuard BASICS could be extended to allow for BASICS and regular peers according to the configuration. The patch would explode in size. I have no acute interest on that.

## Future

I would like to move the unauthorized peer connection attempts log from kernel message buffer into /sys, with something like fixed size array of the last 100 attempts per key, with the corresponding IP address and timestamp. I would though prefer if someone would do it instead of me.

## License

This project is released under the [GPLv2](COPYING).
