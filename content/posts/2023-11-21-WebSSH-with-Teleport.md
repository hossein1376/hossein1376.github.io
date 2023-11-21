---
title: WebSSH With Teleport
date: 2023-11-21T23:19:51+03:30
tags: [ssh, web_ssh, teleport]
---

The easiest way to connect to a remote server is using SSH (secure shell). It comes preinstalled on Linux and Mac,
and the protocol takes care of almost anything. You still need to provide a password or a matching ssh key though.  
On the other hand, if you're using Windows or Android, or if you find yourself on another machine, things get challenging.
There are many solutions for these kinds of situations, what I'm going to talk about today is called WebSSH.

## What is WebSSH

To put it simply, you will get a fully working terminal inside your browser. It's still ssh behind the scenes, with some added steps.
A huge advantage of this approach is the widespread support of browsers everywhere! If your device has a browser,
It probably can be used for connecting to your servers as well.  
Looking up online, you can find different applications and libraries providing this service. I suggest search WebSSH online and
read some about this technology and projects around it.

## Introducing Teleport

[Teleport](https://goteleport.com/) isn't strictly used for web ssh. It's an open-source project for control and audit infrastructure.
Meaning your servers, Kubernetes clusters, running services, databases, and more. It has perineum paid tier with 14-day free trial,
and a self-managed version with is more than enough for most use-cases.  
While I'll focus on the ssh part of its capabilities, I advise you to check its docs and learn more about how it does its services
and what else you can do with Teleport!

## Install

Installing Teleport is straightforward. Run the following script on your server.

```shell
curl https://goteleport.com/static/install.sh | bash -s 14.2.0
```
### Note

At the time of writing this article, the latest version is `14.2`. Teleport has fast release cycles and addresses vulnerabilities quickly,
so it's more than likely there's a new version as the time you're reading this.  
Check out the [download](https://goteleport.com/download/) page for an up-to-date script link.

Installation takes a few minutes and will need around 500 megabytes of storage, which can be a little too much if you don't have
space to spare. But in my experience, with all the feature Teleport presents, it is well worth it.

## Configuring

Run `teleport configure` to create a configuration file. We will update it

```shell
sudo nano /etc/teleport/config.yaml
```

The following is an example file that you need to modify before using. Four instances between `<...>` needs to be replaced by you.

```yaml
version: v3
teleport:
  nodename: <Name>
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  ca_pin: ""
  diag_addr: ""
auth_service:
  enabled: "yes"
  cluster_name: "<Cluster Name>"
  listen_addr: 0.0.0.0:3025
  proxy_listener_mode: multiplex
  authentication:
    second_factor: on
    webauthn:
      rp_id: <teleport.domain.com>
ssh_service:
  enabled: "yes"
  commands:
  - name: hostname
    command: [hostname]
    period: 1m0s
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:3080
  public_addr: <teleport.domain.com>:443
  https_keypairs: []
  https_keypairs_reload_interval: 0s
  acme: {}
```
`nodename` is the name that will be displayed inside the `Resources` page.  
`cluster_name` cannot be changed later and usually is the name of your domain by convention.
`rp_id` and `public_addr` are your domain address that will be used to access Teleport.  

Save the file, start and enable the Teleport service,

```shell
sudo systemctl enable teleport
sudo systemctl start teleport
```

You can view its logs with:

```shell
sudo journalctl -fu teleport
```

It takes a few seconds to completely boot up.

## Domain

I have previously written about Nginx and SSL certificate, you can [read it here](https://blog.godlynice.ir/posts/2023-11-08-nginx-setup/).  
I assume you have a domain, like `teleport.domain.com`, pointing to the current server address. Nginx config file will look like this:

```nginx
server {
    server_name <teleport.domain.com>;
    client_max_body_size 1024M;

    location / {
        proxy_set_header Host $http_host;
        proxy_pass https://127.0.0.1:3080;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
```

Get a valid SSL certificate for it as well. Now, your Teleport instance can be accessed from `teleport.domain.com`!  
You can put the domain behind a CDN as well if you would like to.

## Create New User

In this step, we create a new user with username `admin`, allow ssh access to users `root` and `ubuntu`.  
Modify this command based on your own preferences.

```shell
sudo tctl users add admin --roles=editor,access --logins=root,ubuntu
```

You will get a link for signup. Open it in a browser and complete the registration.
Teleport enforces the use of two-factor authentication by default and I completely support this. 
If someone gets access to your account, they'll have **full control** over everything! Better safe than sorry.

While it's possible to disable 2FA, I strongly advise against it.
Define new users with different roles and access level in the same manner.

## Resources

The current server will be in the `Resources` page, and you can ssh to it using the web interface.
As you can see, it's a fully functional terminal inside the browser. Magical!

Other servers and resources can easily be added too. Follow the instructions and paste the link into desired servers.
You can rename each service in `config.yaml` file, `nodename` field. Remember to restart Teleport, both server and client.

```shell
sudo systemctl restart teleport
```

## More features

As I previously mentioned, you can access control almost all of your infrastructures using Teleport.
Also, create new users with varying roles and access level. List ssh sessions and even replay them like watching YouTube video.
Join ongoing sessions as a collaborator or observer and much, much more.  

I leave it to you to explore more functionality and use-cases.

## Conclusion

In this article, we learned about an exciting open-source project called Teleport. Installed and configured it,
with our knowledge of Nginx and Certbot, with served it behind a reverse proxy and a TLS. With the possibility of using CDN.
We successfully connected to our server via a web interface, also added other servers and configured them as well.

Hopefully, this all comes handy to you, dear reader. As always, let me know what you think and I hope you had fun!

Cheers, Hossein.