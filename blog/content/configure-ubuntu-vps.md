+++
title = "Ubuntu 24.04 LTS cloud server quick configuration."
date = 2024-09-28
draft = false
[taxonomies]
  tags = ["Linux"]
[extra]
  toc = true
	keywords = "Linux, VPS"
+++

This is a personal note post to remind me the steps for basic cloud server configuration on Ubuntu 24.04 LTS.

## Update all packages & Linux kernel

The first thing to do is always updating everything, including the Linux kernel.

```bash
sudo apt update
sudo apt upgrade
sudo reboot
```

## New User Configuration

If there is only a `root` user, you need to add a non-root sudo user for security reasons:

```bash
adduser <newuser> sudo
su <newuser>
```

## Network Configuration

First make sure `networkd` is running.

```bash
sudo systemctl restart systemd-networkd.service
```

Then go to `/etc/netplan` and check if there is existing configuration yaml file, you need to create one if not.


```bash
sudo vim default.yaml
```

The yaml content:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
      addresses:
        - <server_ipv4>/24
        - <server_ipv6>/64
      routes:
        - to: default
          via: <setver_gateway_ipv4>
        - to: default
          via: <setver_gateway_ipv6>
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
          - 2606:4700:4700::1111
          - 2606:4700:4700::1001
```

You don't need `addresses` and `routes` if your provider supports `dhcp4: yes` or `dhcp6: yes`.

### Enable `DNSSEC` and `DNSOverTLS`

```bash
sudo vim /etc/systemd/resolved.conf
```

Enable:

```conf
DNSSEC=yes
DNSOverTLS=yes
```

And restart 

```bash
sudo systemctl restart systemd-resolved.service
```

## SSH Configuration

### Add SSH public key

Copy and paste the public key to server:

```bash
mkdir ~/.ssh/
vim ~/.ssh/authorized_keys # Paste public key
sudo chmod 400 ~/.ssh/authorized_keys
```

### Configure `sshd_config`

```bash
sudo vim /etc/ssh/sshd_config
```

* Listen on un-conventional SSH port (instead of 22).
* Disable root login.
* Disable passwod authentication.

```conf
Port <newport> # Setting is not used after 22.10
PermitRootLogin no
PasswordAuthentication no
```

#### Configure `ssh.socket`

Because Ubuntu 22.10 and later uses [socket-based activation](https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189).

You need to edit the `ssh.socket` trigger to change the `ListenPort` and `ListenAddress` settings:

```bash
sudo vim /usr/lib/systemd/system/ssh.socket
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

### Update SSH client configuration

After you have changed the server settings, make sure to update the settings on the client side:

```bash
vim ~/.ssh/config
```

An example ssh client config file:

```conf
Host <my_server_name>
    HostName <ip1>, <ip2>, <ip3>, ...
    User <user_name>
    Port <ssh_port>
    IdentityFile ~/.ssh/id_<keyfile>
```

### Restart and test new SSH settings

You want to restart and test the new SSH settings before moving on.

```bash
sudo systemctl restart ssh
```

On the client side:

```bash
ssh <my_server_name>
```

Make sure client can login without any problems. Go back if it doesn't work.

## Update `nftables`

First remove `iptables` and `ufw`, we are going to use `nftables` only.

```bash
sudo apt install -y nftables
sudo apt autoremove -y iptables ufw
```

### Add basic nft rules

Edit `/etc/nftables.conf`.

```bash
sudo vim /etc/nftables.conf
```

```conf
#!/usr/sbin/nft -f

flush ruleset
table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        # SSH port.
        tcp dport <ssh_port> accept
	# DHCPv6 port & icmpv6
	udp dport 546 accept
	ip protocol icmp accept
	ip6 nexthdr icmpv6 accept
        # Allow established and related packets.
        ct state vmap { established : accept, related : accept, invalid : drop }
        # Allow loopback.
        iifname lo accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
        # Default policy drop, it is not a router.
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

Make sure the `<ssh_port>` is correct or you will lose the active SSH session immediately after `nftables` restarts.

```bash
sudo systemctl restart nftables.service
```

If it works and you didn't lose the connection, enable it in systemd. Otherwise, reboot to undo the changes.

```bash
sudo systemctl enable nftables.service
```

## (Optional) Install and configure `fail2ban`

This step is optional, but adds more security to our server.

```bash
sudo apt update && sudo apt install -y fail2ban
```

The configuration is simple:

```bash
sudo vim /etc/fail2ban/jail.d/defaults-debian.conf
```

Make sure you have installed `nftables` and uninstalled `iptables`, `ufw`, etc. before installing the `fail2ban` package. It will automatically use `nftables` as ban actions. 

```bash
sudo systemctl restart fail2ban.service
```
