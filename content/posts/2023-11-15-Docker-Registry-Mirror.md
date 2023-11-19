---
title: Docker Private Registry and Cache Mirror Registries
date: 2023-11-15 18:48:25
tags: [docker, nginx]
---

Have you ever wondered about having your very own private docker registry?
Or rather, have you ever faced issues pulling images directly from DockerHub?
What if I tell you, you can have all that, and it's really easy to configure it?
Join me as we dig deeper into configuring a docker private registry and also how to use it as a cache-through proxy.

## Nginx

We're going to use Nginx as a reverse proxy and to secure our registry with an SSL certificate.
If all this sounds unfamiliar to you, I advise you to check out my other post, [Nginx Setup, Reverse Proxy and TLS (SSL)](https://blog.godlynice.ir/2023/11/08/2023-11-08-Nginx-Setup/). 

## Docker

It shouldn't come out as a surprise that we need Docker installed for running up a docker registry.
There are a few options out there on how to proceed with the installation.
In my experiences, following the [official script](https://get.docker.com/) is the best approach available.

## Registry

Let's start by creating the directory,

```shell
mkdir ~/registry
```

Change directory there.

```shell
cd ~/registry
```

Create a docker compose file. I use `nano` here, but you can use any other editor you prefer.

```shell
nano compose.yaml
```

Before going any further, I like to mention that older and newer versions of `docker-compose` have quite a few different APIs.
If `docker compose version` is working for you, you're probably fine. Otherwise, I suggest updating your [Docker](#docker) installation.

Insert the following inside the `compose.yaml` file:

```yaml
services:
  registry:
    image: registry:2
    ports:
    - "5000:5000"
    restart: always
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - ./data:/data
      - ./config.yml:/etc/docker/registry/config.yml
```
You can change the port if you like to. Remember to put the right value inside the Nginx config file as well.
As you can see, we have mentioned a config file which currently doesn't exist. Best to create it.

```shell
nano config.yaml
```

And populate it with the following:

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

We're almost there. Run `docker compose up` to verify it's working correctly.
If it is, bring it down with `docker compose down` and bring it back up again in detached mode with the `docker compose up -d` command.

## Domain

I'm not going into Nginx details. Create a new server block and include the following:

```nginx
server {
    location / {
        if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
            return 404;
        }

        proxy_pass                          http://localhost:5000;
        proxy_set_header  Host              $http_host;
        proxy_set_header  X-Real-IP         $remote_addr;
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;
        proxy_read_timeout                  900;
    }

    server_name <registry.domain.com>;
}
```

You might want to tweak Nginx settings to allow for higher upload size limit. Open up:

```shell
nano /etc/nginx/nginx.conf
```

In the http section, change the upload size limit to `16GB`

```nginx
...
http {
        client_max_body_size 16384m;
        ...
}
...
```

Restart the Nginx and head over to `registry.domain.com/v2` to be greeted with the none too exciting `{}`.
Good news is; this shows everything is working as intended. You can now and pull and push your images from your own registry.  
You can set up authentication for your registry as well, but due to the complications it causes (discussed at [Complications](#complications)),
I decided to skip over that.

## Pull Through Cache

If you intend to use your registry as an image proxy, we need to update the `config.yaml` file:

```yaml
...
proxy:
  remoteurl: https://registry-1.docker.io
```

Check your indentation, as `proxy` field must have none.  
Run `docker compose down` and `docker compose up -d` to restart your registry.

Now, it's time to let Docker know about this new registry mirror.
On your own machine and any others that may want to use this mirror, open this file or create it if not exist

```shell
nano /etc/docker/daemon.json
```

And update it with:

```json
{
  "registry-mirrors": ["https://<registry.domain.com>"]
}
```

That's it! From now on, images will be pulled from your mirror. Docker will continue using DockerHub as a fallback nonetheless.

## Complications

It has been observed that a registry mirror won't be used if there is an authentication placed on it.
That's the reason I did not use authentication for our registry in the first place.
There is a long discussion about this matter on [GitHub issues](https://github.com/moby/moby/issues/30880) dating back to 2017.
Some astute users have found out that docker actually uses your DockerHub credentials for mirrors as well.
So work around? Use the same credentials on both! It seems that this approach will in fact work, but I don't really like it.
It's your choice whether you prefer to keep your registry public or use the same credentials on it as your DockerHub account.

## Conclusion

We created our registry config files, put it behind a reverse-proxy and checked if it works.
As an extra step, we upgraded our registry to a registry mirror.
We also discussed a limitation related to registries with authentication and a possible fix for it.

I hope you had fun as much as I did! As always, you can contact me if there's a problem or if you have any suggestions.
Cheers, Hossein.
