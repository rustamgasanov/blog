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

Since the new `ubuntu` versions migrated from `upstart` to `systemd` in order to unify basic `Linux` service behaviors across all distributions we have to now deal with `systemd` as a default service manager/supervisor. In this article I will describe basics of how to write your first service on this platform.

<!-- more -->
