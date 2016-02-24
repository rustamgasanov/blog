---
layout: post
title: "Dockerize Your Project. Part 2: Building Rails and Postgres images with docker-compose."
date: 2016-02-24 20:46:41 +0100
comments: true
categories: 
  - ubuntu
  - tutorial
  - example
  - rails
  - ruby
  - docker
  - jenkins
  - registry
  - ci
---

<a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">Part 1: Docker Registry and Jenkins setup with docker-compose.</a>

In this tutorial I'll show how to create a `docker` image with your Rails application, push it to the private `Docker Registry`(that we setup in Part 1), and then how to run it with `postgres` database image, wrapping everything in `docker-compose.yml`.

<!-- more -->

### Base image creation

Usually nowadays you would have more than one Ruby-application(micro-service) in your apps stack, so I'll begin with creation of the base image for them. Let's call it `myproject-baseimage`. I picked <a href="https://hub.docker.com/r/phusion/baseimage/" target="_blank">phusion/baseimage:0.9.18</a> as a base and of course we want to have `ruby 2.3` and `bundler` out of the box.

```
$ mkdir myproject-baseimage
$ cd myproject-baseimage
```

Start with 
