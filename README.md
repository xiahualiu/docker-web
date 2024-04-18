# Nginx Jenkins Zola Docker Compose

My docker compose project for my Oracle Ampere A1 instance.

It contains several docker containers:

* [Nginx](https://nginx.org/) container, for serving website content such as Jenkins and blog.
* [Jenkins](https://www.jenkins.io/) container, CI of other projects.
* [Zola](https://www.getzola.org/) container, my blog site generator.

## Why Docker containers

The most straight forward answer is for security reasons, because containers are isolated from each other and the OS, even if one of them is compromised by external attacks, the bad actor cannot obtain the whole access to the system data.

And with the docker compose file, it is easier to redeploy the same services on multiple machines.

## Requirements

* Docker Compose version > 3.0.

Docker compose needs to be highier than 3.0 version.

* WireGuard VPN

The Jenkins server is hosted behind the WireGuard VPN for security reason. In order be able to access the Jenkins control website, the user must first use WireGuard VPN to connect to the interal `wg0` interface on the server.

The example WireGuard server and client configuration can be found under the `wireguard` folder. Because WireGuard takes advantages of the kernel module, it is better to have it installed directly on the bare metal OS.

* Firewall allows port 80, 443, and the WireGuard UDP port.
