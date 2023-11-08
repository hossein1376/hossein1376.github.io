---
title: Nginx Setup, Reverse Proxy and TLS (SSL)
date: 2023-11-08 17:29:16
tags: [nginx, certbot]
---

Join me as we dive into what Nginx and TLS is, how to install Nginx and configure it and above all, how to use it to serve up our application.
I assume you have a Linux server with root access, some knowledge with the Terminal, a domain you own and an application that is already running inside VPS.

## What is Nginx

[Nginx](https://www.nginx.com) is a widely used web server which can act as load-balancer, reverse proxy, HTTP cache, mail server and much more. It's simple and fast, as well as highly customizable. It was originality developed in 2004, written in C and was published as free and open-source.  
Currently, Nginx is the most popular choice for web servers, followed closely by Apache. It supports variety of operating systems, all versions of HTTP, has great integration with other services, and it's super easy to get started with it.

## What is Reverse Proxy?

Reverse Proxy is an application which is placed between your application and the internet, act as a middle man. Why would you want to do that, you ask? Well, it turns out that there are _many_ reasons to do so. It increases scalability, helps with performance and security, among other reasons.  
In our case, goal is to run an application on a local port, which may be blocked by our system's firewall. Add TLS security layer, optionally block malicious requests or distribute traffic among different applicatons.

## What is TLS?
As a quick note, I am by no means a security expert. The following is meant to a very high-level overview of the matter.

TLS (Transport Layer Security) and SSL (Secure Sockets Layer) are often used interchangeably, though they differ with each other in more than one area. But for the sake of this article, we won't go into details and instead, focus on why HTTPS is important.  
You see, with the good old HTTP protocol, your internet traffic is unencrypted. Meaning that, it's almost trivial for anyone to view your online activities. Needless to say, that's a huge concern and needs to be taken care of immediately.
As with almost everything in the security business, there are huge fields and studies dedicated to the subject. The result being TLS protocol which prevents your data from eavesdropping and tampering.  

In this approach, each website has a certificate that was issued especially for them. When you enter its url into the address bar and press enter, your browser will exchange keys with the remote server. Further requests and responses, will be encrypted with those keys. 
So, your data will look gibberish for anyone noisy enogh to view it. Though, the website you are viewing can be sniffed out. But what you're doing and the data that is being transferred will be encrypted and secure for most use-cases.

While you can generate your own certificates and they're fine for testing or developing purposes, they're not suitable for deploying. Every modern browser will immediately warn visitors about it and they generally should be avoided.

## Enough Theory, Installation Time

Luckily, nginx is popular enough to be available on almost all distributions. Install it with your package manager of choice:

```bash
apt install nginx
# or
dnf install nginx
# or
pacman -S nginx
```

After installation, it's highly recommended to start and enable it.

```bash
systemctl enable nginx
systemctl start nginx
systemctl enable nginx
```

If you see any error messages, I can think of these possible reasons: 
1. check your Nginx installation 
2. check if another web server is already installed 
3. check if another application is running on port 80 or 443

If you are using a firewall, make sure to allow port 80 and port 443.

## Add Domain to Nginx

A big advantage of using web servers is you can have multiple applications listening on the same port, usually the standard 80 and 443. For this feature, we need to utilize the `server block` in Nginx. 

Though you can create Nginx configuration files everywhere, it's common to put them in `/etc/nginx/sites-avalible/` and name the file after its domain. I use `nano` here, but you can use any other editor you like.

```bash
nano /etc/nginx/sites-avalible/example.com
```

Make sure to replace `example.com` with your own address. Check if you have the correct authorization, or add `sudo` to it.

## Configuration FIle

Getting used to Nginx configuration files take a little time and practice. Most of your server block config files will end up looking almost identical, with small variations.  
A big gotcha is the trailing semicolon; make sure to always check for it. I have included a sample, take a look:

```nginx
server {
    listen 443;
    listen [::]:443;

    server_name <example.com>;
        
    location / {
        proxy_pass <app_server_address:app_port>;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    if ($host = <example.com>) {
        return 301 https://$host$request_uri;
    }
   server_name <example.com>;

    listen 80;
    listen [::]:80;

    return 404;
}
```

There are 4 instances where **you need to replace**, indicated with the `<...>` symbols.
This file will serve your application on port 443, and will redirect all the requests on port 80 to 443. For now, there's no reason to do so, as we're not using a TLS certificate. But we'll use the second block to redirect HTTP to HTTPS later on.

## Tell Nginx About it!

Nginx looks into the `sites-enabled` directory for the sites it serves, so we need to link our newly created config file there with this command:

```bash
ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

Just to make sure, let's run a check to see whether our configurations are correct or not:

```bash
nginx -t
```

If everything is all right, restart Nginx

```bash
systemctl restart nginx
```

Done! If you visit your domain now, you should see your application running.

## Certificates with Certbot

Not all certificates are free, as they are issued only by a few trusted Certificate Authorities. Cue, [Let's Encrypt](https://letsencrypt.org/) enters!
By using [Certbot](https://certbot.eff.org/), you can get free and legit certificates for your domains. Catch is they're valid for only 3 months, but the good thing is certbot will attempt to renew them whenever they're about to expire. So, no worries there :)

Now, there are some different versions and installation methods for certbot, but I'll stick to the official guide. Remove any other installations of certbot you have. 

```bash
apt remove certbot
```

To begin, install snap:

```bash
apt install snap
```

Install certbot

```bash
snap install core; snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
```

### Note

It may be possible for certbot to integrate nicely with Nginx, reading server configs and even automatically update configuration files for you. While it's much more convenient, it's totally possible for it not to work. For the sake of this article, I assumed you need to update the configurations manually.

Phew. Finally, we are ready for those sweet sweet certificates. Make sure your domain's DNS is pointing to the current server address, before running:
```bash
certbot --nginx -d example.com 
```

The first time, it may ask you some questions. Like your email address, agreeing to TOS and such.
Hopefully if everything goes according the plan, you will see something along these lines

```text
...
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at: /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2024-02-01.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.
...
```

Nice. See those two lines with the certificate and key path? Copy them somewhere handy as we will reference them everytime we want to use certificates for this domain.

## Add Certificate to Nginx

Now, it's time to update our Nginx configuration file.

```bash
nano /etc/nginx/sites-avalible/example.com
```

We need to let nginx know that it needs to enforce TLS on port 443, and where to find the certificate and its related config files.

```nginx
server {
    listen 443 ssl; # <- ssl was added here
    listen [::]:443 ssl; # <- here as well

    server_name <example.com>;
        
    location / {
        proxy_pass <app_server_address:app_port>;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # these 4 are new!
    ssl_certificate /etc/letsencrypt/live/<example.com>/fullchain.pem
    ssl_certificate_key /etc/letsencrypt/live/<example.com>/privkey.pem
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem
    include /etc/letsencrypt/options-ssl-nginx.conf # this file (not line!) may or may not be missing for you
}

server {
    if ($host = <example.com>) {
        return 301 https://$host$request_uri;
    }
   server_name <example.com>;

    listen 80;
    listen [::]:80;

    return 404;
}
```

Now, a couple of important issues. First of all, replace all the placeholders with your domain and application address. 
Then, `ssl` has been added after `443` to tell nginx about the certificate situation and all. 
In addition, four new lines have been added. First two are the same path we saw earlier when we received our certificate. Conveniently, fill then in.
Next two lines though, are not as clear as one might hope. They provide configurations needed to serve application with the certificate.

Save the file and run `nginx -t`. If everything is ok, great! Otherwise, you probably miss the `/etc/letsencrypt/options-ssl-nginx.conf` file. You have two ways; create a file at that location and paste the following there, or replace the line `include ...` with these:


```nginx
ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
```

Don't forget to restart Nginx!

```bash
systemctl restart nginx
```

## Congratulations!

That was some work, huh? It may look quite a lot of trouble, but it definitely worth it. Your application is now running behind a reverse proxy, serving on HTTPS and will redirect all HTTP traffic too!

You can add more configurations such as IP or region blocking, rate limiting, load balancing and much more if you like to.

## Conclusion

We talked a little about history of Nginx and why TLS is important. We proceeded to install Nginx and config it. Next, we installed certbot and got ourself a shiny new certificate. Lastly, we added the certificate to the Nginx configuration file and successfully served the application!
I hope you had fun as much as I did. If you have any questions, problems or suggestions; don't hesitate to get in touch with me :=)

Cheers, Hossein.