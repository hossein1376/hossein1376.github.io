---
title: SSH Security - Part 1
date: 2024-01-19T10:46:03+03:30
tags: [security, ssh, linux]
---

I don't think there are any reasons for me to lecture about the importance of online security; it's public knowledge how
crucial it is. Managing a VPS requires taking multiple security measures to be sure about safety of your infrastructure
and data.  
and data.  

I'm planning a series where I talk about different security layers, meant to be stacked over each other. In this one, 
we're going to start from ground up and set up the basic minimum and the essential settings.

![SSH Logo](/logos/ssh.png "SSH Logo")

## Firewall

By default, all the ports (there are 65,535 of them) are open and accessible by the public. You can think of ports as
gateways, _or doors_, which are used to communicate data between services. Both local and online.  
The common sense is to close them and allow only a few trusted individuals from some ports. Firewalls do exactly
that!

Mind you, firewalls can be quite complex and not so easy to work with, especially for those of us who aren't an IT
expert. That's why `ufw` is so amazing. It stands for 'uncomplicated fire wall' and it allows us to manage the underlying
system's firewall with just a few commands. `ufw` is very popular choice and is included by default in some distros,
and can be installed from the official repository otherwise.  
Run `apt install ufw` or the equivalent for the installation.

Now, if you run `ufw status` you'll be notified that it's not enabled yet. Go ahead and activate it by `ufw enable`.  
So, here's the important part. All incoming requests will be denied, unless you specifically tell it not to. We can
do that by adding rules.  

Go ahead and open your desired ports with `ufw allow PORT`. If you want to remove a role, run `ufw status numbered` and
then `ufw delete NUMBER`.

## New Users

The user `root` comes on almost all distros, and have the highest permission. These all make it a prime attack target.  
It's crucial to create new users as soon as you get your hands on your server.  

To do that, run `adduser username` command and follow its prompts to finish the new user creation. You can change users
with `su username` command.

## SSH Key-Based Authentication

It's a preferred approach, compared to text passwords. Run `ssh-keygen` to create a new key for the user. By default,
it puts them inside `~/.ssh` folder. The `id_rsa` is the super confidential key that must be kept private. While, the
`id_rsa.pub` file and its content can be put into other servers, so you can access them without knowing the password!  
On remote server create the `authorized_keys`, if not exists, and put the contents of `id_rsa.pub` file there. It should
be exactly in one line without any line breaks or extra characters, otherwise it won't work.


## SSHD Config

The configuration file located at `/etc/ssh/sshd_config` should be edited as soon as you get the chance. Let's go over
some of the essential options.  

- `Port` defaults to 22, and I highly recommend to change it to something entirely random. (and no, 23 is not that
clever)  
**Note:** remember to open that port on your firewall as well, **otherwise you'll be locked out of your server!**  

- `PermitRootLogin` should be set to `no`. Every server has a root user, so it's the easiest target, among other reasons.  

- Preferably, set `PasswordAuthentication` to `no` if you've already enabled SSH key-based authentication.  

## What's Next?

Firewall selectively allows some of the incoming requests. With ssh keys, the need for memorizing and typing passwords
are removed. By modifying `sshd` settings, we reduce chance of SSH breaches.

We barely have scratched the surface yet. Still, with the settings we did, our server is in a much better security state. 
In the next steps, I'm going to introduce two Intrusion Prevention System (IPS) to ban more persistent attempts.