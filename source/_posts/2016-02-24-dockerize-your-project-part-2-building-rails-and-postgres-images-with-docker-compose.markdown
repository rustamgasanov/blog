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

Usually nowadays you would have more than one Ruby-application in your stack, so in order to have a base image for them, let's begin with creation of `baseimage` for 
