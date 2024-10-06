+++
title = "Understand routing and NAT with WireGuard VPN."
date = 2024-10-05
draft = false
[taxonomies]
  tags = ["Linux"]
[extra]
  toc = true
	keywords = "Network, nftables, NAT"
+++

Hi I am the maintainer of the GitHub project [wg_gaming_installer](https://github.com/xiahualiu/wg_gaming_installer). This post intends to provide explanations of how these `nftables` rules worked. In the meanwhile, it also serves a very good reading material for anyone who wants to know more about NAT(Network Address Translation) and IP routing.

## IPv4 & IPv6 Packet Structure

Before we move on, please refer to:

* [IPv4 header format](https://en.wikipedia.org/wiki/IPv4)
* [IPv6 header format](https://en.wikipedia.org/wiki/IPv6)

For basic knowledge of IPv4/IPv6 packet formats.

Highlight these fields:

* **Protocol** (For IPv6), **Next Header** (For IPv6).
* **Source Address**.
* **Destination Address**.

These fields provide important information for IP routing, and the NAT operations.

## `nftables` Workflow

![](images/nf-hooks.png)

For simplicity we are going to assume that we don't have any bridge device inside our system.

This post only describes packet processing steps on the WireGuard server, where all the magic happens.

We will use **inet address** from now on to represent an IPv4 or IPv6 address. This notion is consistent with `nftables`.

### Packets received from a WireGuard peer

For packets sent by a WireGuard client, it will first be processed by the WireGuard tunnel.

#### Exit WireGuard tunnel

WireGuard driver decrypts the UDP packet, matches the packet with its peer records, then accepts this packet on the WireGuard interface as an ingress packet. Check [WireGuard document](https://www.wireguard.com/#simple-network-interface) for more information.

For this ingress packet, the **source address** will be the peer's WireGuard interface inet address upon exit the tunnel (The `AllowIPs` on the server side only allows packets on the `AllowIPs` list for a peer).

If you are using my installer script, the **source address** may be `10.66.66.2/24` or `fd42:42:42::2/64`.

Refering to the `nftables` workflow diagram which is shown above, the **Driver RX path** is WireGuard interface device in this scenario.

#### Prerouting Hook

There is no NAT rules matched for this packet, since our DNAT rules specify that the `iifname` to be the sever's public network interface name. This means the **destination address** is not changed when the packet is sent from WireGuard interface.

#### Routing Decision

* If the **destination address** of this packet matches the server's WireGuard inet address (`10.66.66.1`), this packet will be sent to the **Input Hook** of `nftables`. 

This case is shown as `Local? Yes` branch on the `nftables` workflow diagram.

* If the **destination address** does not match WireGuard's inet address, then it will proceed to the route table next (can be printed with `ip route`), it means the packet will be forwarded to another interface.

This case is shown as `Local? No` branch on the `nftables` workflow diagram.

#### Input Hook

If the packet goes to `Local? Yes` branch, it triggers the **Input Hook** next.

The **Input Hook** is probably most well-known by people as firewall rules. You can set up a list of rules to accept/drop packets here for the applications running on the server.

You can find examples on the `nftables` website [here](https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_server) for the **Input Hook** chains.

>Note: For applications listening on these interfaces, the received packet's **source address** is the sender WireGuard peer's address (such as `10.66.66.2`), they can simply reply to this address (the WireGuard peer) next.

#### Forward Hook

If the packet goes to `Local? No` branch, it triggers the **Forward Hook** next.

For the installer script, we allow a packet to be forwarded between the server's public network interface and WireGuard interface. And this is important for the DNAT rules we are going to talk about for packets received on the public network interface in a minute.

If the WireGuard peer hopes to access an inet address that is on the public Internet, then it will be routed by the route table. For most cases, it falls under the `default` entry, which points to the public network gateway.

This means the packet will be sent to the gateway that linked with the public network interface. And the gateway then will send the packet to other devices on the internet.

You can also add additional forward rules here if you want the WireGuard peer to be able to access another network interface on the server, such as a docker container's network interface, or even the `lo` interface.

#### Postrouting Hook

After the routing decision is made and the packet goes through the **Forward Hook** then it comes to the **Postrouting Hook**.

In the installer script, there is only one SNAT rule, and it is called **Masquerade SNAT**.

```conf
oifname "$SERVER_NIC_INTERFACE" masquerade comment "WireGuardGamingInstaller"
```

What it does is very simple, it changes the **source address** of a packet to what the `oifname` inet address is (this address is obtained from the route table). This is important because, without this step, when the packet reaches its destination on the Internet, the receiving server cannot reply to the packet because it doesn't know where this packet comes from! (It only sees the packet is sent by the WireGuard peer's address (for example, `10.66.66.2`), but it cannot find the sender device after sending the ARP request, because this WireGuard peer's inet address is not reachable from the destination server.)

Also the **Masquerade SNAT** rule keeps track of the established connection so that the reply packet at the same TCP/UDP port with source address equal to this sent packet's destination address can be DNAT to the correct WireGuard peer.

#### Egress

The packet now has new **source address** (the server's public inet address), unchanged **destination address**, and is ready to be sent to the gateway device next!

The **Driver TX path** is the server's public network interface device in this scenario.

### Packets received from the server's public network interface

The **Driver RX path** is the server's public network interface in this scenario.

#### Prerouting Hook

In the installer script, we set some DNAT rules to match the packets coming from the server's public network interface.

Specifically, if a packet contains TCP or UDP connection to certain ports listed in the rules, its **destination address** will be changed.

And after the change, the packet now contains **destination address** to a certain WireGuard peer.

This is where the port forward magic happens. From the outside, the packet sender only sees it is sending packets to the server. Inside the server, the packet changed its destination, and will be routed to the WireGuard gateway next.

Also DNAT rules keep track of the established connections, so that the reply packets will be automatically SNAT to the server's public network interface before leaving the server, which is kind of similar to the **Masquerade** rules we saw above.

#### Routing Decision

If the received packet matched the DNAT rules, its **destination address** will be changed, and it will be routed to the WireGuard gateway next.

>Note: For those packets that do not match the DNAT rules, their next stop will still be **Input Hook** as usual, because the routing decision will go to `Local? Yes` branch. We will ignore these packets and focus on the DNAT-ed packets.

#### Forward Hook

We accept packets routed from the server's public network interface to WireGuard's network interface.

#### Postrouting Hook

The **source address** will not be changes, since we only masquerade packets output from the server's public network interface.

#### Egress to WireGuard

For WireGuard, it reads the **destination address** and found the destination peer, then encrypt the data with this peer's public key, then send the encrypted UDP packet thru the WireGuard tunnel.

The corresponding WireGuard peer receives the packet, and read the original **source address** as it was received on the server's public network interface, so it can send the reply packet with this **source address**. 

## Review

This post goes over how packets sent from:

* WireGuard peers to server.
* WireGuard server to peers.

are processed by the server's `nftables` rules, and routed by the system.

The most intriguing part of this process is how these DNAT and SNAT rules work together to make the packets forwarded from the server to client seamlessly.

* For the client side, it sees the packets are directly sent by the original sender. Because client's peer `AllowIPs` contains every inet address, it accepts all packets with any **source address** from the server. It also means all outgoing packets will be routed by WireGuard interface on the client side. 
* For the server side, it masquerades the packets from client peer inet address (`10.66.66.2`) to the public network interface address, then send them to the receiver host over the default gateway. When someone over the Internet request a new connection on one of the port, the `nftables` will DNAT the packet to the WireGuard client, so that the connection can be established.

The DNAT rules we added makes someone on the Internet can create a new connection to one of the WireGuard peers. Without the DNAT rules, only the WireGuard peers can establish a new connection to someone over the Internet with the help of the **Masquerade SNAT** rule.

## Reserve DNAT ports

When the client establishes a new TCP/UDP connection, it will open a random port from the **Ephemeral Port Range**, you can check the range by:

```bash
$ sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768    60999
```

If you set a port number within this range for your DNAT rules, strange thing may happen.

For example, WireGuard `peer1` established a TCP connection with source port `60000` on the server. And you have a DNAT rule on `60000` to `peer2`.

Because in the installer script, the prerouting priority is `dstnat`=-100, which is lower than conntrack's priority `NF_IP_PRI_CONNTRACK`=-200, your DNAT rule will not work in this situation because the conntrack's rule triggers first.

In order to solve this problem, we need to add our DNAT rules to the [system's reserved ports](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html#ip-variables).

```bash
sysctl -w net.ipv4.ip_local_reserved_ports = <forward_port_range>
```
