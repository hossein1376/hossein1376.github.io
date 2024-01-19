---
title: VPS Remote Browser
date: 2023-12-05T10:09:33+03:30
tags: [docker, browser, firefox, chrome, nginx]
---

Remote servers can be used in a variety of different ways. Host contents, serve requests, scrap data patriotically and much more.
But have you ever wondered if you can browse the web using your VPS? And I don't mean something like a v.p.n or proxy. 
Having an on-call browse on your personal server sounds like a thrilling idea, and it really is!  
Let's get into how to have a remote browser, inside another browser :)

![Neko Logo](/logos/neko.png "Neko Logo")

## Docker

We're going to use Docker, as it allows running applications without us worrying about dependencies or conflicts. 
It's a fantastic piece of technology, which I highly advise you to get more into.  

Anyway, you can install it via the [official script](https://get.docker.com/). To make sure, run `docker compose version`
to verify the installation. If it's good, let us move on.

## N.eko

I've seen a few projects providing remote browsers, but I came to like [n.eko](https://github.com/m1k1o/neko). It supports
plenty of browsers and architectures, and it doesn't limit itself to only browsers and has a wide range of other features.  
While we explore just the simplest of what n.eko has to offer, feel free to check out its docs for more.

## Installation

Connect to your server, and create a new folder to hold our compose file.

```shell
mkdir neko && cd neko
```

Use your editor of choice to create and edit a `compose.yaml` file with the following:

```yaml
services:
  neko:
    image: "m1k1o/neko:firefox"
    restart: "unless-stopped"
    shm_size: "2gb"
    ports:
      - "8080:8080"
      - "52000-52100:52000-52100/udp"
    environment:
      NEKO_SCREEN: 1280x720@30
      NEKO_PASSWORD_ADMIN: username
      NEKO_PASSWORD: password
      NEKO_EPR: 52000-52100
      NEKO_ICELITE: 1
```

Let's talk about customization! While there are many browsers to choose from. I'm going with `firefox`, but n.eko supports
most of the popular browsers. Screen size and login credentials are set here as well.  
Note that higher resolution will require more resources, as we will discuss shortly in the [Performance](#performance) section.
The good thing is, you can manually change the resolution inside the application as well.

Save the file and run `docker compose up -d`. After a few seconds, it will be running on the background and ready for the
next step.

## Nginx

It's essential to secure our online activity, so we're going to use Nginx to handle the reverse proxy and SSL certificates. 
If all this sounds unfamiliar to you, check out my post: [Nginx Setup, Reverse Proxy and TLS (SSL)](/posts/2023-11-08-nginx-setup/).

Here's a Nginx sample config file:
```nginx
server {
    listen 443 ssl http2;
    server_name <client.example.com>;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Protocol $scheme;
    }

    # cerificate and stuff
}
```

Make sure to replace your domain name, and change the `proxy_pass` if you had modified n.eko running port.  
Restart Nginx and visit your domain. You should be prompted with a login page. After filling in, you can request for 
control via the little keyboard icon at the bottom of the screen. Then, start surfing the net!

## Performance

Browsers inherently consume *a lot of* resources. With the aforementioned configs, I suggest a server with at least 2gb
of ram and 2 CPU cores. For higher resolutions and better performance, something like `1920x1080@30`, 8 CPU cores and
+4gb of ram will probably be needed.

## Conclusion

Harnessing the power of Docker, Nginx and cloud, we successfully ran a remote browser. It is accessible through the net
via an url. It's secured by a TLS certificate and is put behind a reverse proxy, with a login authentication.  
Thanks to open-source projects like n.eko, we have a wide range of browsers to choose from and options to customize. 
Make sure you have a strong enough VPS at your arsenal!

Hopefully you had fun as much as I had. As always, if you have any question, feel free to ask. Share your thoughts to let
me know what you think (I've been watching way too much YouTube, apparently; hit the like button and subscribe...?). 

Cheers, Hossein :)