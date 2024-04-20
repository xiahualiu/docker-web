# Nginx Jenkins Zola Docker Compose

My docker compose project for my Oracle Ampere A1 instance.

It contains several docker containers:

* [Nginx](https://nginx.org/) container, for serving website content such as Jenkins and blog.
* [Certbot](https://certbot.eff.org/) container, for obtaining and renewing website SSL certificates.
* [Jenkins](https://www.jenkins.io/) container, CI of other projects.
* [Zola](https://www.getzola.org/) container, my blog site generator.

## Why Docker containers

The most straight forward answer is for security reasons, because containers are isolated from each other and the OS, even if one of them is compromised by external attacks, the bad actor cannot obtain the whole access to the system data.

And with the docker compose file, it is easier to redeploy the same services on multiple machines.

## Requirements

* A **domain** name. (mine is fredrice.us)

Only with a domain name you can get the SSL certificate and harden your website.

* **Docker Compose** version > 3.0.

Docker compose needs to be highier than 3.0 version.

* **WireGuard VPN**

The Jenkins server is hosted behind the WireGuard VPN for security reason. In order be able to access the Jenkins control website, the user must first use WireGuard VPN to connect to the internal `wg0` interface on the server.

The example WireGuard server and client configuration can be found under the `wireguard` folder. Because WireGuard takes advantages of the kernel module, it is better to have it installed directly on the bare metal OS.

* Firewall allows port 80, 443, and of course, your SSH port. (You don't need to enable the WireGuard port system wide, as it is included in the `PostUp` section of the WireGuard configuration file)

There are different firewall applications for Linux distributions such as `iptables` and `ufw`, etc. Make sure the new firewall rules are persist, otherwise you will lose them after a reboot, for `iptables` you will need `iptables-persistent`.

## Folder Structure

* `blog` folder contains the the zola blog site.
* `nginx` folder contains the log and configuration files for the Nginx service.
* `wireguard` folder contains server/client configuration samples.
* `zola` folder contains the Dockerfile for building the zola container.

## Running Services

To start/stop the webserver & Jenkins server:

```bash
docker compose up webserver # --detach
docker compose down
```

To build the blog static site:

```bash
docker compose run zola build
docker compose restart webserver
```

To renew the SSL certificate:

```bash
docker compose run certbot renew
```

Disable any type of DNS proxy service before you run the renew command, if you are using CDN services such as Cloudflare.