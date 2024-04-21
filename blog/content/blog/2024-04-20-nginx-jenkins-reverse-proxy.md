+++
title = "Set up Jenkins and Nginx Reverse Proxy in Docker Containers"
[taxonomies]
  tags = ["docker"]
[extra]
  toc = true
+++

This post shows how to set up Jenkins in docker, then use the Nginx container to reverse proxy the Jenkins website.

## Requirements

* You have configured your Nginx container and it is able to show HTTP or HTTPS content when tested on your browser. You can check my previous post [Set up Let's Encrypt (Certbot) and Nginx in Docker Containers](/blog/nginx-certbot-docker) to setup Nginx and HTTPS.
* The Nginx container and the Jenkins container are **on the same computer**.

## Docker Compose YAML

Here is an example Docker compose file:

```yaml
services:
  # Nginx service
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro   # Nginx conf folder
      - ./nginx/log/:/var/log/nginx:rw        # Log folder
      - ./certbot/www/:/var/www/certbot/:ro   # Certbot challenge folder (needed by certbot)
      - ./certbot/conf/:/etc/letsencrypt/:ro  # Certbot folder (needed by certbot)

    networks:
      - local-net # Not required but recommended
    depends_on:
      - jenkins   # Need to know jenkins host for reverse proxy
  # Jenkins service
  jenkins:
    image: jenkins/jenkins:lts
    user: root
    volumes:
      - ./jenkins/config/:/var/jenkins_home:rw # Jenkins config folder
    networks:
      - local-net # Not required but recommended
networks:
  local-net: # Not required but recommended
```

The `local-net` is not required, since docker's default behavior is to share a single network between different containers[^1]. By adding `local-net` we restrict that only those service on `local-net` can access each other. This makes future management easier if more containers are added later on.

## Nginx Host Configuration

Here I assume you have already set up HTTPS for Nginx. If you want to se HTTP instead:

* delete the 301 redirection in the HTTP virtual host configuration.
* copy whole HTTPS `location` to HTTP part. 

Put this Nginx file at `./nginx/conf` folder, it is mounted to the `nginx` container in the `docker-compose.yml`.

```nginx
# HTTP
server {
    listen 80;
    listen [::]:80;

    server_name <jenkins.domain.name>;

    # Redirect to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# JENKINS HTTPS
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    http2 on;

    server_name <jenkins.domain.name>;

    ssl_certificate <path-to-fullchain.pem>;
    ssl_certificate_key <path-to-privkey.pem>;

    access_log            /var/log/nginx/jenkins.access.log;
    error_log             /var/log/nginx/jenkins.error.log;

    location / {
        # proxy_params file
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass          http://jenkins:8080;
        proxy_read_timeout  90s;

        # Fix potential "It appears that your reverse proxy setup is broken" error.
        proxy_redirect      http://jenkins:8080 https://<jenkins.domain.name>;
    }
}
```

## Restart Nginx Service

Once you have the `docker-compose.yml` and the `nginx` configuration files, you can then restart the service and check if Jenkins web page shows in browser.

```bash
docker compose restart webserver
```

## Postscripts

For most user cases, it is **NOT** recommended to have Jenkins directly exposed on the public internet. Since Jenkins has been found to have a list of security vulnerabilities in the history. 

Although Jenkins developpers can fix those security vulnerabilities swiftly, there will always be a time window for the bad actors to exploit those 0-day vulnerabilities[^2].

Instead, Jenkins web page is usually hosted behind a VPN. Only users who can access the VPN network can visit Jenkins web page. This method provides strong protection to Jenkins, since most VPN uses very advanced encryption and authentication algorithms to block out the non authorized users.

In my project's setup, my Jenkins web page is hosted behind the WireGuard VPN, you can read my post [Make Jenkins Accessible Only through WireGuard VPN](/blog/protect-jenkins) to know more about it.


[^1]: Reference from [Docker Compose Networking](https://docs.docker.com/compose/networking/) from [docks.docker.com](https://docs.docker.com/).

[^2]: A [zero-day](https://en.wikipedia.org/wiki/Zero-day_vulnerability) (also known as a 0-day) is a vulnerability or security hole in a computer system unknown to its owners, developers or anyone capable of mitigating it.