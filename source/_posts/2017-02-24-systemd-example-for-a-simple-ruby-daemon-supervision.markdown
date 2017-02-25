---
layout: post
title: "Systemd Example For a Simple Ruby Daemon Supervision"
date: 2017-02-24 21:34:14 -0300
comments: true
categories: 
  - ubuntu
  - tutorial
  - example
  - systemd
  - ruby
  - supervision
  - daemon
---

Since the new `ubuntu` versions migrated from `upstart` to `systemd` in order to unify basic `Linux` service behaviors across all distributions we now have to deal with `systemd` as a default service manager/supervisor. In this article I will describe basics of how to write your first service on this platform.

<!-- more -->

Services directory location:

```
/lib/systemd/system/*.service
```

Logs location:

```
/var/log/syslog
```

It's even more convinient to fetch them with a special `journalctl` utility which reads a `systemd` journal:

```
$ journalctl --since "2017-02-25 00:00:00"
```

Now let's create a simple Ruby script which will run an infinite loop:

```
$ cd /home/username/
$ vim mydaemon.rb
```

```
$stdout.reopen('/home/username/mydaemon.log', 'a')
$stdout.sync = true
loop.with_index do |_, i|
  puts i
  sleep(3)
end
```

And start it as a daemon:

```
$ ruby mydaemon.rb &
[1] *pid*
```

Checking logs to make sure the daemon is running as expected:

```
$ tail -f mydaemon.log
0
1
2
3
^C
```

And stop it:

```
$ kill *pid*
```

It's time to add a `systemd` service which will start this daemon automatically and restart it whenever it was stopped or crashed.

```
$ vim /lib/systemd/system/mydaemon.service

[Unit]
Description=Simple supervisor

[Service]
User=username
Group=username
WorkingDirectory=/home/username
Restart=always
ExecStart=/usr/bin/ruby mydaemon.rb
```

In contrary to `upstart` `systemd` does not start new services automatically. You can check it's status with:

```
$ systemctl status mydaemon
● mydaemon.service - Simple supervisor
   Loaded: loaded (/lib/systemd/system/mydaemon.service; static; vendor preset: enabled)
   Active: inactive (dead)
```

And start it with:

```
$ systemctl start mydaemon
```

Now status should show mydaemon.rb was started and running with a pid assigned

```
$ systemctl status mydaemon
● mydaemon.service - Simple supervisor
   Loaded: loaded (/lib/systemd/system/mydaemon.service; static; vendor preset: enabled)
   Active: active (running) since *date*
   Main PID: *pid* (ruby)
   Tasks: 2
   Memory: 4.0M
   CPU: 52ms
   CGroup: /system.slice/mydaemon.service
           └─*pid* /usr/bin/ruby mydaemon.rb
```

If you now try to kill the process - supervisor should immediately restart it:

```
$ kill *pid* 
$ systemctl status mydaemon
● mydaemon.service - Simple supervisor
   Loaded: loaded (/lib/systemd/system/mydaemon.service; static; vendor preset: enabled)
   Active: active (running) since *date*
   Main PID: *new_pid* (ruby)
   Tasks: 2
   Memory: 4.0M
   CPU: 49ms
   CGroup: /system.slice/mydaemon.service
           └─*new_pid* /usr/bin/ruby mydaemon.rb
```

Helpful resources:

[Systemd man](https://www.freedesktop.org/software/systemd/man/systemd.service.html)<br>
[Systemd for upstart users](https://wiki.ubuntu.com/SystemdForUpstartUsers)

Bigger systemd service example:

```
[Unit]
Description=My description here

After=another.target
After=some.service

[Service]
Type=simple # forking/oneshot/dbus/notify/idle

PIDFile=/absolute/file/name

User=username
Group=groupname

WorkingDirectory=/home/username/some/dir

Restart=on-failure

Environment=ONE='one' "TWO='two two' too" THREE=
EnvironmentFile=/path/to/file/with/variables

ExecStartPre=/run/some/command
ExecStartPre=/run/some/other/command
ExecStart=/usr/bin/node /opt/webhook/server.js
ExecStartPost=/lastly/run/this

[Install]
WantedBy=multi-user.target
```

Restart options:

{% img /images/systemd_restart_settings.png %}

Upstart <-> Systemd commands cheatsheet:

{% img /images/systemd_upstart.png %}
