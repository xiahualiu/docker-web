+++
title = "Set up Let's Encrypt (Certbot) and Nginx in Docker Containers."
date = 2024-04-18
draft = false
[taxonomies]
  tags = ["Docker"]
[extra]
  toc = true
	keywords = "Docker, Nginx, Certbot, Let's Encrypt"
+++

This post shows how to get Let's Encrypt SSL certificates for your self-hosted website on the Nginx container.

## Requirements

* You have ssh access to your server's command line.
* You have at least one active domain name, and the DNS records for all domain names are set correctly.

For example, if you brought `google.com` TLD (Top Level Domain), you need to set up these DNS A/AAAA records on DNS providers, such as `blog.google.com`, `www.google.com`, `google.com`, `jenkins.google.com` to the correct destination IPs. This is required for certbot to issue SSL cert. If you don't have a TLD, a subdomain name is OK as well, but less secure.

If you are using Cloudflare DNS service, make sure you have disabled the DNS Proxy - all records are shown as **DNS only - reserved IP** under the *Proxy status* column.

## Writing Docker Compose

The bare minimum `docker-compose.yml`:

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
      - ./nginx/log/:/var/log/nginx:rw        # Nginx Log folder
      - ./certbot/www/:/var/www/certbot/:ro   # Certbot challenge folder
      - ./certbot/conf/:/etc/letsencrypt/:ro  # Certbot output folder
  # Lets Encrypt service
  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw   # Certbot challenge folder
      - ./certbot/conf/:/etc/letsencrypt/:rw  # Certbot output folder
    depends_on:
      - webserver # Need webserver to run
```

## Testing Nginx

Run `docker compose up webserver` to see if there is anything wrong with the nginx container.

Note you cannot access the default Nginx index page yet, because the default nginx configuration files are not in the container, due to the first `./nginx/conf/:/etc/nginx/conf.d/:ro` volume mounting instruction. Since your `./nginx/conf` folder is empty, the nginx conf folder inside the container is empty as well.

Let's create a simple configuration file in the `./nginx/conf` path, let's assume `./nginx/conf/app.conf`.

```
server {
    listen 80;
    listen [::]:80;

    server_name <sub.domain1> <sub.domain2> ... ;

    # Needed for Lets Encrypt, keep for cert renewals.
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Test connectivity with 404
    location / {
        return 404;
    }
}
```
(Replace `<sub.domain1>`,etc. with your real domain names)

Run `docker compose up webserver` or `docker compose restart webserver`. And input `http://<sub.domain1>`, `http://<sub.domain2>`, etc. in your browser to see if you received 404 error messages. (Sometimes the browser will ask if you want to proceed with accessing insecure sites. You need to choose yes, otherwise you won't be able to see 404 messages.)


### Troubleshooting

#### Firewall issues
If you didn't see the 404 page, or it shows there is no connection to the host, this usually indicates some firewall issues. If you are using cloud providers such as Google Cloud Platform, AWS, Oracle Cloud, make sure you have enabled the `80` and `443` ports in the Virtual Network section on the control panel; also you need to enable `80` and `443` in OS, depending on the firewall application you have, it may be `iptables`:

First check if `iptables` already allows 80 and 443:

```bash
sudo iptables -L INPUT -n
```

If there is no such rules **AND** the last rule is DROP/REDIRECT to PROHIBIT, you need to add 80 and 443 to the INPUT chain:

```bash
sudo iptables -I INPUT 1 -m state --state NEW -m multiport -p tcp --dports 80,443 -j ACCEPT
sudo ip6tables -I INPUT 1 -m state --state NEW -m multiport -p tcp --dports 80,443 -j ACCEPT
```

(Remember to save it by using `iptables-save` otherwise it won't persist after reboot)

Or `ufw`:

Check if `ufw` has 80,443 rules:

```bash
sudo ufw status
```

If ufw is `Active`, and the default rule is `DENY`. Then check if you have 80,443 there, if not, add them by:

```bash
sudo ufw allow 80,443/tcp
```

#### DNS issues

There could also be a DNS resolving issue. You can test your domain by running `dig` command. (You need to install `dig` if you don't have it)

```bash
dig <sub.domain1> # sub.domain2, etc.
```

Also, Windows PowerShell has a nice command `Resolve-DnsName` you can use to test if your DNS is correct:

```bash
Resolve-DnsName <sub.domain1> # sub.domain2, etc.
```

If the output IP addresses are not correct, you need to re-visit the DNS provider and make sure all DNS records are good. Note changes to DNS records can take up to 24 hours to synchronize across different regions.

## Run Certbot

After you can see the correct Nginx page, you are halfway there!

The `certbot` container can issue and renew SSL certificates for your sites now. First let's do a dry run:

```bash
docker compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d <sub.domain1>,<sub.domain2>,...
```

There will be several questions popped up, such as your email address, accept TOS, etc. Answer all of them then, wait for `certbot` finish.

If there are no errors, you can then remove the `--dry-run` parameter and run again:

```bash
docker compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d <sub.domain1>,<sub.domain2>,...
```

The output SSL pem files will be in `./certbot/conf/live/<sub.domain1>` folder, there will only be **ONE** set of certificates for all of your domain names. You need to change the owner of the `./certbot` folder otherwise docker cannot mount the new certbot files to the nginx container.

```bash
sudo chown -R $USER ./certbot
```

### Renew SSL certificates

```bash
docker-compose run --rm certbot renew
```

### Common Questions

> Why run `cerbot` with `certonly` instead of `--nginx`?

This is because the `certbot` `--nginx` actually queries and modifies the nginx configuration files, and because nginx is in another container, `certbot` has no idea where `nginx` is and will return error.

Although there are other ways to work around and make `--nginx` option work, I highly recommend using `certonly` here and only getting the SSL certificates from Let's Encrypt CA. Write your own HTTPS configurations later on. This gives you better control over your sites.

## Update Nginx HTTPS Configuration

After `certbot` issues your domain certificates successfully, you can then update your Nginx configuration to enable HTTPS:

```
server {
  listen 80;
  listen [::]:80;

  server_name <sub.domain1> <sub.domain2> ... ;

  # Needed for Lets Encrypt, keep for cert renewals.
  location /.well-known/acme-challenge/ {
    root /var/www/certbot;
  }

  # Redirect to HTTPS sites
  location / {
    return 301 https://$host$request_uri;
  }
}

# Example HTTPS static site conf
server {
  listen 443 default_server ssl;
  listen [::]:443 default_server ssl;

  http2 on;

  server_name <sub.domain1>;

  root /var/www/public/; # Example public folder
  index index.html;      # Index html

  ssl_certificate /etc/letsencrypt/live/<sub.domain1>/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/<sub.domain1>/privkey.pem;

  location / {
    try_files $uri $uri/ =404;
  }
}

# Example HTTPS reverse proxied site conf
server {
    listen 443 default_server ssl;
    listen [::]:443 default_server ssl;

    http2 on;

    server_name <sub.domain2>;

    ssl_certificate /etc/letsencrypt/live/<sub.domain1>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<sub.domain1>/privkey.pem;

    location / {
      # proxy_params
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass          http://<app-header>:<port>;
      proxy_read_timeout  90s;
      proxy_redirect      http://<app-header>:<port> <sub.domain2>;

      allow 10.66.66.0/24; # Only allow VPN Peers to visit
      deny all;            # Deny all other public IPs
    }
}
```

If you don't have the site ready yet, you can use the same `404` error code as the HTTP section and modify the `location` field later.

Then restart Nginx container by:

```bash
docker compose restart webserver
```

Try accessing your site now, see if it is secure in your browser!

## Make it More Secure

You can still make it better. There are many SSL hardening articles online, such as disabling unsafe ciphers, etc.

You can test your site at [immuniweb](https://www.immuniweb.com/ssl/) to find any vulnerabilities and use corresponding Nginx configurations to eliminate them.

You can also find my Nginx configuration [here](https://github.com/xiahualiu/docker-nginx-jenkins-zola/blob/main/nginx/conf/app.conf) for this blog site if you need any references as well.

## Postscripts

You should **NOT** enable Cloudflare DNS proxy for all domains during this whole process, and you should disable it as well **BEFORE** you renew your SSL certificates in the future if needed.

You can, however, enable Cloudflare DNS proxy after you set HTTPS to enable CDN service from Cloudflare, but you need to choose the **FULL** architecture in the *SSL/TLS->Overview* tab, or you will see "too many redirects" error when open your website in browsers.

