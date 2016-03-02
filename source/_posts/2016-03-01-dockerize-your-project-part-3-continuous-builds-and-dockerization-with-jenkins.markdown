---
layout: post
title: "Dockerize Your Project. Part 3: Continuous builds and dockerization with Jenkins."
date: 2016-03-01 22:00:19 +0100
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

In <a href="http://rustamagasanov.com/blog/2016/02/23/dockerize-your-project-part-1-registry-and-jenkins-setup/" target="_blank">the first part</a> of this series we've set up `jenkins` and private `docker registry`. Then, in <a href="http://rustamagasanov.com/blog/2016/02/23/http://rustamagasanov.com/blog/2016/02/24/dockerize-your-project-part-2-building-rails-and-postgres-images-with-docker-compose/" target="_blank">part 2</a> we've created a build configuration and dockerized the `Rails` application manually. Of course we don't want to do this by hand everytime the code in the main branch of the project changes, so the time has come to configure `jenkins` for automated builds and dockerization.

<!-- more -->

