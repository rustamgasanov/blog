---
layout: post
title: "How to use ERB with ECO"
date: 2015-11-04 19:39:31 -0500
comments: true
categories: 
  - erb
  - eco
  - rails
  - backbone
  - marionette
  - templates
---

In my current project we use Rails with Backbone(Marionette). For Rails, it's essential to use `erb`. For Backbone, we use `jst.eco`. Everything works fine unless you need Ruby helpers in javascript templates. Here I'll show how we can combine the power of `erb` and `jst.eco`.


asset_url/asset_path
