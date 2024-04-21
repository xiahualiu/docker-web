+++
title = "Make Jenkins Accessible Only through WireGuard VPN"
[taxonomies]
  tags = ["docker"]
[extra]
  toc = true
+++

In the previous post [Set up Jenkins and Nginx Reverse Proxy in Docker Containers](/blog/nginx-jenkins-reverse-proxy), we successfully deployed the Jenkins controller node and reverse proxied the website page with Nginx.

However, it is not ideal to directly expose your Jenkins website on the public internet, since it could be the target of someone who wants to infiltrate your system.

In the real world, the Jenkins website is usually protected by some VPN, which makes it only accessible to those "authorized users".

In this post I will show you how to use one of the most popular VPN right now on Linux, [WireGuard VPN](https://www.wireguard.com/) to protect the Jenkins web page on your instance.

## Requirements

* You have Jenkins.
* You have Nginx.
* You have configured Nginx to reverse proxy Jenkins web page.

## Install WireGuard VPN

Install WireGuard VPN is fairly easy in most modern Linux distributions, you can check the [official install document](https://www.wireguard.com/install/) for more information.

Although there is a container version of it, I **DON'T** recommend using it, because WireGuard VPN relies on the Linux kernel module to function safely, and in docker containers WireGuard cannot access the bare metal OS.

## Writing WireGuard Server & Client Configuration files

The example configuration files can be found in my repository: 

* Example Server Configuration: [wg0.conf](https://github.com/xiahualiu/docker-nginx-jenkins-zola/blob/main/wireguard/wg0.conf)
* Example Client Configuration: [wg0_client.conf](https://github.com/xiahualiu/docker-nginx-jenkins-zola/blob/main/wireguard/wg0_client.conf)

Note that in the example configuration files, the WireGuard subnet (CIDR) is `10.66.66.0/24`, and server use `10.66.66.1`, the client use `10.66.66.2`. These values are not specific, you can change them if you want, however this post will use these values later on.

Also the server use the default WireGuard VPN port `51820`.

### Generate Keys

Use this one line command to generate a single key pair:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

There will be 2 new files named `privatekey` and `publickey` created, and you can use `cat` to see the key content.

You will need to generate 2 key pairs, one for your server and one for your client.

After generating the key pairs, you need to fill out the `<Server-Private-Key>`, etc fields in the `wg0.conf` and `wg0-client.conf`.

### WireGuard Client

For client connection, you need to copy and paste the `wg0-client.conf` content to your WireGuard client software.

Note that in the client configuration file, the `AllowedIPs` means which network traffic should be sent to the VPN tunnel, and it **should be the `wg0` IP address of the server**, which in our case, it is `10.66.66.1`.

Why it should be `10.66.66.1` instead of the public IP address? You will understand why at the next step:

## Change DNS record

Now we need to move on changing the DNS record for our Jenkins site, instead of mapping `jenkins.domain.name` to the server's public IP address, we map it to the internal `wg0` interface IP address. 

After that this will happen:

* When a user **without** the WireGuard VPN tries to visit the site `jenkins.domain.name`. Because this domain name get resolved to a LAN address `wg0`, he will not be able to access the website.
* When a user **with** the WireGuard VPN visits the site, because WireGuard client software on his machine redirects the network packet (set by `AllowedIPs` field in the client configuration) to the server `wg0` interface. The request will reach to Nginx and eventually Jenkins. 

This allows us to block the non-authorized users out in our system to some content, but the user still can access the site if he overrides the host setting of `jenkins.domain.name` on his machine.

Next we are going to enable the IP filter function on Nginx, so that Nginx only allows requests from WireGuard VPN IP addresses.

## Enable IP filter Nginx

Go to your Nginx configuration file and add these 2 lines:

```nginx
location / {
  # ...
  allow 10.66.66.0/24; # Only allow Wireguard Peers to visit
	deny all;            # Deny all other IPs
  # ...
}
```

And restart Nginx container `docker compose restart webserver`, to apply the new configurations.

Now the Nginx will only server Jenkins to whose IP address belongs to the `10.66.66.0\24` CIDR and will return `403` to those who doesn't. And users on the WireGuard VPN has a source IP address that belongs to `10.66.66.0\24`.

This IP filtering mechanism eliminates the chance that someone can visit your Jenkins site with modified host file.

## Postscripts

Note that there is still a very slim chance that the attacker can pretend he is on the WireGuard VPN subnet even after all those settings, to eliminate this possiblity you can add a firewall rule to the `eth0` or similar public network interface, and blocking all IP addresses from `eth0`.

Because WireGuard VPN use the tunneled connection over the `51820` port, this rule will not interrupt the WireGuard connection.