---
title: 'SSH Security - Part 2: Fail2Ban'
date: 2024-02-08T21:51:09+03:30
tags: [security, ssh, linux, fail2ban]
---

In my last [post](/posts/2024-01-11-ssh-security-part-1/), I talked about basic SSH security settings. Those settings will get you far, but they may not
be enough, especially if you have open ports for services like SSH.  
Persistent attackers will port-scan, then they'll try to brute force their way into the VPS.

All login attempts will be logged and recorded inside the server, wouldn't it be convenient to ban those who are
continuously failing?

![Fail2Ban Logo](/logos/fail2ban.png "Fail2Ban Logo")

## Fail2Ban
[Fail2Ban](https://github.com/fail2ban/fail2ban) is an open-source project written in Python which reads log files, process them and by modifying
firewall rules, it bans IP addresses which make too many failed requests. It does all this out of the box, and it barely
causes an overhead!   
You can install Fail2Ban with your package manager of choice. Make sure to start and enable it as well.  

While I'm talking about SSH here, the same principle applies for all services with a login interface. Fail2Ban supports
many popular applications and has a jail for them in its config file. But, it doesn't know which applications are
installed on our system and which ones we want to protect. So, it's up to us to activate the desired ones.

## Jail

You can think of each jail as handful of commands that tell Fail2Ban where it can find the log files for that
application, and what it should do. Since we are going to modify them, we need to create a local version of the 
`jail.conf` file, so it wouldn't be overwritten later when we do an update.

```shell
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

and then, let's start editing this new file.

```shell
nano /etc/fail2ban/jail.local
```

You can ignore all the top level comments. Down in the `[DEFAULT]` section, we are going to tell Fail2Ban how to behave.    
By reading comments of each option, you can choose whether to activate it or not. For example, by uncommenting
`bantime.increment = true`, if the offender's IP has been previously banned, it will receive incrementally longer ban
times! Cool, right?

There are three values I like to change. I believe `bantime` must be much longer to deter bad actors. You can set it on
`24h` or even longer (`168h` in my case). Keep in mind that if there's a false positive, or you mistyped your password 
a couple of times, this is the duration you may get banned! You want it to be long enough to a punishment, but not too 
long.  
`findtime` is the time window of previous failed attempts, and `maxretry` is how many times one can fail before they get
banned. I set them to `1h` and `3` respectively. You do you.

There are many more options to discover. For now, I'll skip over these to focus on SSHD Jail settings. Scroll down
until you reach the jails section and the `sshd` one. This is how it should look like after you editing it:

```yaml
mode    = aggressive
port    = 22
logpath = %(sshd_log)s
backend = %(sshd_backend)s
enabled = true
```

If you have changed your SSH port, as I had strongly advised, then replace that here instead of 22.  
Then, save this file and exit. We're done :)

## Punishment!

Fail2Ban is in action! It will scan log files such as `/var/log/auth.log` or `/var/log/secure` and will ban offenders.
Even if none gets banned immediately, check back later and run

```shell
fail2ban-client status sshd
```

to get details of our jail. Also, we can view Fail2Ban's own log with `less +FG /etc/fail2ban` to have a realtime show
of incoming request and all that action happening silently everyday on the internet.

## The Catch

Fail2Ban is great and I used it a for long time. Still, it is a passive approach to the security. That's why I moved all
my servers to a different tool. Make sure to check out my next post where I'll talk about an all new exciting tool.

My regards, Hossein.