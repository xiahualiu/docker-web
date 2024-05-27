+++
title = "Make Jenkins Accessible Only through WireGuard VPN"
date = 2024-04-19
draft = false
[taxonomies]
  tags = ["Docker"]
[extra]
  toc = true
	keywords = "Jenkins, WireGuard, VPN"
+++

In the previous post [Set up Jenkins and Nginx Reverse Proxy in Docker Containers](/blog/nginx-jenkins-reverse-proxy), we successfully deployed the Jenkins controller node and reverse proxied the website page with Nginx.

However, it is not ideal to directly expose your Jenkins website on the public internet, since it could be the target of someone who wants to infiltrate your system.

Instead, the Jenkins website is usually protected by some VPN, which makes it only accessible to those "authorized users".

In this post I will show you how to use the popular VPN, [WireGuard VPN](https://www.wireguard.com/) to protect your Jenkins website.

## Requirements

* You have Jenkins.
* You have Nginx.
* You have configured Nginx to reverse proxy Jenkins web page.

## Install WireGuard VPN

Install WireGuard VPN is fairly easy on most modern Linux distributions, you can check the detailed [official install document](https://www.wireguard.com/install/) for more information.

Although there is a container version of WireGuard, I **DON'T** recommend using it, because WireGuard VPN relies on the Linux kernel module to function safely, and in docker containers WireGuard cannot access the bare metal OS.

## Writing WireGuard Server & Client Configuration files

The example configuration files can be found in my repository: 

* Example Server Configuration: [wg0.conf](https://github.com/xiahualiu/docker-nginx-jenkins-zola/blob/main/wireguard/wg0.conf)
* Example Client Configuration: [wg0_client.conf](https://github.com/xiahualiu/docker-nginx-jenkins-zola/blob/main/wireguard/wg0_client.conf)

Note that in the example configuration files, the WireGuard subnet (CIDR) is `10.66.66.0/24`, and server used `10.66.66.1`, the client used `10.66.66.2`. These values are not specific, you can change them to any value you want, however this post will use these values later on for demonstration purposes.

Also the server configuration used the default WireGuard VPN port `51820`.

### Generate Keys

Use this one line command to generate a single key pair:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

This command creates 2 new files named `privatekey` and `publickey`, you can use `cat` to print the key content.

You will need to generate 2 key pairs, one for your server and one for your client. This means you will have 2 public keys and 2 private keys in total.

After generating the 2 key pairs, you can then fill out the `<Server-Private-Key>`, etc fields in the `wg0.conf` and `wg0-client.conf`.

### WireGuard Client

For client connection, you need to copy and paste the `wg0-client.conf` content to your WireGuard client software.

Note that in the client configuration file, the `AllowedIPs`, meaning which network traffic should be sent to the VPN tunnel, **should be the `wg0` IP address of the server**. In our case, it is `10.66.66.1`.

Why it is `10.66.66.1` instead of the public IP address? You will know it later, let's move onto the next step.

## Change the DNS record

Now we need to move on changing the DNS record for our Jenkins site, instead of pointing `jenkins.domain.name` to the server's public IP address, we modify it, so that it points to the internal `wg0` interface IP address. 

After that this will happen:

* When a user **without** the WireGuard VPN tries to visit the site `jenkins.domain.name`. Because this domain name is resolved to a LAN address `wg0`, he will not be able to access the website.
* When a user **with** the WireGuard VPN visits the site, the WireGuard VPN software on his machine tunnels the network packet (set by `AllowedIPs` field in the client configuration) to the server `wg0` interface. The request will then reach Nginx and eventually Jenkins. 

This allows us to block the non-authorized users out in our system to some content, but the user still can access the site if he overrides the host setting of `jenkins.domain.name` on his machine.

To solve the above issue, next we are going to enable the IP filter function on Nginx, so that Nginx only allows requests from WireGuard VPN IP addresses.

## Enable the IP filter on Nginx

Go to your Nginx configuration file and add these 2 lines:

```
location / {
  # ...
  allow 10.66.66.0/24; # Only allow Wireguard Peers to visit
	deny all;            # Deny all other IPs
  # ...
}
```

Then restart Nginx container `docker compose restart webserver`, to apply the new configurations.

Now the Nginx only serves Jenkins to whose IP address belongs to the `10.66.66.0\24` CIDR and returns `403` to those who doesn't. And since users on the WireGuard VPN has a source IP address that belongs to `10.66.66.0\24`, their access won't be interrupted.

This IP filtering mechanism eliminates the chance that someone can visit your Jenkins site with modified host file.

## Postscripts

Note that there is still a very slim chance that the attacker can pretend he is on the WireGuard VPN subnet even after all those settings, to eliminate this possiblity you can add a firewall rule to the `eth0` or similar public network interface, and blocking all IP addresses from `eth0`.

Because WireGuard VPN uses the tunneled connection over the `51820` port, this rule will not disrupt the WireGuard VPN connection.
