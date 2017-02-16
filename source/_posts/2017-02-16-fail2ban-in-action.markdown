---
layout: post
title: "fail2ban In Action"
date: 2017-02-16 01:12:19 +0100
comments: true
categories:
  - ubuntu
  - tutorial
  - example
  - fail2ban
  - firewall
  - iptables
---

Just checked the journal of my server and discovered it's under(luckily unsuccessful) attack for quite a period of time. Quick check through the journal has revealed numerous ssh login attempts:

<!-- more -->

```
$ journalctl --since "2017-02-15 11:50:00"
```

{% img /images/fail2ban_1.png %}

Well, it is a nasty situation. Putting all those ips to `iptables` manually would be painful and would require to check the journal every day for new attacker's ip entries. Fortunately, this task is easily solvable with `fail2ban` utility.

```
$ apt-get install fail2ban
```

It works out of the box under `systemd` supervision, but it is a reasonable idea to check config and adjust some settings like `bantime`/`findtime` or `recidive` block:

```
$ cd /etc/fail2ban
$ cp jail.conf jail.local  # Generating local config out of default one
$ vim jail.local           # Make your changes and save it
$ service fail2ban restart
```

Now it's time to check logs

```
$ tail -f /var/log/fail2ban.log
```

Boom, banhammer in action:

{% img /images/fail2ban_2.png %}

Corresponding entries in `iptables`

```
$ iptables -L
```

{% img /images/fail2ban_3.png %}

Last detail: it seems like attacks were incoming from machines that are already infected. Anyway, problem is now solved.
